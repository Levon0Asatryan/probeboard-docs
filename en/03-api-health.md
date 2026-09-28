# 3. What "API health" means

This chapter defines the thing probeboard measures. It is deliberately detailed:
"health" is the central concept of the thesis, and most of the design decisions
that follow are consequences of taking it apart properly.

## 3.1 Vocabulary

| Term | Definition | Example |
|---|---|---|
| **SLI** — service level indicator | A measured quantity | fraction of probes that succeeded |
| **SLO** — service level objective | A target for an SLI over a window | ≥ 99.5% of probes succeed over 30 days |
| **SLA** — service level agreement | An SLO with a business consequence | credit refunded if the SLO is missed |
| **Error budget** | The failure allowed by the SLO | 0.5% of 30 days = 3h 36m |
| **Burn rate** | How fast the budget is being consumed relative to uniform | 14.4× = a 30-day budget gone in 50 minutes |

probeboard produces SLIs, lets the user declare SLOs, and computes error budgets
from them. SLAs are out of scope — they are a contractual artefact, not a
measurement.

## 3.2 Health is a conjunction, not a boolean

An HTTP request traverses several independent subsystems. Each can fail or
degrade on its own, and each failure means something different operationally:

```
 probeboard        DNS        TCP         TLS        server        body
     │              │          │           │           │            │
     ├─ resolve ────┤          │           │           │            │
     ├─ connect ───────────────┤           │           │            │
     ├─ handshake ─────────────────────────┤           │            │
     ├─ send request ──────────────────────────────────┤            │
     ├─ wait (server works) ───────────────────────────┤            │
     └─ read body ──────────────────────────────────────────────────┤
```

So probeboard measures six dimensions, not one:

| # | Dimension | Question | Failure looks like |
|---|---|---|---|
| H1 | **Reachability** | Does the name resolve, the port accept, the certificate validate? | NXDOMAIN, connection refused, expired certificate |
| H2 | **Availability** | Does it answer in time, with an acceptable status? | 503, timeout |
| H3 | **Correctness** | Is the answer the *right* answer? | `200 OK` + `{"error": "..."}` |
| H4 | **Latency** | How long, and where is the time spent? | p95 doubled after a deploy |
| H5 | **Stability** | Consistent, or flapping? | up/down/up/down within one hour |
| H6 | **Hygiene** | Will it break soon for a foreseeable reason? | certificate expires in 6 days |

Two consequences are load-bearing for the whole design.

**Correctness is not availability (H3 ≠ H2).** An endpoint returning `200 OK`
with `{"error": "database unavailable"}` is *available* and *unhealthy*. A
monitor that reads only the status line reports it green. This is the precise
difference between monitoring a *web page* and monitoring an *API*, and it is
why assertions are core rather than optional.

**Latency is a distribution, not a number (H4).** An endpoint answering 95% of
requests in 40 ms and 5% in 30 s has a mean near 1.5 s — a value it never
actually exhibits. The mean is the least informative available statistic.
Percentiles are the correct summary, and percentiles are **not averageable**,
which constrains the storage design (§3.6).

## 3.3 The request lifecycle, as measured

### 3.3.1 Phase boundaries

Every phase boundary is an observable event in both Go (`net/http/httptrace`)
and Node (socket events / undici `diagnostics_channel`). probeboard records the
**absolute timestamp of each boundary**, not the duration of each phase:

| Boundary | Go `httptrace` | Node |
|---|---|---|
| `dns_start` | `DNSStart` | socket `lookup` start |
| `dns_done` | `DNSDone` | socket `lookup` event |
| `connect_start` | `ConnectStart` | before `connect` |
| `connect_done` | `ConnectDone` | socket `connect` event |
| `tls_start` | `TLSHandshakeStart` | before `secureConnect` |
| `tls_done` | `TLSHandshakeDone` | socket `secureConnect` event |
| `first_byte` | `GotFirstResponseByte` | first `data` on response |
| `transfer_done` | body read complete | response `end` |

