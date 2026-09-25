# 7. Backend architecture

Implements §6 under the constraints of §2. Every non-obvious choice is tied to
the requirement that forces it.

## 7.1 Components

```
┌────────────┐   HTTPS/JSON   ┌──────────────┐
│ probeboard │◀──────────────▶│ probeboard   │
│    web     │                │     api      │──┐
│ React/Vite │                │   NestJS     │  │
└────────────┘                └──────────────┘  │
                                                │  same codebase,
┌──────────────────────────────────────────┐    │  separate entrypoint
│              PostgreSQL                  │◀───┤
│  config · probe results · aggregates     │    │
│  leases · incidents · outbox             │    │
└──────────────────────────────────────────┘    │
        ▲                                       │
        │                      ┌────────────────┴──────┐
        └──────────────────────│ probeboard worker ×N  │
                               │  scheduler + prober   │──▶ the internet
                               │  + rollup + notifier  │
                               └───────────────────────┘
```

**Postgres is the only infrastructure dependency.** No Redis, no message broker.
Leases, the rollup state and the notification outbox all live in the same
database, which means one transaction boundary, one backup, and `docker compose
up` satisfies NFR-16 with three containers.

Redis would buy faster counters and pub/sub. It would also add a second source
of truth for scheduling state, and the loss of transactional coupling between
"probe recorded" and "aggregate updated" — the exact coupling NFR-3 depends on.
Recorded as an ADR with that reasoning.

**api** serves the REST API and owns all user-facing reads and writes. It never
probes.

**worker** is the same NestJS codebase started with a different entrypoint and
module set. Run `N` of them; they coordinate only through Postgres. Four loops:

| Loop | Period | Job |
|---|---|---|
| scheduler | 1 s | claim due endpoints, execute probes, record results |
| rollup | 10 s | fold new results into aggregates |
| evaluator | 10 s | open/close incidents, detect `UNKNOWN`, enqueue notifications |
| dispatcher | 5 s | drain the notification outbox |

Separating them means a stuck SMTP server cannot delay probing — a variant of
NFR-1 applied to the system's own internals.

## 7.2 The path of one probe

```
1. scheduler   claim endpoints WHERE next_run_at <= now()   [SKIP LOCKED]
2. scheduler   next_run_at = scheduled_at + interval        (before probing)
3. prober      resolve DNS → validate every IP → pin → request
4. prober      capture phase boundaries, status, body (bounded)
5. prober      evaluate assertions → success/failure + class
6. worker      INSERT probe_result
7. rollup      upsert 1m / 1h / 1d aggregate buckets        [atomic SQL]
8. evaluator   consecutive counters → state → incident open/close
9. evaluator   INSERT notification_outbox
10. dispatcher send email, mark delivered or retry
```

Step 2 before step 3 is deliberate: the next slot is computed from the
*scheduled* time, not from completion, so a slow probe delays one cycle instead
of shifting every future one (NFR-2).

## 7.3 Scheduling

### The claim

```sql
WITH due AS (
  SELECT endpoint_id
  FROM   endpoint_runtime
  WHERE  enabled
    AND  next_run_at <= now()
    AND  (leased_until IS NULL OR leased_until < now())
  ORDER  BY next_run_at
  FOR UPDATE SKIP LOCKED
  LIMIT  $batch
)
UPDATE endpoint_runtime r
SET    leased_until = now() + $lease,
       leased_by    = $worker_id,
       scheduled_at = r.next_run_at,
       next_run_at  = r.next_run_at + make_interval(secs => r.interval_s)
FROM   due
WHERE  r.endpoint_id = due.endpoint_id
RETURNING r.*;
```

| Clause | Requirement |
|---|---|
| `FOR UPDATE SKIP LOCKED` | NFR-3 — workers get disjoint batches, nobody blocks |
| `leased_until < now()` | NFR-4 — a dead worker's claim is reclaimable after the lease expires |
| `next_run_at + interval` | NFR-2 — drift cannot accumulate; it is computed from the schedule, not the clock |
| no per-worker configuration | NFR-7 — scale by starting another process |

**Catch-up guard.** If a worker was down for an hour, `next_run_at + interval`
is still in the past and the endpoint would be probed 60 times in a burst. When
`next_run_at` falls more than one interval behind, it is snapped forward to the
next future slot and the skipped span is recorded as `UNKNOWN` (§3.5.2) rather
than back-filled with fiction.

