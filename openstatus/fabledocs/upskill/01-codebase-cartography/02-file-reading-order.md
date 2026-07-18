# File Reading Order

Thirty files, ordered so each one makes the next easier. Per file: why it matters, what to look for, what to ignore. Junior path = 1–14. Mid path adds 15–24. Senior path adds 25–30.

## Junior path — the spine (1–14)

| # | File | Why / look for | Ignore |
| --- | --- | --- | --- |
| 1 | [CLAUDE.md](../../../CLAUDE.md) | The conventions constitution: services layer, audit pattern, scope enforcement, comment discipline | — |
| 2 | [package.json](../../../package.json) | Scripts + workspaces; note `packageManager: pnpm@10.26.0` | overrides |
| 3 | [turbo.json](../../../turbo.json) | Task graph; `dx → seed → migrate → env` | env allowlist details |
| 4 | [packages/db/src/db.ts](../../../packages/db/src/db.ts#L6-L12) | One Drizzle client over libSQL **HTTP** — that "http" matters later for transactions | — |
| 5 | [packages/db/src/schema/workspaces/workspace.ts](../../../packages/db/src/schema/workspaces/workspace.ts#L10-L35) | The tenant row; `plan` + `limits` JSON column | `dsn` (deprecated, comment says so) |
| 6 | [packages/db/src/schema/monitors/monitor.ts](../../../packages/db/src/schema/monitors/monitor.ts#L13-L73) | Central entity; soft delete via `deletedAt`; partial index L68–72; `regions` stored as comma-joined text | otel columns on first pass |
| 7 | [packages/services/src/context.ts](../../../packages/services/src/context.ts#L13-L47) | `Actor` union (7 actor types!), `ServiceContext`, `withTransaction` reuse-or-open | `isTx` symbol trick (read the comment though) |
| 8 | [packages/services/src/errors.ts](../../../packages/services/src/errors.ts#L1-L82) | Full error taxonomy with codes | — |
| 9 | [packages/services/src/monitor/create.ts](../../../packages/services/src/monitor/create.ts#L16-L63) | The canonical mutation verb — memorize the 6-step shape | — |
| 10 | [packages/services/src/audit/emit.ts](../../../packages/services/src/audit/emit.ts#L102-L149) | Fail-closed audit; auto-diff; no-op skip rule L118–132 | deepEqual internals |
| 11 | [packages/api/src/trpc.ts](../../../packages/api/src/trpc.ts#L148-L241) | Session → workspace resolution via cookie; test escape hatch L161–169 | analytics `after()` block on first pass |
| 12 | [packages/api/src/router/monitor.ts](../../../packages/api/src/router/monitor.ts#L77-L106) | A thin router: parse → service → error map | the long input schemas |
| 13 | [packages/api/src/service-adapter.ts](../../../packages/api/src/service-adapter.ts#L32-L87) | Error translation; note L79–86: unknown errors never leak messages | — |
| 14 | [apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx](../../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L43-L97) | Client component: `useTRPC()` + TanStack Query, URL state via nuqs | table plumbing |

## Mid path — the distributed system (15–24)

| # | File | Why / look for |
| --- | --- | --- |
| 15 | [apps/workflows/src/index.ts](../../../apps/workflows/src/index.ts#L102-L194) | Second Hono app; wide-event logging middleware; route mounts |
| 16 | [apps/workflows/src/cron/checker.ts](../../../apps/workflows/src/cron/checker.ts#L65-L183) | Cron → Cloud Tasks fan-out; maintenance exclusion subquery; the batched status fetch labeled "N+1 fix" L127–151 |
| 17 | [apps/workflows/src/cron/checker.ts](../../../apps/workflows/src/cron/checker.ts#L245-L356) | Task construction: payload per jobType, deterministic task *name* = idempotency key L336 |
| 18 | [apps/checker/handlers/checker.go](../../../apps/checker/handlers/checker.go#L47-L120) | Go probe handler: shared-secret auth L52, fly region replay L58–67, per-request HTTP client |
| 19 | [apps/checker/handlers/checker.go](../../../apps/checker/handlers/checker.go#L120-L332) | Retry closure, assertion eval, Tinybird event, conditional `UpdateStatus` calls |
| 20 | [apps/checker/checker/update.go](../../../apps/checker/checker/update.go#L29-L93) | Status change → *another* Cloud Task → workflows `/updateStatus` |
| 21 | [apps/workflows/src/checker/index.ts](../../../apps/workflows/src/checker/index.ts#L86-L356) | The alerting brain: quorum L216, incident create/resolve, monitor.status writes |
| 22 | [apps/workflows/src/checker/alerting.ts](../../../apps/workflows/src/checker/alerting.ts#L17-L255) | Notification fan-out; SMS quota; dedupe-by-insert L131–139 |
| 23 | [apps/server/src/libs/middlewares/auth.ts](../../../apps/server/src/libs/middlewares/auth.ts#L53-L122) | API key auth: custom-key-then-Unkey; scopes plumbing |
| 24 | [apps/status-page/src/proxy.ts](../../../apps/status-page/src/proxy.ts#L15-L96) | Multi-domain routing + access control composed into one `composePageAction` |

## Senior path — judgment material (25–30)

| # | File | Why / look for |
| --- | --- | --- |
| 25 | [packages/services/test/helpers.ts](../../../packages/services/test/helpers.ts#L37-L50) | `withTestTransaction` rollback-signal trick; its documented constraints (no savepoints over libsql HTTP) |
| 26 | [packages/services/src/auth/require-scope.ts](../../../packages/services/src/auth/require-scope.ts#L25-L48) | RBAC as a one-line precondition; the actor-type no-op table in the docblock |
| 27 | [apps/server/src/routes/v1/monitors/post.ts](../../../apps/server/src/routes/v1/monitors/post.ts#L44-L90) | The *legacy* way: inline Drizzle + manual limit checks — contrast with #9 |
| 28 | [packages/services/src/page-subscriber/upsert.ts](../../../packages/services/src/page-subscriber/upsert.ts#L52-L239) | Unauthenticated mutation done carefully: plan gate, token capability, audit with `subscriber` actor |
| 29 | [apps/workflows/src/cron/monitor.ts](../../../apps/workflows/src/cron/monitor.ts#L47-L196) | Free-plan pause lifecycle: Redis idempotency, multi-step Cloud Tasks workflow |
| 30 | [apps/server/src/index.ts](../../../apps/server/src/index.ts#L89-L179) | Wide-event middleware + tail sampling — observability as code |

## How to read each file (transferable procedure)

1. **Skim imports** — they tell you the dependency direction before you read a line of logic.
2. **Find the exported symbols** — that's the contract.
3. **Pause-and-predict** — before reading a function body, guess its steps from the name; note where you were wrong (that's where the codebase deviates from convention, i.e., where bugs and interviews live).
4. **Write one sentence** — "This file owns ___ and must never ___."

Senior noticing to practice on this list: files 9 vs 27 implement the same feature two ways. Being able to say *why* the repo is migrating (audit invariant, reuse across transports, Edge-safety) and what the migration risks are (behavior drift, double validation) is a complete mid-level interview answer about refactoring strategy.
