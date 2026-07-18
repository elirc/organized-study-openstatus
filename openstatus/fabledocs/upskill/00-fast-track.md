# Fast Track — One Weekend

Goal: by Sunday night you can run OpenStatus locally, trace two end-to-end flows out loud, and have made one safe change with a passing test. Everything below marked __inferred__ comes from repo manifests/CI, not from a run performed while writing this doc (see [verification log](09-reference/verification-log.md)).

## Saturday morning — install and run

Requirements (from [README.md](../../README.md#L93-L101)): Node ≥ 20, pnpm ≥ 8.6.2, Bun, Turso CLI. Windows users: the Turso CLI and some `bun`-based scripts assume a POSIX shell; Docker is the smoother path.

**Option A — Docker** (__inferred__ from [README.md](../../README.md#L58-L76)):

```sh
cp .env.docker.example .env.docker
docker compose up -d
# Dashboard: http://localhost:3002 — Status pages: http://localhost:3003
```

**Option B — manual** (__inferred__ from [README.md](../../README.md#L102-L142) and [turbo.json](../../turbo.json)):

```sh
pnpm install
turso dev --db-file openstatus-dev.db   # terminal 1: local libSQL on :8080
pnpm dx                                 # terminal 2: env → migrate → seed (turbo task chain)
pnpm dev:dashboard                      # then open http://localhost:3000
```

Note the task graph in [turbo.json](../../turbo.json): `dx` depends on `seed`, which depends on `migrate`, which depends on `env`. That's the repo teaching you its own setup order.

**Run one test suite** (__inferred__ from [.github/workflows/test.yml](../../.github/workflows/test.yml) and [CLAUDE.md](../../CLAUDE.md)) — CI does: sqld container → `pnpm migrate` + `pnpm seed` in `packages/db` → `pnpm test` at root. Locally, with `turso dev` running:

```sh
cd packages/db && pnpm migrate && pnpm seed && cd ../..
pnpm test               # turbo fans out; server + services use `bun test`
```

## Saturday afternoon — the first 10 files, in order

Open these in order. For each: read it fully, then say aloud what it owns.

1. [CLAUDE.md](../../CLAUDE.md) — the repo's own conventions doc: services layer, audit log, scope enforcement. This is the constitution; everything else is case law.
2. [package.json](../../package.json) — workspaces, scripts. Note `dev:dashboard` filters to two packages.
3. [packages/db/src/schema/monitors/monitor.ts](../../packages/db/src/schema/monitors/monitor.ts#L13-L73) — the central entity. Note `deletedAt` soft delete and the partial index at L68–72.
4. [packages/db/src/schema/workspaces/workspace.ts](../../packages/db/src/schema/workspaces/workspace.ts#L10-L35) — the tenant. `limits` is a JSON text column: plan quotas as data.
5. [packages/services/src/context.ts](../../packages/services/src/context.ts#L13-L47) — `Actor` union + `ServiceContext` + `withTransaction`. The spine of every mutation.
6. [packages/services/src/monitor/create.ts](../../packages/services/src/monitor/create.ts#L16-L63) — a complete service verb: scope check → parse → transaction → quota → insert → audit. Memorize this shape.
7. [packages/api/src/service-adapter.ts](../../packages/api/src/service-adapter.ts#L32-L87) — how service errors become tRPC errors. Error taxonomy in one screen.
8. [packages/api/src/trpc.ts](../../packages/api/src/trpc.ts#L148-L241) — how a session becomes a workspace-scoped context (`enforceUserIsAuthed`).
9. [apps/workflows/src/cron/checker.ts](../../apps/workflows/src/cron/checker.ts#L65-L243) — how monitors become Cloud Tasks probe jobs.
10. [apps/workflows/src/checker/index.ts](../../apps/workflows/src/checker/index.ts#L86-L356) — how probe results become incidents and notifications.

## Saturday evening — trace two flows aloud

Do these as teach-backs: explain each flow out loud as if an interviewer asked "walk me through how X works in a codebase you know." Full trace tables live in [01-codebase-cartography/05-key-flows.md](01-codebase-cartography/05-key-flows.md).

**Flow 1 — creating a monitor:** dashboard client ([client.tsx](../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L43-L46) shows the tRPC hook style) → `monitorRouter` ([monitor.ts](../../packages/api/src/router/monitor.ts#L77-L106)) → `createMonitor` service ([create.ts](../../packages/services/src/monitor/create.ts#L16-L63)) → audit row ([emit.ts](../../packages/services/src/audit/emit.ts#L102-L149)).

**Flow 2 — a probe failing:** cron fan-out ([cron/checker.ts](../../apps/workflows/src/cron/checker.ts#L65-L243)) → Go checker ([checker.go](../../apps/checker/handlers/checker.go#L47-L332)) → `/updateStatus` ([checker/index.ts](../../apps/workflows/src/checker/index.ts#L86-L356)) → incident + notifications ([alerting.ts](../../apps/workflows/src/checker/alerting.ts#L17-L255)).

Pause-and-predict before reading: *when 3 of 6 regions report an error, does the monitor flip to error?* Then check the quorum condition at [checker/index.ts:216](../../apps/workflows/src/checker/index.ts#L216).

## Sunday — one safe change + one test

Pick ticket J1 or J2 from [06-contribution-practice/01-good-first-tickets.md](06-contribution-practice/01-good-first-tickets.md) (both are test-only or additive). The template change: add a missing test that `updateMonitorRetry` emits a `monitor.update` audit row, copying the shape of the existing happy-path test at [monitor.test.ts](../../packages/services/src/monitor/__tests__/monitor.test.ts#L79-L108) — use `withTestTransaction` + `expectAuditRow` from [helpers.ts](../../packages/services/test/helpers.ts#L37-L50).

Run just that package's tests (__inferred__): `cd packages/services && bun test src/monitor`.

## Teach-back exercise (do not skip)

Stand up. Two minutes, out loud, no notes: *"OpenStatus is a status page and monitoring platform. When I click 'create monitor' in the dashboard, here's everything that happens, and here's what happens the first time that monitor's endpoint goes down."* If you stall anywhere, that's the file you re-read. This narration **is** a mid-level interview answer.

## What the fast path skips

The Slack agent, MCP server, ConnectRPC v2 API, private locations, the marketing `web` app, Stripe billing, Tinybird pipe internals, i18n on status pages, and the Go checker's TCP/DNS paths. All are covered (or explicitly scoped out) in modules 01–05.