`endpoint_runtime` is a **separate narrow table** from `endpoints`. Scheduling
columns are written on every probe; configuration columns almost never are.
Splitting them keeps the hot row small, keeps HOT updates on one index, and
stops autovacuum churn on the config table.

### Concurrency

Each worker runs probes on a bounded concurrency pool (default 50). A hung
endpoint occupies one slot bounded by its own timeout, never the loop (NFR-1).
The lease must exceed `timeout + slack`, or a slow probe's own lease expires
mid-flight and a second worker duplicates it.

## 7.4 The probe executor

A pure function, per the blackbox_exporter lesson (§4.5):

```ts
probe(config: EndpointProbeConfig, deps: { resolver; clock; dispatcherFactory })
  : Promise<ProbeOutcome>
```

No database, no scheduler, no global state. Fully testable against a local HTTP
server that can be made to hang, reset, return bad TLS, or answer slowly.

**Timing.** Absolute phase boundaries via undici `diagnostics_channel` and
socket events (§3.3.1), never durations. `total_ms` measured directly, never
summed.

**Body handling.** Read at most `MAX_BODY_BYTES` (default 64 KB), used for
assertions, never persisted in full (NFR-13). Only assertion outcomes and a
truncated excerpt on failure are stored.

**Failure classification.** Node error codes map to the §3.4 taxonomy in one
table. Unmapped codes become `UNKNOWN_ERROR` with the raw code retained — never
silently coerced to a generic failure.

### SSRF guard (NFR-11)

Not URL validation. Four steps, all required:

1. Reject non-`http(s)` schemes, credentials in the URL, and non-standard ports
   outside the allowed set.
2. Resolve the hostname ourselves, collecting **every** A/AAAA answer.
3. Check each against a `node:net` `BlockList` — loopback, private, link-local,
   CGNAT, multicast, reserved, `169.254.169.254`, and IPv4-mapped IPv6 forms
   (`::ffff:127.0.0.1`). Any hit fails the whole probe.
4. **Pin the connection** to a validated IP with a custom undici dispatcher, so
   the socket cannot reach an address that was never checked. This closes the
   DNS-rebinding window between step 2 and connect (§4.8).

Re-run steps 2–4 on every redirect hop. A blocked probe records
`BLOCKED_BY_POLICY`, which is `UNKNOWN`, not `DOWN` — the guard must never
manufacture an outage.

## 7.5 Data model

```
users ──▶ services ──▶ endpoints ──▶ endpoint_runtime   (1:1, hot)
                           │
                           ├──▶ probe_results     (partitioned by day)
                           ├──▶ probe_stats       (1m / 1h / 1d buckets)
                           └──▶ incidents ──▶ notification_outbox

tags ──▶ service_tags, endpoint_tags
maintenance_windows ──▶ service | endpoint
slos ──▶ service | endpoint
```

### probe_results — raw, partitioned

```sql
CREATE TABLE probe_results (
    endpoint_id   uuid        NOT NULL,
    started_at    timestamptz NOT NULL,
    outcome       probe_outcome NOT NULL,      -- up | down | degraded | unknown
    failure_class failure_class,               -- §3.4, null on success
    status_code   smallint,
    total_ms      integer,
    dns_ms        integer,
    connect_ms    integer,
    tls_ms        integer,
    ttfb_ms       integer,
    transfer_ms   integer,
    assertions    jsonb,                       -- per-assertion results (§4.4)
    error_message text,
    cert_expires_at timestamptz,
    worker_id     text,
    PRIMARY KEY (endpoint_id, started_at)
) PARTITION BY RANGE (started_at);
```

Daily partitions. Retention is `DROP PARTITION` — O(1), no vacuum, no bloat.
This is the direct fix for the third Uptime Kuma defect in §4.2, where retention
runs two `DELETE` statements on the hot path of every probe.

### probe_stats — the aggregates that serve every chart

