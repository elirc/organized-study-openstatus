# Verification Log

Running log of what was actually inspected while writing this curriculum. Anything not listed here that appears in the docs is labeled **inferred** at the point of use.

- **Date:** 2026-07-11 (curriculum authored in one session)
- **Repo state:** fresh clone at `openstatus/`, branch `main`, **zero git commits** (`git log` → "does not have any commits yet"). No git history was available; all "why" claims come from code and comments, not commit messages.
- **Commands run:** none that build/test the project. No `pnpm install`, no `pnpm test`, no `docker compose up` were executed while writing these docs. Every command in [command-cheatsheet.md](command-cheatsheet.md) is therefore marked **inferred** (from `package.json` scripts, `turbo.json`, `.github/workflows/*`, `DOCKER.md`, `CLAUDE.md`) unless noted otherwise.

## Files read in full

| File | What was verified |
| --- | --- |
| `package.json` (root) | workspaces, scripts, pnpm@10.26.0, turbo 2.6.3, biome, oxlint |
| `turbo.json` | task graph: `dx` → `seed` → `migrate` → `env`; env allowlist for `build` |
| `README.md`, `CLAUDE.md` | product identity, stack, manual/docker setup, services & audit conventions |
| `.github/workflows/test.yml` | CI runs sqld service container, `pnpm migrate`, `pnpm seed`, `pnpm test` |
| `packages/services/src/context.ts` | Actor union, ServiceContext, withTransaction, getReadDb, extractActorId |
| `packages/services/src/audit/emit.ts` | deepEqual, diffTopLevel, emitAudit fail-closed semantics, no-op skip rule |
| `packages/services/src/auth/require-scope.ts` | scope matrix by actor type, deny logging, ForbiddenError |
| `packages/services/src/errors.ts` | full ServiceError taxonomy |
| `packages/services/src/monitor/create.ts` | createMonitor: requireScope → parse → tx → limit check → insert → audit |
| `packages/services/test/helpers.ts` | withTestTransaction rollback trick, ctx factories, expectAuditRow |
| `packages/services/src/page-subscriber/upsert.ts` | self-signup upsert, token creation, audit with subscriber actor |
| `packages/api/src/trpc.ts` | tRPC init, enforceUserIsAuthed, workspace-slug cookie, `after()` analytics |
| `packages/api/src/root.ts` | edge + lambda router merge |
| `packages/api/src/service-adapter.ts` | toServiceCtx, toTRPCError full mapping |
| `packages/db/src/db.ts` | drizzle libsql/http client |
| `packages/db/src/schema/monitors/monitor.ts` | monitor table, partial index, soft delete |
| `packages/db/src/schema/workspaces/workspace.ts` | workspace table, limits JSON column, plan |
| `apps/server/src/index.ts` | Hono app, wide-event middleware, sampling, route mounts |
| `apps/server/src/routes/v1/index.ts` | authMiddleware + requireWriteScope mounting, OpenAPI doc |
| `apps/server/src/libs/middlewares/auth.ts` | x-openstatus-key, custom key → Unkey fallback, scopes, dev bypass |
| `apps/workflows/src/index.ts` | workflows Hono app, tail sampling, route mounts |
| `apps/workflows/src/cron/checker.ts` | sendCheckerTasks fan-out, maintenance exclusion, Effect retry, task naming |
| `apps/workflows/src/cron/monitor.ts` | free-plan pause workflow, Redis idempotency keys, Cloud Tasks steps |
| `apps/workflows/src/checker/index.ts` | /updateStatus handler, quorum logic, incident create/resolve |
| `apps/workflows/src/checker/alerting.ts` | triggerNotifications, SMS quota, notificationTrigger dedupe, upsertMonitorStatus |
| `apps/checker/handlers/checker.go` | HTTPCheckerHandler, retries, assertion eval, Tinybird send, UpdateStatus calls |
| `apps/checker/checker/update.go` | UpdateStatus → Cloud Tasks → workflows /updateStatus |
| `apps/dashboard/src/lib/trpc/client.tsx` | TRPCProvider, query client reuse |
| `apps/status-page/src/proxy.ts` | domain resolution, password cookie, composePageAction dispatch |

## Files read partially (line ranges noted where cited)

| File | Portion |
| --- | --- |
| `packages/services/src/monitor/update.ts` | L1–100 (updateMonitorGeneral, updateMonitorRetry) |
| `packages/services/src/monitor/internal.ts` | L1–90 (getMonitorInWorkspace, countMonitorsInWorkspace, validateTagIds) |
| `packages/services/src/monitor/__tests__/monitor.test.ts` | L1–130 (fixtures, createMonitor happy paths) |
| `packages/api/src/router/monitor.ts` | L1–120 (input schemas, delete/deleteMonitors/updateMonitors) |
| `packages/api/src/router/statusPage.ts` | L1–130 (public get query shape) |
| `packages/tinybird/src/client.ts` | L1–120 (OSTinybird, Noop fallback, pipe definitions) |
| `packages/db/src/schema/pages/page.ts` | L1–50 (page table: password, accessType, customDomain) |
| `packages/db/src/seed.mts` | L1–40 (seeded workspace id 1, team plan) |
| `apps/server/src/routes/v1/monitors/post.ts` | L1–90 (createRoute, plan-limit checks) |
| `apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx` | L1–100 (useTRPC + useQuery composition) |

## Directory listings inspected

`apps/*`, `packages/*`, `packages/db/src/schema/*`, `packages/services/src/*` (all entities), `apps/server/src/routes/*` (v1, mcp, rpc, slack, public), `apps/server/src/routes/v1/monitors/*`, `apps/workflows/src/*`, `apps/checker/*` (handlers, checker, pkg, cmd), `apps/dashboard/src/*` (app, components, lib), `apps/status-page/src/app/*`, `packages/notifications/*` (provider list), `.github/workflows/*`.

## Known uncertainties (labeled at point of use in the docs)

1. **Runtime behavior of every command** — nothing was executed. Ports (3000/3002/3003), `pnpm dx`, `turso dev` behavior come from README/CLAUDE.md/DOCKER.md.
2. **`notification_trigger` unique constraint** — dedupe in `alerting.ts` relies on an insert throwing; the exact unique index was not read from the schema file. Marked *investigate* where cited.
3. **Whether `statusPage.get` (public tRPC) can serve data for password-protected pages when called directly** (bypassing the proxy). The output schema is a `selectPublicPageSchemaWithRelation`; whether it strips enough is unverified. Marked *possible risk*, not a bug.
4. **libSQL HTTP transaction isolation level** — quota check inside `withTransaction` may or may not serialize concurrent creates. Marked *investigate*.
5. **Tinybird pipe definitions** (`packages/tinybird`) were read on the client side only; the actual `.pipe` files under `packages/tinybird` datasources were not inspected.
6. **apps/web, apps/docs, apps/screenshot-service, apps/ssh-server, apps/private-location, apps/railway-proxy** — only listed, not read. The curriculum deliberately centers on dashboard/server/workflows/checker/status-page/db/services.
7. Pre-existing `docsclaude/`, `ralph/`, `OPEN_SOURCE_UPSKILL_DOCS_MASTER_PROMPT.md`, `CRITIQUE-codex-vs-claude.md` trees exist at repo root; they were **not** used as sources and were left untouched.
