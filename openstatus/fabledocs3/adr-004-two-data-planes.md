# ADR-004: Two data planes — libSQL for state, Tinybird for telemetry

**Status:** Accepted, stable. The unmeasured loss rate on the telemetry plane is the open wound.

## Context

Two workloads with opposite shapes share one product. Transactional state (workspaces, monitors, pages, incidents) is low-volume, read-mostly, and needs ACID + tenancy. Probe telemetry is an append-only firehose — a 30-second monitor across 28 regions produces ~80k rows/day *per monitor* — and is queried analytically (uptime percentages, latency percentiles, time-windowed charts). One store serving both means the firehose eventually threatens the plane users are clicking on.

## Decision

Split by workload. OLTP state lives in libSQL via Drizzle (`packages/db`). Probe results **never touch SQL**: the Go checker appends each result directly to Tinybird (`ping_response__v8`), and all reads go through typed pipe clients in `packages/tinybird`. Pipes are versioned by name suffix (`__v8`) because analytics schemas evolve destructively. Cross-plane consistency is explicitly *not* enforced — it is eventual, and UI code must tolerate disagreement (an incident row in SQL while Tinybird still shows healthy pings).

## Evidence

- `apps/checker/handlers/checker.go:50, 255` — the checker writes `ping_response__v8` directly; note L255-257: a failed Tinybird send is **logged and dropped**, no buffer, no retry queue.
- `packages/tinybird/src/client.ts:15-97` — typed pipe clients; the package owns nothing but this.
- `packages/db/src/schema/` — the OLTP entity map (workspace → monitor/page/incident/notification/audit_log); telemetry ids cross the boundary as *strings* in Tinybird rows, not foreign keys.

## Alternatives considered

1. **Everything in Postgres** (partitioned ping table + rollups). Rejected: viable at small scale but couples the failure domains and makes the write-heavy plane a permanent operational tax on the transactional one; also forfeits Tinybird's materialized-pipe query latency for status-page charts.
2. **Self-hosted ClickHouse.** Rejected (inferred): right engine, wrong operational cost for the team size — same reasoning as Cloud Tasks in ADR-003. Tinybird *is* managed ClickHouse with a pipe/contract layer.
3. **Buffer telemetry through the queue or a local spool** before Tinybird. Not adopted: the checker fires and forgets. This is the decision's sharpest edge — see negative consequences.

## Consequences

**Positive:** the firehose can never take down CRUD; each plane gets a right-shaped engine and query model; pipe versioning gives the analytics schema an explicit evolution mechanism the SQL side lacks; the `packages/tinybird` contract layer means product code never writes raw ClickHouse SQL.

**Negative:** *the uptime data — the product itself — has the weakest durability in the system*: a Tinybird outage or network blip silently loses rows, and there is no loss-rate metric, so the effective data SLA is unknown (risk R3 in `../fabledocs/upskill/09-reference/risk-register.md`). Cross-plane joins are impossible; every screen that combines state and telemetry does two reads and reconciles in code. Eventual consistency leaks into UX copy ("operational" next to an open incident is *correct* here, and someone has to write that sentence carefully).