```sql
CREATE TABLE probe_stats (
    endpoint_id  uuid        NOT NULL,
    granularity  stat_grain  NOT NULL,          -- m1 | h1 | d1
    bucket_start timestamptz NOT NULL,
    count_up          integer NOT NULL DEFAULT 0,
    count_down        integer NOT NULL DEFAULT 0,
    count_degraded    integer NOT NULL DEFAULT 0,
    count_unknown     integer NOT NULL DEFAULT 0,
    count_maintenance integer NOT NULL DEFAULT 0,
    -- Seconds observed: up + down + degraded. An unknown probe adds none.
    covered_seconds   integer NOT NULL DEFAULT 0,   -- time-weighted uptime (§3.5.1)
    up_seconds        integer NOT NULL DEFAULT 0,
    degraded_seconds  integer NOT NULL DEFAULT 0,
    sum_total_ms      bigint  NOT NULL DEFAULT 0,
    min_total_ms      integer,
    max_total_ms      integer,
    sum_ttfb_ms       bigint  NOT NULL DEFAULT 0,
    hist_total        integer[] NOT NULL DEFAULT array_fill(0, ARRAY[20]),
    PRIMARY KEY (endpoint_id, granularity, bucket_start)
);
```

Histogram edges, fixed and documented (ms):
`10, 25, 50, 75, 100, 150, 200, 300, 500, 750, 1000, 1500, 2000, 3000, 5000,
7500, 10000, 15000, 30000, ∞`

p95 is linear interpolation inside the containing bucket. Buckets merge by
integer addition, so minute → hour → day rollup is associative and exact
(§3.6). The same array renders as the latency heatmap.

Both count- and time-weighted uptime are derivable, because the `count_*`
columns and the seconds columns are both stored — §3.5.1's comparison becomes a
query, not a redesign. `covered_seconds` alone is not enough for the
time-weighted form: it is the denominator, and the numerator needs
`up_seconds` (and `degraded_seconds`, once M6 writes `degraded`) held
separately, because the counts cannot supply it when an endpoint's interval
changed inside the window.

Seconds are attributed whole to the bucket holding `started_at`, so a window
shorter than the longest interval in it does not get a meaningful time-weighted
answer. M5 states that limit rather than interpolating across bucket edges.

### The rollup must be atomic

The second Uptime Kuma defect (§4.2) is read-modify-write in application code.
probeboard increments inside the database:

The fold aggregates a batch of raw rows into one row per bucket in SQL, and adds
that row to whatever is already stored:

```sql
INSERT INTO probe_stats AS s (endpoint_id, granularity, bucket_start,
                              count_up, covered_seconds, up_seconds,
                              sum_total_ms, min_total_ms, max_total_ms,
                              hist_total)
SELECT endpoint_id, 'm1', date_trunc('minute', started_at),
       count(*) FILTER (WHERE outcome = 'up'),
       coalesce(sum(interval_s) FILTER
                (WHERE outcome IN ('up','down','degraded')), 0)::int,
       coalesce(sum(interval_s) FILTER (WHERE outcome = 'up'), 0)::int,
       coalesce(sum(total_ms), 0), min(total_ms), max(total_ms),
       <the batch's own 20-element histogram>
FROM   new_rows
GROUP  BY endpoint_id, date_trunc('minute', started_at)
ON CONFLICT (endpoint_id, granularity, bucket_start) DO UPDATE SET
    count_up        = s.count_up        + EXCLUDED.count_up,
    covered_seconds = s.covered_seconds + EXCLUDED.covered_seconds,
    up_seconds      = s.up_seconds      + EXCLUDED.up_seconds,
    sum_total_ms    = s.sum_total_ms    + EXCLUDED.sum_total_ms,
    min_total_ms    = least(s.min_total_ms, EXCLUDED.min_total_ms),
    max_total_ms    = greatest(s.max_total_ms, EXCLUDED.max_total_ms),
    hist_total      = ARRAY(SELECT a + b
                            FROM unnest(s.hist_total, EXCLUDED.hist_total)
                                 WITH ORDINALITY AS h(a, b, ord)
                            ORDER BY ord);
```

No row is ever read into a worker and written back, so concurrent workers cannot
lose an update (review rule g15).