Storing boundaries rather than durations is what
[openstatus's checker](https://github.com/openstatusHQ/openstatus/blob/main/apps/checker/checker/http.go)
does (`Timing{DnsStart, DnsDone, ConnectStart, …}`), and it is the better
choice for three reasons:

1. Any phase duration is derivable; the reverse is not true.
2. Redirects and connection reuse produce overlapping or absent phases. With
   durations these become ambiguous — the `blackbox_exporter` project has an
   open bug of exactly this shape, where `phase="tls"` reports nonsense when a
   probe follows more than one redirect.
3. Gaps between phases (scheduler jitter, socket-pool wait) stay visible instead
   of being silently folded into a neighbouring phase.

### 3.3.2 Derived phases and what each one blames

| Phase | Derivation | A spike here blames |
|---|---|---|
| `dns_ms` | `dns_done − dns_start` | the resolver, not the endpoint |
| `connect_ms` | `connect_done − connect_start` | network path, packet loss, full SYN backlog |
| `tls_ms` | `tls_done − tls_start` | certificate chain, OCSP, server CPU |
| `ttfb_ms` | `first_byte − (tls_done ∥ connect_done)` | **the API's own processing** |
| `transfer_ms` | `transfer_done − first_byte` | payload size, bandwidth |
| `total_ms` | `transfer_done − dns_start` | the user's experience |

`ttfb_ms` is the only phase that measures the API itself; the rest measure the
path to it. Reporting only `total_ms` — which is what most self-hosted monitors
do — conflates the two and makes "our API got slower" indistinguishable from
"our DNS provider got slower".

A caveat worth recording in the thesis: the phases do not sum to the total.
Observed discrepancies of roughly 10% are normal, since connection setup,
scheduling and socket-pool waits fall between the instrumented boundaries.
`total_ms` must therefore be measured directly, never computed as a sum.

## 3.4 Failure taxonomy

FR-20 requires failures to be distinguishable. The classification below is the
full set, with the concrete signals that produce it in a Node probe executor.

| Class | Dimension | Node signal | Operational meaning |
|---|---|---|---|
| `DNS_NXDOMAIN` | H1 | `ENOTFOUND` (`getaddrinfo`); `ENOTFOUND`, `ENODATA` (c-ares) | name does not exist — usually config, not outage |
| `DNS_FAILURE` | H1 | `EAI_AGAIN` (`getaddrinfo`); `ESERVFAIL`, `EREFUSED`, `ETIMEOUT`, `ECONNREFUSED`, `EBADRESP`, `ENOTIMP`, `EFORMERR` (c-ares) | resolver itself is failing |
| `CONNECTION_REFUSED` | H1 | `ECONNREFUSED` | host up, nothing listening — process is down |
| `CONNECTION_TIMEOUT` | H1 | `UND_ERR_CONNECT_TIMEOUT`, `ETIMEDOUT`, `EHOSTUNREACH`, `ENETUNREACH` | packets dropped or rejected — firewall or dead host |
| `CONNECTION_RESET` | H1 | `ECONNRESET`, `EPIPE`, `UND_ERR_SOCKET` | peer killed the connection mid-flight |
| `TLS_EXPIRED` | H1/H6 | `CERT_HAS_EXPIRED` | certificate lapsed — foreseeable, therefore preventable |
| `TLS_UNTRUSTED` | H1 | `UNABLE_TO_VERIFY_LEAF_SIGNATURE`, `DEPTH_ZERO_SELF_SIGNED_CERT`, `SELF_SIGNED_CERT_IN_CHAIN`, `UNABLE_TO_GET_ISSUER_CERT`, `UNABLE_TO_GET_ISSUER_CERT_LOCALLY` | chain incomplete or self-signed |
| `TLS_HOSTNAME_MISMATCH` | H1 | `ERR_TLS_CERT_ALTNAME_INVALID` | certificate is for a different name |
| `TLS_HANDSHAKE_FAILED` | H1 | `EPROTO`, any `ERR_SSL_*` | protocol/cipher mismatch |
| `RESPONSE_TIMEOUT` | H2 | `UND_ERR_HEADERS_TIMEOUT` | connected, server never answered |
| `BODY_TIMEOUT` | H2 | `UND_ERR_BODY_TIMEOUT` | headers arrived, body stalled |
| `STATUS_MISMATCH` | H2 | status ∉ accepted set | the endpoint answered, with the wrong answer |
| `ASSERTION_FAILED` | H3 | assertion evaluation | the payload is wrong |
| `TOO_MANY_REDIRECTS` | H2 | redirect budget exhausted | redirect loop |
| `BLOCKED_BY_POLICY` | — | SSRF guard (NFR-11) | *probeboard refused*, not an endpoint failure |

**Two resolvers, two vocabularies.** An unguarded probe resolves through
`dns.lookup`, which is `getaddrinfo`; the SSRF guard resolves every A and AAAA
record through `resolve4`/`resolve6`, which is c-ares. They report the same
failures with different codes — c-ares never produces `EAI_AGAIN` — so each is
read through its own table. In particular, c-ares' `ECONNREFUSED` means *the
resolver* refused the query, not the endpoint, and never maps to
`CONNECTION_REFUSED`. A code on neither list stays `UNKNOWN_ERROR` with the
code preserved: mapping a documented code reads the signal; mapping an
unrecognised one would invent a diagnosis.

`EHOSTUNREACH` and `ENETUNREACH` are the dropped-packet case reported actively
(an ICMP unreachable or a `REJECT` rule rather than a `DROP`), so a firewall's
choice between the two must not change the class. The same address measured
`EHOSTUNREACH` on macOS and `ENETUNREACH` on Linux, so both are listed.

Two design points here matter more than the list itself:

**`BLOCKED_BY_POLICY` is not a failure of the monitored endpoint.** It must be
excluded from uptime arithmetic, or the SSRF defence would silently manufacture
outages. Conflating "we refused to probe" with "it is down" is a correctness
bug in the measurement, not a cosmetic one.

**The class determines the alert, not just the label.** `DNS_NXDOMAIN` on a
newly created monitor is a typo; `CONNECTION_REFUSED` at 03:00 is an outage.
Same red dot, entirely different response.

## 3.5 Availability arithmetic

### 3.5.1 Count-weighted vs time-weighted uptime

Two definitions are available, and the difference is not academic:

```
count-weighted:   uptime = successful probes / total probes
time-weighted:    uptime = seconds observed up / seconds in window
```

Uptime Kuma — the closest comparable system — uses count-weighted, summing `up`
and `down` counters per bucket. It is simple and it is wrong in three cases:

1. **The interval changed mid-window.** 60 s probes and 300 s probes get equal
   vote, so history is silently reweighted whenever a user edits a monitor.
2. **There are gaps.** If the monitoring system itself was down for six hours,
   those hours simply vanish from the denominator, and uptime reads 100%.
3. **Maintenance windows.** Excluded probes reduce the denominator, which is
   usually intended — but only if it is a deliberate decision, not a side
   effect.

Time-weighted uptime attributes to each probe the interval it represents, so
gaps become explicitly `UNKNOWN` time rather than disappearing.

**Recommendation:** store both numerator components per bucket — probe counts
*and* covered seconds — so the thesis can report both and quantify the
divergence. That comparison is a small, genuinely novel evaluation result:
nobody publishes how far apart the two definitions land in practice.

### 3.5.2 `UNKNOWN` is a first-class state

Most naive implementations have two states and therefore conflate "the endpoint
is healthy" with "we have no data". A crashed worker then renders as a wall of
green — the monitoring system fails silently in the one direction that matters.

probeboard treats a missing expected probe as `UNKNOWN`, excluded from both the
numerator and the denominator of uptime, and surfaced distinctly in the UI. This
is the user-visible half of NFR-4.

## 3.6 Latency statistics and the storage constraint

FR-34 requires p95. NFR-8/NFR-9 require long windows to be served from
aggregates. These two requirements collide, because **percentiles cannot be
averaged**: given p95 for each of 24 hourly buckets, there is no arithmetic that
recovers the true p95 of the day.

Three viable designs:

| Option | Per bucket | p95 over 30d | Cost | Notes |
|---|---|---|---|---|
| **A. Raw retention** | nothing | exact | unbounded | violates NFR-8 outright |
| **B. Fixed histogram buckets** | ~20 counters, e.g. 10/25/50/100/250/500/1000/2500/5000/10000 ms | interpolated, error bounded by bucket width | ~20 integers | mergeable by simple addition; also yields the H4 heatmap for free |
| **C. t-digest / DDSketch sketch** | serialized sketch, ~1–3 KB | relative-error guarantee (DDSketch: configurable, e.g. 1%) | larger, needs a library | the approach TimescaleDB's toolkit and Datadog both use |

**Recommendation: B.** Histogram buckets are mergeable by integer addition,
which means the rollup from minute → hour → day is trivial and provably
associative; they need no extension or third-party library; and the same
structure renders directly as the latency heatmap. The accuracy loss is bounded
and quantifiable — which itself becomes an evaluation result: measure exact p95
on retained raw data against bucket-interpolated p95, and report the error.

Option C belongs in the thesis as the alternative considered, with the
Timescale/Datadog precedent cited. Option A is the straw man that NFR-8 exists
to rule out.

Per bucket, probeboard therefore stores: `count_up`, `count_down`,
`count_unknown`, `covered_seconds`, `sum_ms`, `min_ms`, `max_ms`, and the
histogram — plus the same for `ttfb_ms` if per-phase percentiles are wanted.

## 3.7 The state machine

States (§3.2 dimensions mapped to what the user sees):

| State | Meaning |
|---|---|
| `UP` | last probe succeeded, latency under the warning threshold |
| `DEGRADED` | probes succeed, latency over the warning threshold |
| `DOWN` | N consecutive failures (FR-24) |
| `PENDING` | failing, but N not yet reached — not an incident |
| `MAINTENANCE` | inside a declared window; probes run, incidents suppressed |
| `PAUSED` | disabled by the user; no probes |
| `UNKNOWN` | expected probe did not arrive (§3.5.2) |

Transitions, with hysteresis so a single bad probe cannot open an incident and a
single good probe cannot close one:

```
        fail                 fail ×N            success ×M
  UP ──────────▶ PENDING ──────────▶ DOWN ──────────────▶ UP
   ▲                │                                      
   └── success ─────┘        (latency > threshold) UP ⇄ DEGRADED
```

`PENDING` exists so that the retry window is *visible* rather than hidden
inside a counter — the user can see "failing, 2 of 3" instead of a green dot
that suddenly turns red. Uptime Kuma uses the same three-value encoding
(`0=DOWN, 1=UP, 2=PENDING, 3=MAINTENANCE`), which is good evidence the shape is
right.

Only transitions produce notifications. Uptime Kuma marks these with an
`important` flag on the heartbeat row, giving a cheap "show me only the
transitions" query for incident history — a pattern worth copying directly.

## 3.8 Incidents and their derived metrics

An incident is a `DOWN` interval with a cause:

- `opened_at` — timestamp of the *first* failed probe, not the Nth. The outage
  began when it began; the confirmation delay is probeboard's latency, not the
  endpoint's downtime. Recording the Nth would systematically under-report
  downtime by `(N−1) × interval`.
- `closed_at` — timestamp of the first successful probe of the recovery run,
  symmetrically.
- `cause` — the failure class that opened it (§3.4).
- `detection_latency` = `confirmed_at − opened_at`, a metric about probeboard
  itself, worth reporting in the evaluation chapter.

From the incident series:

```
MTTR  = mean(closed_at − opened_at)                  how bad, when it breaks
MTBF  = mean(opened_at[i+1] − closed_at[i])          how often it breaks
availability ≈ MTBF / (MTBF + MTTR)
```

## 3.9 SLOs and error budgets

Given target `S` (e.g. 0.995) over window `W` (e.g. 30 days):

```
error budget          = (1 − S) × W                    = 3h 36m for 99.5%/30d
budget consumed       = downtime observed in W
budget remaining %    = 1 − consumed / budget
burn rate            = observed error rate / (1 − S)
```

A burn rate of 1 exhausts the budget exactly at the end of the window; 14.4
exhausts a 30-day budget in 50 minutes.

Alerting on burn rate rather than on a raw failure is the modern practice, and
the Google SRE workbook's multiwindow multi-burn-rate scheme is the reference
design: a short and a long window must *both* exceed the threshold before the
alert fires, which kills the noise of a single-window trigger.

| Tier | Burn rate | Long window | Short window | Action |
|---|---|---|---|---|
| 1 | 14.4× | 1 h | 5 min | page |
| 2 | 6× | 6 h | 30 min | page |
| 3 | 3× | 24 h | 2 h | ticket |

**Recommendation:** implement the error-budget computation and the tier-1 burn
alert. It costs almost nothing on top of the aggregates §3.6 already stores, and
it moves the statistics chapter from "we display a percentage" to "we evaluate a
policy" — a materially stronger thesis position.

## 3.10 What a black-box prober cannot know

Stating the limits explicitly is better defence material than pretending there
are none:

- **Why** it is slow. probeboard can say the server took 900 ms to first byte.
  It cannot say whether that was a slow query, GC, or CPU starvation. That
  requires instrumentation inside the process — white-box (see prior art).
- **Whether real users are affected.** A probe from one location is not the same
  as traffic from everywhere. Regional failures are invisible to a single-region
  prober; this is why commercial products probe from many regions, and why §1.5
  lists multi-region as future work.
- **Partial degradation.** One URL per monitor means one code path. An endpoint
  healthy for `GET /health` and broken for `POST /orders` looks fine.
- **Low-frequency faults.** A 60-second interval samples 1,440 times a day. A
  fault affecting 0.1% of requests will usually be missed entirely. probeboard
  measures *availability of the endpoint to a periodic client*, which is a
  proxy for — not identical to — the success rate real users experience.

That last point is the sharpest honest limitation, and it deserves a paragraph
in the thesis: sampling-based availability is a lower-bound estimator whose
confidence depends on probe frequency.
