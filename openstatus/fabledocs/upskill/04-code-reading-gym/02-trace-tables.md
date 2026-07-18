# Trace Tables

Fill each table yourself first (columns: step, file:line, value shape, owner, transformation, risk), then compare. Blank templates at the bottom.

## Trace 1 (UI → API): clicking "Delete" on a monitor

| Step | File:line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | dashboard action bar (find it: `components/data-table/monitors/data-table-action-bar.tsx`) | `{ id: number }` | UI | user intent → mutation input | double-click double-fire (mutation pending state?) |
| 2 | [trpc.ts:148-200](../../../packages/api/src/trpc.ts#L148-L200) | session + cookie → `ctx.workspace` | middleware | authn → tenant resolution | stale `workspace-slug` cookie |
| 3 | [monitor.ts:78-93](../../../packages/api/src/router/monitor.ts#L78-L93) | zod-parsed `{id}` | router | wire → service input; `NotFoundError` swallowed for idempotency | swallow hides genuine cross-tenant probing (also returns "success") |
| 4 | `deleteMonitor` service ([delete.ts](../../../packages/services/src/monitor/delete.ts) — open it) | monitor row | service | soft-delete write + audit | verify it sets `deletedAt` not a hard delete |
| 5 | [emit.ts:102-149](../../../packages/services/src/audit/emit.ts#L102-L149) | audit row with `before` | audit | snapshot serialization | secrets in snapshots (none on monitor) |
| 6 | client cache | — | TanStack Query | invalidation → refetch list | missing invalidation = ghost row in UI |

## Trace 2 (persistence): one `/updateStatus` request, error path, 3-region monitor, second region reporting

| Step | File:line | Value shape | Transformation | Risk |
| --- | --- | --- | --- | --- |
| 1 | [checker/index.ts:96-110](../../../apps/workflows/src/checker/index.ts#L96-L110) | `{monitorId:"42", status:"error", region:"ams", cronTimestamp}` | JSON → zod-typed | payload drift vs Go sender |
| 2 | [checker/index.ts:122-126](../../../apps/workflows/src/checker/index.ts#L122-L126) | upsert `monitor_status(42, ams)=error` | LWW write | out-of-order overwrites |
| 3 | [checker/index.ts:128-134](../../../apps/workflows/src/checker/index.ts#L128-L134) | monitor row, `status:"active"`, `regions:["ams","fra","iad"]` | DB read + zod parse | **snapshot is now the truth for this request** |
| 4 | [checker/index.ts:137-151](../../../apps/workflows/src/checker/index.ts#L137-L151) | `affectedRegions=["fra","ams"]`, count=2 | count same-status rows | includes rows from any age |
| 5 | [checker/index.ts:216](../../../apps/workflows/src/checker/index.ts#L216) | `2 >= 3/2` → true | quorum verdict | denominator = *current* regions list |
| 6 | [checker/index.ts:285-298](../../../apps/workflows/src/checker/index.ts#L285-L298) | guard `monitor.status !== "error"` passes; write `status="error"` | verdict → aggregate flip | concurrent request saw same snapshot |
| 7 | [checker/index.ts:301-316](../../../apps/workflows/src/checker/index.ts#L301-L316) | find open incident → none → insert `{startedAt: cronTimestamp}` | detection → record | TOCTOU find/insert |
| 8 | [alerting.ts:57-139](../../../apps/workflows/src/checker/alerting.ts#L57-L139) | joined notification rows → trigger insert → sends | fan-out | trigger-then-send at-most-once |

## Trace 3 (auth): custom API key on `GET /v1/monitor/:id`

| Step | File:line | Value shape | Transformation | Risk |
| --- | --- | --- | --- | --- |
| 1 | [auth.ts:57-62](../../../apps/server/src/libs/middlewares/auth.ts#L57-L62) | header string `os_ab12cd34…` | presence check | — |
| 2 | [auth.ts:159-167](../../../apps/server/src/libs/middlewares/auth.ts#L159-L167) | `prefix = key.slice(0,11)` | prefix → row lookup | prefix collision (8 hex chars) → wrong row, then hash mismatch → 401 (safe but confusing) |
| 3 | [auth.ts:169-183](../../../apps/server/src/libs/middlewares/auth.ts#L169-L183) | hash verify, expiry | secret → identity | timing characteristics of verify (bcrypt-style = OK) |
| 4 | [auth.ts:184-199](../../../apps/server/src/libs/middlewares/auth.ts#L184-L199) | `{ownerId, keyId, scopes, createdById}` | identity → workspace claim + debounced write | write-on-read path |
| 5 | [auth.ts:89-99](../../../apps/server/src/libs/middlewares/auth.ts#L89-L99) | workspace row (zod-validated) | claim → tenant context; wide-event enrichment | invalid workspace row → 400 |
| 6 | [v1/index.ts:147](../../../apps/server/src/routes/v1/index.ts#L147) | scopes | write gate (no-op for GET) | — |
| 7 | route handler | monitor row filtered by `workspace_id` | tenant-scoped read | forgetting the filter = IDOR (this is where reviews look) |

## Trace 4 (error): quota exceeded during monitor create (tRPC path)

| Step | File:line | Value | Transformation |
| --- | --- | --- | --- |
| 1 | [create.ts:25-28](../../../packages/services/src/monitor/create.ts#L25-L28) | `existing=10, limits.monitors=10` | comparison → `throw LimitExceededError("monitors", 10)` |
| 2 | [errors.ts:58-62](../../../packages/services/src/errors.ts#L58-L62) | `code:"LIMIT_EXCEEDED", message:"monitors limit reached (10)"` | domain error construction |
| 3 | tx rollback ([context.ts:40-47](../../../packages/services/src/context.ts#L40-L47)) | — | throw inside `transaction(fn)` aborts — nothing persisted, no audit row (nothing happened, correctly) |
| 4 | [monitor.ts router catch](../../../packages/api/src/router/monitor.ts#L98-L105) → [service-adapter.ts:57-65](../../../packages/api/src/service-adapter.ts#L57-L65) | `TRPCError TOO_MANY_REQUESTS` | domain code → transport code (+ documented reasoning in comment) |
| 5 | [trpc.ts:109-118](../../../packages/api/src/trpc.ts#L109-L118) | error shape + null `zodError` | formatter → wire |
| 6 | client | `error.data.code === "TOO_MANY_REQUESTS"` | wire → UX (ticket J9 makes this an upgrade prompt) |

Risk column for the whole trace: the *count* in step 1 raced with a concurrent create (R7) — the error path is clean, the success path is where the race hides.

## Trace 5 (async): one 30s-periodicity monitor through a single cron tick

| Step | File:line | Value shape | Risk |
| --- | --- | --- | --- |
| 1 | [cron/checker.ts:69-75](../../../apps/workflows/src/cron/checker.ts#L69-L75) | queue path for "30s"; `timestamp=now` | — |
| 2 | [cron/checker.ts:98-115](../../../apps/workflows/src/cron/checker.ts#L98-L115) | monitor rows → zod array parse (throws on any bad row!) | one malformed row fails the whole tick (`safeParse` then throw at L117–120) |
| 3 | [cron/checker.ts:153-183](../../../apps/workflows/src/cron/checker.ts#L153-L183) | per region: TaskInput ×2 (now and +30s) | double-scheduling semantics |
| 4 | [cron/checker.ts:336-352](../../../apps/workflows/src/cron/checker.ts#L336-L352) | task name `monitor-42-ams-1720684800000`; scheduleTime seconds | ms→s divide (L350) — unit boundaries |
| 5 | GCP → [checker.go:47-75](../../../apps/checker/handlers/checker.go#L47-L75) | JSON body → `HttpCheckerRequest` | secret check; fly-replay |
| 6 | [checker.go:276](../../../apps/checker/handlers/checker.go#L276) | up to `retry` attempts | conflates assertion-fail with network-fail for retry purposes |
| 7 | [checker.go:255](../../../apps/checker/handlers/checker.go#L255) | PingData → Tinybird | lost-on-failure |

Note step 2's failure mode — that's a **blast radius** observation: one workspace's weird row can stop *everyone's* 30s checks for a tick. Verify by reading L115–120 closely; this is confirmed behavior (throw), not hypothesis.

## Blank template

| Step | File:line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |

Do-it-yourself assignments: (a) subscriber verify (token → acceptedAt) through `packages/services/src/page-subscriber/verify.ts`; (b) a degraded→active recovery; (c) `workspace-slug` cookie lifecycle across login. Self-grade — Basic: steps and files right. Solid: value shapes concrete (real example values). Strong: every row has a non-empty, *specific* risk.
