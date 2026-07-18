# Data Model & Persistence

## Entity map (system of record: libSQL via Drizzle)

```
workspace ─┬─ monitor ─┬─ monitor_status (per region)
           │           ├─ incident
           │           ├─ monitor_tags ⇄ (join)
           │           └─ notifications ⇄ (join) ─ notification ─ notification_trigger
           ├─ page ─┬─ page_component (→ monitor?) ─ groups
           │        ├─ status_report ─ status_report_update
           │        ├─ maintenance ⇄ page_component
           │        └─ page_subscriber ⇄ page_component
           ├─ api_key
           ├─ audit_log
           └─ users ⇄ (users_to_workspaces)
Tinybird: ping_response__v8 (probe results — append-only, workspaceId/monitorId as strings)
```

Schema root: [packages/db/src/schema/](../../../packages/db/src/schema) — one directory per entity with `table.ts` + `validation.ts` (zod) + `index.ts`. That validation-beside-table convention is why DB reads can be `parse`d back to rich types.

## Reading the monitor table like a senior

[monitor.ts:13-73](../../../packages/db/src/schema/monitors/monitor.ts#L13-L73):
- **Three lifecycle axes**: `active` (should probe), `status` (probe verdict), `deletedAt` (soft delete). Independent booleans/columns → 8 combined states; know which are legal.
- **Partial index** `WHERE deleted_at IS NULL` on workspaceId (L68–72) — the index matches the dominant query (`listMonitors` in workspace, live only). Transferable: index the *filtered* working set, not the raw column.
- **Serialized columns**: `regions` comma-joined text (L26), `assertions`/`headers` JSON text (L34, L45). Cheap to ship, invisible to the DB: no FK to a regions table, no JSON validation. The zod layer compensates on read.
- **Nullable `workspaceId`** (L37) — the tenancy column *allows null*. Every query compensates with explicit `eq(workspaceId, …)`; a NOT NULL + FK would make the invariant structural. (Candidate improvement; check data before migrating.)
- Timestamps as unix-seconds integers with SQLite `strftime` defaults (L59–64).

## Transactions: where consistency is (and isn't) enforced

- **User-facing mutations**: always inside `withTransaction` — quota check + insert + audit atomically ([create.ts:24-62](../../../packages/services/src/monitor/create.ts#L24-L62)). The audit is fail-closed by design ([emit.ts:92-101](../../../packages/services/src/audit/emit.ts#L92-L101)).
- **Pipeline mutations**: *no transactions*. `/updateStatus` performs upsert → read → count → conditional update → incident insert as separate statements ([checker/index.ts:122-316](../../../apps/workflows/src/checker/index.ts#L122-L316)). Consistency there is best-effort + idempotency backstops.
- **Isolation caveat** (*investigate*): the client is `drizzle-orm/libsql/http` ([db.ts:6-12](../../../packages/db/src/db.ts#L6-L12)); transaction semantics over HTTP (and whether concurrent txs serialize the count-then-insert quota check) should be verified against libsql docs before you rely on them in an argument.
- **Test transactions**: rollback-only wrapper, no savepoints available ([helpers.ts:29-50](../../../packages/services/test/helpers.ts#L29-L50)).

## The second data plane

Probe results never touch SQL: the Go checker writes rows to Tinybird (`ping_response__v8`, [checker.go:50, 255](../../../apps/checker/handlers/checker.go#L50)); dashboards/status pages read typed pipes ([client.ts:74-97](../../../packages/tinybird/src/client.ts#L74-L97)). Split rationale + interview framing in [08/04 Step 3](../08-interview-prep/04-system-design-from-this-repo.md). Consequence to internalize: **cross-plane consistency is eventual and unenforced** — an incident row can exist while TB shows healthy pings (clock skew, ingestion lag). UI code must tolerate that.

## How to safely change the schema here

1. Write the schema change in `packages/db/src/schema/...`; run `pnpm generate` (drizzle-kit) to produce the SQL migration; review the generated SQL — SQLite has limited `ALTER TABLE` (many changes become table-rebuilds).
2. Update the sibling `validation.ts` zod schema **in the same PR** — reads parse; a drifted zod schema turns a working migration into runtime 500s.
3. Deploy order matters: **three apps + workflows share this DB**. Additive-nullable first; backfill; then tighten. Never rename in one step — add, dual-write, migrate readers, drop.
4. CI applies migrations before tests ([test.yml:51-57](../../../.github/workflows/test.yml#L51-L57)); seed changes ride in [seed.mts](../../../packages/db/src/seed.mts).
5. Rollback plan: migrations here are forward-only files — "rollback" = new forward migration. Say that phrase in interviews; it's how most real systems work.

## Drills

1. The 8 states of (active × status-error × deleted): mark each legal/illegal/suspicious. What does the scheduler do for `active=true, deletedAt≠null` ([cron/checker.ts:98-108](../../../apps/workflows/src/cron/checker.ts#L98-L108) — look carefully: does it filter deletedAt?). That check *is* the drill — go read.
2. Write the migration plan (5 steps) to make `monitor.workspace_id` NOT NULL. Include the query that finds violating rows first.
3. Sketch the `audit_log` table from what `emitAudit` inserts ([emit.ts:134-148](../../../packages/services/src/audit/emit.ts#L134-L148)), then diff against [audit_log.ts](../../../packages/db/src/schema/audit_logs/audit_log.ts).

## Interview angle

- Schema design & indexes → [08/03 Q2, Q4](../08-interview-prep/03-api-and-data-modeling-questions.md).
- "How do you evolve a schema without downtime?" → the additive/dual-write/tighten litany with this repo's multi-app constraint as your concrete example.
- OLTP vs analytics store split → 08/04 Step 3.
