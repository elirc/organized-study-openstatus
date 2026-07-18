# Part 3 — Data access

## What it is and why it exists

Two data planes, split by workload:

- **OLTP plane:** libSQL (Turso) via Drizzle. Schema lives in `packages/db/src/schema/` — one directory per entity, each with `table.ts` + a sibling zod `validation.ts`. The client is `drizzle-orm/libsql/http` (`packages/db/src/db.ts:6-12`). Holds workspaces, monitors, pages, incidents, notifications, audit logs.
- **Telemetry plane:** Tinybird. The Go checker appends every probe result to `ping_response__v8` (`apps/checker/handlers/checker.go:50, 255`); dashboards and status pages read through typed pipe clients (`packages/tinybird/src/client.ts:15-97`). Note the `__v8` suffix — pipes are versioned by name because analytics schemas evolve destructively.

Why two planes: a monitor checked every 30s from 28 regions produces ~80k rows/day *per monitor*. Put that in your transactional database and the write volume of the firehose eventually threatens the CRUD plane that users are clicking on. The split means the write-heavy plane can never take down the transactional one. The price, and you must internalize this: **cross-plane consistency is eventual and unenforced.** An incident row can exist in SQL while Tinybird still shows healthy pings (ingestion lag, clock skew). UI code tolerates that; yours must too.

## The hard decisions

### 1. SQLite/libSQL as the system of record

**Rejected:** Postgres, the default answer. I went libSQL because the OLTP shape here is read-mostly, low-write, tenant-scoped — SQLite semantics fit, Turso gives us replicas near the edge (which pairs with the Edge-runtime tRPC from Part 1), and operationally it's nearly free. **What still bothers me, in ascending order:** SQLite's crippled `ALTER TABLE` means many migrations become table rebuilds; timestamps are unix-seconds integers with `strftime` defaults (`packages/db/src/schema/monitors/monitor.ts:59-64`), a foot-gun next to JS milliseconds — we've had at least one off-by-1000 (see `getScheduledTime` mixing conventions, `apps/workflows/src/cron/monitor.ts:393-407`); and I have never fully satisfied myself about transaction isolation over the *HTTP* driver. That last one is an open question I want you to hold with appropriate suspicion — don't build anything that depends on serializable semantics without verifying against libsql docs first.

### 2. Cheap serialization over normalized purity

Look at the monitor table (`monitor.ts:13-73`): `regions` is comma-joined text, `assertions` and `headers` are JSON-in-text, and — the one I regret — `workspaceId` is *nullable* despite being the tenancy column. **Rejected:** a regions join table, JSON columns with CHECK constraints, NOT NULL + FK from day one. The serialized columns were the right call — they shipped weeks earlier and the sibling zod `validation.ts` compensates on read (this is why DB reads get `parse`d back into rich types; keep the zod file in the same PR as any schema change or a working migration becomes runtime 500s). The nullable `workspaceId` was **not** the right call: it means tenancy is enforced by every query remembering `eq(workspaceId, …)` instead of by the schema. The compensations are real — fetch-or-throw helpers in `packages/services/src/monitor/internal.ts:28-47`, even join-table ids ownership-validated (L63-84) — but structural beats disciplined, always. The NOT NULL migration is written in my head and has been for a year: find violating rows, backfill, tighten. Additive-nullable first; backfill; then constrain — that's the litany for *every* schema change here, because three apps plus workflows share this database and deploy independently.

### 3. Transactions for users, best-effort for the pipeline

User-facing mutations are strict: `withTransaction(ctx, …)` wraps quota check + insert + audit atomically (`packages/services/src/monitor/create.ts:24-62`), and the audit insert is fail-closed — if it fails, the whole mutation rolls back (`packages/services/src/audit/emit.ts:92-101`). The probe pipeline is the opposite: `/updateStatus` does upsert → read → count → conditional update → incident insert as *separate statements, no transaction* (`apps/workflows/src/checker/index.ts:122-316`). **Rejected:** wrapping the pipeline in transactions too. At the time: throughput fear plus HTTP-driver transaction distrust. Consistency there rests on idempotency backstops instead. **This is the tradeoff that bothers me most in the entire codebase** — the quorum flip is a read-check-write across awaits, so two regions reporting simultaneously can both read the pre-flip status. The honest fix is serializing per-monitor (queue key = monitorId) or one transaction around flip + incident + trigger. It's on the roadmap; the war stories in Part 5 are the argument for prioritizing it.

## War story

**Caused: "why is this customer's monitor paused?"** Support escalation: a free-tier customer's monitor silently stopped checking. Root cause was legitimate — the free-plan limiter in workflows paused it (`apps/workflows/src/cron/monitor.ts:292-303`). The incident was that *nobody could tell*: that code path writes `monitor.active` directly, bypassing the services layer, so it emits **no audit row**. The support engineer queried `audit_log`, found nothing, and escalated what should have been a one-minute answer. Lesson: our proudest invariant — "every mutation is audited" — is only true for *user-facing* mutations. Pipeline and lifecycle writes (`checker/index.ts:227-231` too) go around the layer. The actor union already has `system`; adding system-actor audit rows is cheap and just needs doing. Say invariants precisely or they'll embarrass you in a support thread.

## Before you move on

1. A monitor has `active=true, deletedAt≠null`. What does the cron scheduler at `apps/workflows/src/cron/checker.ts:98-108` do with it? Go read the query — does it filter `deletedAt`? (This is a live bug-suspicion; resolve it with evidence.)
2. Write the 5-step migration plan to make `monitor.workspace_id` NOT NULL, including the query that finds violating rows and the deploy-order constraint from three apps sharing the DB.
3. Why is it *correct* for the status page to show "operational" while an incident row exists — and what UI copy decision does that force?