**Every column must be folded the same way on both paths.** An earlier version
of this snippet inserted `array_fill(0, ARRAY[20])` and incremented
`hist_total[$k]` only in the `DO UPDATE` branch, which silently dropped the
first probe of every bucket from the histogram — the insert path wrote a zero
vector and nothing ever added that probe back. The rule the shape enforces:
whatever the `VALUES`/`SELECT` side contributes, `DO UPDATE` adds the same
thing to what is stored, so the two paths differ only in what they start from.
Merging the whole array elementwise, rather than touching one index, makes that
symmetry hard to break.

### Idempotency

The rollup is driven off a single watermark over **transaction ids**, not over
time, and the fold and the watermark advance commit together — so the step is
exactly-once in effect (review rule g13).

A timestamp watermark loses data, and M5 demonstrated it rather than reasoning
about it. `started_at` is set before the probe runs and the row commits some
time later, so two writers can commit out of order: a pass that folds
everything up to time *T* and stores *T* will never look below *T* again, and a
row with `started_at < T` that commits afterwards is skipped for ever. The
window is exactly the probe's own duration, so the rows most likely to be lost
are the slow ones — the outages.

Instead every `probe_results` row carries `insert_xid xid8 DEFAULT
pg_current_xact_id()`, and a pass folds `insert_xid` in `[last_xid, horizon)`
where the horizon is `pg_snapshot_xmin(pg_current_snapshot())`, read once per
pass. Below that horizon no transaction is still in flight, so no row can
appear later beneath a watermark that has already passed it. `xid8` rather than
`xid` because it does not wrap.

The natural key stays `(endpoint_id, started_at, attempt_id)`: a retry of one
attempt's write is idempotent under `ON CONFLICT DO NOTHING`, while a second
attempt in the same millisecond is legitimately its own row.

### endpoint_runtime

```sql
CREATE TABLE endpoint_runtime (
    endpoint_id           uuid PRIMARY KEY,
    enabled               boolean     NOT NULL,
    interval_s            integer     NOT NULL,
    next_run_at           timestamptz NOT NULL,
    scheduled_at          timestamptz,
    leased_until          timestamptz,
    leased_by             text,
    state                 endpoint_state NOT NULL,     -- §3.7
    consecutive_failures  smallint NOT NULL DEFAULT 0,
    consecutive_successes smallint NOT NULL DEFAULT 0,
    last_probe_at         timestamptz,
    last_success_at       timestamptz
);
CREATE INDEX ON endpoint_runtime (next_run_at) WHERE enabled;
```

The partial index is what makes the claim query cheap at 500+ endpoints.

## 7.6 Incident evaluation

Runs in the evaluator loop, not in the prober, so notification logic cannot
slow down probing.

```
on probe result:
  failure → consecutive_failures++, consecutive_successes = 0
  success → consecutive_successes++, consecutive_failures = 0

  state UP|DEGRADED and failures == 1            → PENDING
  state PENDING and failures >= threshold_open   → DOWN, open incident
  state DOWN and successes >= threshold_close    → UP, close incident
  success and total_ms > latency_warn_ms         → DEGRADED
```

`opened_at` is the timestamp of the **first** failed probe of the run, not the
one that crossed the threshold (§3.8); `confirmed_at` records the crossing, so
detection latency is measurable. Symmetrically for `closed_at`.

**`UNKNOWN` detection** is a separate sweep: any enabled endpoint whose
`last_probe_at` is older than `2 × interval + grace` is marked `UNKNOWN`. This
is the only mechanism that notices probeboard itself failing.

**Maintenance** is checked before any state transition. Inside a window, results
are recorded and counted as `count_maintenance`, no incident opens, no email is
sent.

## 7.7 Notifications

A transactional **outbox**, not a direct send:

```sql
CREATE TABLE notification_outbox (
    id            uuid PRIMARY KEY,
    user_id       uuid NOT NULL,
    service_id    uuid NOT NULL,
    kind          notification_kind NOT NULL,   -- incident_open | incident_close | cert_expiry | slo_burn
    group_key     text NOT NULL,                -- service + kind + window
    payload       jsonb NOT NULL,
    not_before    timestamptz NOT NULL,         -- grouping delay
    attempts      smallint NOT NULL DEFAULT 0,
    delivered_at  timestamptz,
    last_error    text,
    UNIQUE (group_key)
);
```

The row is written **in the same transaction that opens the incident**. Either
both commit or neither does — an incident can never exist without its
notification, and a notification can never exist for an incident that was rolled
back (review rule g14).

