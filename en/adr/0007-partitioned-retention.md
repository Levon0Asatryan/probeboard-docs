# ADR-0007: Daily partitions with DROP for retention

- **Status:** accepted
- **Date:** 2026-09-12
- **Relates to:** NFR-8

## Context

Raw probe results accumulate at roughly 21.6M rows per month at the target scale
of 500 endpoints on a 60-second interval. They must be retained for a configured
window and then removed, without the removal itself becoming a load problem.

## Options considered

### Option A — `DELETE FROM probe_results WHERE started_at < ...`
What Uptime Kuma does — and it issues two such statements **on every heartbeat**,
verified in `server/uptime-calculator.js`.

### Option B — declarative range partitioning by day, retention by `DROP TABLE`

## Decision

Option B.

## Rationale

`DELETE` at this volume writes as much WAL as the original inserts, leaves dead
tuples for autovacuum to reclaim, and competes with the probe write path for I/O.
Running it per probe, as Uptime Kuma does, means a full retention sweep for every
observation recorded.

Dropping a partition is a catalogue operation: constant time, no row scanning, no
dead tuples, no vacuum pressure. Partition pruning additionally narrows every
time-ranged query to the relevant days.

`DROP TABLE` on a partition is not the whole story, and M5 measured the
difference. A plain `DROP` needs `ACCESS EXCLUSIVE` on the **parent**, so it
waits behind any reader of any partition and blocks every insert while it
waits: with one reader holding the partition being retired, a retention pass
let **2 inserts through in 3 seconds**. `ALTER TABLE ... DETACH PARTITION
CONCURRENTLY` takes only `SHARE UPDATE EXCLUSIVE` on the parent; the same
scenario let **433 inserts through, p99 7.8 ms**. So retention detaches
concurrently first, then drops the detached table, which by then no query can
reach. It cannot run inside a transaction block, which is why the pass is
sequenced explicitly rather than wrapped in one.

Against the `DELETE` it replaces, on 720,000 rows (155.9 MiB): detach and drop
took **81 ms and wrote 41 KB of WAL**; `DELETE` took 152 ms, wrote **40.4 MB of
WAL — 980× more**, left 720,000 dead tuples, and freed nothing until a 1,057 ms
`VACUUM`.

## Consequences

- Easy: O(1) retention, faster range queries, no vacuum debt.
- Retention must not drop what has not been aggregated yet. The guard is a
  query, and it fails closed: a partition whose rows are not yet folded is
  left in place and reported, never dropped on the assumption that the rollup
  will have caught up.
- Harder: partitions must be created ahead of time by a maintenance job; a
  missing future partition makes inserts fail. This must be monitored — with a
  visible alert, not a silent fallback to a default partition, which would
  quietly reintroduce the unbounded table.
- The partition key is fixed at `started_at`, so retention is time-based only;
  per-endpoint retention policies are foreclosed.
- Revisit if: per-user retention tiers become a product requirement.