| PRD rule | Mechanism |
|---|---|
| E-3 one email per service | `UNIQUE (group_key)`; `not_before = now() + 60s` collects siblings before the dispatcher picks it up |
| E-4 no spam | `incidents.last_notified_at` + decaying schedule 30 m / 2 h / 6 h / daily |
| E-8 never silently dropped | `attempts` with exponential backoff; permanent failure sets `last_error` and surfaces in the UI — the failure is never swallowed (review rule g2) |

Email is rendered server-side; SMTP configuration is environment-only, never in
source (review rule g10).

## 7.8 API surface

```
POST   /auth/register            POST /auth/login          POST /auth/logout
POST   /auth/verify-email        POST /auth/password

GET    /services                 POST /services
GET    /services/:id             PATCH /services/:id       DELETE /services/:id
GET    /services/:id/health

POST   /services/:id/endpoints
GET    /endpoints/:id            PATCH /endpoints/:id      DELETE /endpoints/:id
POST   /endpoints/:id/pause      POST /endpoints/:id/resume
POST   /endpoints/:id/check-now

GET    /endpoints/:id/results?from&to&limit
GET    /endpoints/:id/stats?from&to&granularity
GET    /endpoints/:id/uptime?window
GET    /endpoints/:id/incidents
GET    /incidents?status

GET    /tags                     POST /maintenance-windows
GET    /slos                     POST /slos
GET    /healthz                  GET  /metrics
```

**Contract rules.** Every list endpoint is paginated with a cursor and a bounded
`limit`. `/stats` picks granularity from the requested span and refuses spans
that would require raw rows (NFR-9). Every input is validated at the boundary
with a schema, including URL and header shape (review rule g12). Errors return a
stable machine-readable `code`, never a stack trace.

## 7.9 Module layout — `probeboard-api`

```
src/
  main.ts                 api entrypoint
  worker.ts               worker entrypoint
  common/                 config, logging, errors, pagination, crypto
  auth/                   register, login, sessions, password hashing
  services/               service CRUD, header storage
  endpoints/              endpoint CRUD, validation, quotas
  probing/
    probe.executor.ts     pure function (§7.4)
    ssrf.guard.ts         resolve → classify → pin
    timing.ts             phase boundaries
    assertions/           evaluation, versioned schema
    failure-classes.ts
  scheduler/              claim loop, lease, catch-up
  rollup/                 watermark, atomic upserts, retention
  incidents/              state machine, UNKNOWN sweep, maintenance
  notifications/          outbox, grouping, email/webhook channels
  stats/                  uptime, percentiles from histograms, SLO math
  metrics/                Prometheus exposition
```

`probing/` depends on nothing else in the tree — that is what keeps it a pure
function and what makes the security tests easy to write.

## 7.10 Testing

| Level | Covers | Notes |
|---|---|---|
| unit | assertions, failure mapping, percentile interpolation, histogram merge, state machine, burn rate | pure functions, no I/O |
| integration | probe executor against a local server that hangs, resets, serves bad TLS, redirects to `127.0.0.1` | real sockets, no internet |
| integration | claim query with N concurrent workers | real Postgres, no mocks |
| system | §2.3 criteria: load, fault injection, SSRF, retention | these produce the evaluation chapter |

No focused or skipped tests reach a commit (review rule g19); every bug fix
lands with the test that fails without it (g18).

## 7.11 Decisions to record as ADRs

| # | Decision | Alternatives rejected |
|---|---|---|
| 1 | Postgres only, no Redis | Redis for leases/queue — second source of truth, loses transactional coupling |
| 2 | Hand-rolled `SKIP LOCKED` leases | pg-boss, BullMQ, `@nestjs/schedule` — the last is broken by construction for multi-instance |
| 3 | Fixed histogram buckets | t-digest/DDSketch, raw retention |
| 4 | Absolute phase boundaries | per-phase durations — breaks on redirects |
| 5 | Structured versioned assertions | Gatus-style string DSL — kept as optional sugar |
| 6 | Worker in the same codebase, separate entrypoint | separate repo, or probing inside the API process |
| 7 | Daily partitions + `DROP` | `DELETE`-based retention |
| 8 | Transactional outbox for email | direct send from the evaluator |
