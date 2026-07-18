# Pattern Catalog

Sixteen patterns you can now *recognize on sight* — here and in any other repo. Recognition, not name-dropping: for each, know the problem it solves and the failure mode that appears when it's missing.

---

## Pattern 1: Service verb per file

Problem it solves: business logic reusable across transports (tRPC, REST, Slack, MCP, jobs) without copy-paste; small review units.
General shape: `packages/services/src/<entity>/<verb>.ts` exporting `async function verbEntity({ ctx, input })`.
Real example: [create.ts](../../../packages/services/src/monitor/create.ts#L16-L63)
Second example: [upsert.ts](../../../packages/services/src/page-subscriber/upsert.ts#L52)
Why this implementation works: uniform signature means adapters are mechanical ([service-adapter.ts](../../../packages/api/src/service-adapter.ts#L18-L24)); one-file-per-verb keeps diffs reviewable.
Failure modes: verbs calling verbs can double-open transactions (mitigated by `withTransaction` reuse); "internal.ts" grab-bags growing into a second god-module.
Use it when: multiple entry points need the same mutation. Avoid when: a query is truly view-specific (then it's presentation, not domain).
Interview angle: "Where do you put business logic in a Node app?" — this is the service-layer answer with receipts.
Drill: pick `packages/services/src/maintenance/` and verify every verb follows the signature.

## Pattern 2: Context object with actor union

Problem it solves: every mutation needs to know *tenant* and *who is acting* — without threading five parameters.
General shape: `ServiceContext = { workspace, actor, db?, … }`; `Actor` is a discriminated union of 7 caller types.
Real example: [context.ts](../../../packages/services/src/context.ts#L13-L29)
Second example: consumed at [emit.ts:134-146](../../../packages/services/src/audit/emit.ts#L134-L146) (`extractActorId`, `tryGetActorUserId`).
Why it works: exhaustive `switch` over the union ([context.ts:60-97](../../../packages/services/src/context.ts#L60-L97)) — adding an actor type breaks compilation everywhere it matters.
Failure modes: context becoming a kitchen sink; optional fields (`db?`) silently defaulting (see Pattern 3).
Interview angle: discriminated unions + exhaustiveness is a top-3 TS interview topic ([Q-cards](../08-interview-prep/01-js-ts-node-deep-dive.md)).
Drill: add a hypothetical `{ type: "cli" }` actor on paper; list every file the compiler would force you to touch.

## Pattern 3: Reuse-or-open transaction (`withTransaction`)

Problem it solves: verbs must compose (a verb calling another verb) without nested transactions.
General shape: if ctx carries a tx, use it; else open one.
Real example: [context.ts:40-47](../../../packages/services/src/context.ts#L40-L47)
Second example: read-side twin `getReadDb` at [context.ts:56-58](../../../packages/services/src/context.ts#L56-L58).
Why it works: `isTx` uses drizzle's symbol-based `is()` because `instanceof` breaks across pnpm's duplicated module instances — the comment at L33–35 documents a real-world module-resolution landmine.
Failure modes: forgetting to pass `tx` (not `defaultDb`) to writes inside the block — writes escape the transaction; CLAUDE.md calls this out explicitly.
Interview angle: "How do you compose transactional operations?" — ambient-transaction pattern vs explicit unit-of-work.
Drill: find one write inside any `withTransaction` block and confirm it uses `tx`.

## Pattern 4: Fail-closed audit in the same transaction

Problem it solves: an audit trail you can trust — no mutation without a record.
General shape: `emitAudit(tx, ctx, entry)` inside the mutation's tx; a failed audit insert rolls back the mutation.
Real example: [emit.ts:92-149](../../../packages/services/src/audit/emit.ts#L92-L149)
Second example: caller at [create.ts:53-59](../../../packages/services/src/monitor/create.ts#L53-L59).
Why it works: atomicity makes "every mutation has an audit row" an **invariant**, not a hope. Auto-diff (`changedFields`) + no-op skip (L118–132) keeps rows meaningful.
Failure modes: availability cost — audit table trouble blocks all writes (deliberate trade); secrets in snapshots (countered by stripping, e.g. [upsert.ts:154-158](../../../packages/services/src/page-subscriber/upsert.ts#L154-L158)).
Use when: compliance/debugging value exceeds the availability risk. Avoid when: high-frequency machine writes (note the checker pipeline uses a *different*, non-blocking audit).
Interview angle: "fail-open vs fail-closed" — this is the concrete example to cite.
Drill: explain why metadata-only audit rows still land when the diff is empty (read L118–132's comment).

## Pattern 5: Precondition guard as first line (`requireScope`)

Problem it solves: authorization checked before any work, uniformly.
General shape: `requireScope(ctx, "write")` as line 1 of every write verb, before parse and tx.
Real example: [require-scope.ts:25-48](../../../packages/services/src/auth/require-scope.ts#L25-L48); call site [create.ts:21](../../../packages/services/src/monitor/create.ts#L21)
Why it works: no DB dependency → cheap rejection; placement rule documented in the docblock ("a failed check shouldn't open a tx just to roll it back").
Failure modes: forgetting it on a new verb (the repo treats this like a missing audit — reviewable by convention + a standard test case); the no-op for `user` actors means member-role RBAC is *absent by design* (docblock L19: "separate project").
Interview angle: authn vs authz, and *where* authz belongs in a layered app.
Drill: `grep -rL "requireScope" packages/services/src/*/create.ts` style audit — find any write verb missing it (and check its tests for `'rejects read-only actor'`).

## Pattern 6: Error taxonomy + adapter mapping

Problem it solves: services can't know their transport; transports need precise status codes.
General shape: typed `ServiceError` subclasses → per-transport translator.
Real example: [errors.ts](../../../packages/services/src/errors.ts#L1-L82) → [service-adapter.ts:32-87](../../../packages/api/src/service-adapter.ts#L32-L87)
Why it works: the `switch` on `err.code` is exhaustive; unknown errors get a generic message so internals never leak (L79–86); `LIMIT_EXCEEDED → 429` mapping is *documented in place* with its reasoning.
Failure modes: a second transport inventing different mappings (v1 REST uses 402 for quota — inconsistency already live); wrapping errors twice and losing `cause`.
Interview angle: "How do you design error handling across layers?"
Drill: write the equivalent mapping table for HTTP status codes from memory, then check.

## Pattern 7: Thin router / fat service

Problem it solves: keeps transport handlers ~10 lines: validate → call → map.
Real example: [monitor.ts router delete](../../../packages/api/src/router/monitor.ts#L78-L93)
Second example (counter-example, legacy): [v1 post.ts](../../../apps/server/src/routes/v1/monitors/post.ts#L44-L90) — inline Drizzle + manual quota checks, no audit.
Why it works: business rules exist once; routers can't drift.
Failure modes: "just one quick query" in a router — bypasses audit + Edge-safety (CLAUDE.md names this smell).
Interview angle: layering violations and how teams migrate off them incrementally.
Drill: diff the two examples and list every behavior the legacy path lacks.

## Pattern 8: Workspace scoping by construction (fetch-or-throw)

Problem it solves: tenant isolation (anti-IDOR) without remembering a WHERE clause at every call site.
General shape: `getXInWorkspace(tx, id, workspaceId)` helpers that throw `NotFoundError`.
Real example: [internal.ts:28-47](../../../packages/services/src/monitor/internal.ts#L28-L47)
Second example: relation validation [validateTagIds](../../../packages/services/src/monitor/internal.ts#L63-L84) — even join-table ids are ownership-checked (throws `ForbiddenError`).
Why it works: 404-for-other-tenants avoids existence leaks; helpers make the safe path the easy path.
Failure modes: raw queries that skip the helper; returning 403 vs 404 inconsistently (this repo: tags → 403, monitors → 404 — worth noticing).
Interview angle: IDOR — the #1 practical security question for CRUD apps.
Drill: [fake-code contrast #2](../04-code-reading-gym/03-fake-code-contrasts.md) builds on this.

## Pattern 9: Plan limits as data

Problem it solves: pricing changes without code changes; per-workspace overrides.
General shape: `workspace.limits` JSON column; checks compare counts against it.
Real example: [workspace.ts:22](../../../packages/db/src/schema/workspaces/workspace.ts#L22) (column), [create.ts:25-28](../../../packages/services/src/monitor/create.ts#L25-L28) (check)
Second example: SMS quota at [alerting.ts:72-121](../../../apps/workflows/src/checker/alerting.ts#L72-L121).
Failure modes: count-then-insert races; limits JSON drifting from its zod schema; UI showing stale limits.
Interview angle: "How would you implement plan-based feature gating?"
Drill: find where `limits` gets parsed/validated (follow `selectWorkspaceSchema`).

## Pattern 10: Idempotency via deterministic identity

Problem it solves: at-least-once delivery (cron, queues, retries) without duplicate effects.
Three implementations in one repo:
1. Cloud Tasks task name `monitor-{id}-{region}-{timestamp}` — enqueue dedupe ([cron/checker.ts:336](../../../apps/workflows/src/cron/checker.ts#L336), ALREADY_EXISTS swallowed at L192–202)
2. `notificationTrigger` unique insert — send dedupe ([alerting.ts:131-139](../../../apps/workflows/src/checker/alerting.ts#L131-L139))
3. Redis `workflow:email:{user}:{step}:{run}` keys — email dedupe ([cron/monitor.ts:425-444](../../../apps/workflows/src/cron/monitor.ts#L425-L444))
Why it works: identity derived from *business facts* (monitor, region, tick), not random UUIDs, so retries collide on purpose.
Failure modes: key too coarse (suppresses legitimate repeats) or too fine (dupes get through); Redis TTL expiry re-arming old workflows.
Interview angle: "How do you make a webhook/queue consumer idempotent?" — cite all three variants.
Drill: for each of the three, state what *business event* the key encodes and what a wrong key would cause.

## Pattern 11: Quorum-based state transition

Problem it solves: one flaky region shouldn't page anyone.
General shape: flip aggregate state only when `affected >= regions/2` (or single-region).
Real example: [checker/index.ts:216](../../../apps/workflows/src/checker/index.ts#L216)
Why it works: per-region state (`monitor_status`) is separated from aggregate state (`monitor.status`), so evidence accumulates before the expensive transition (incident + notifications).
Failure modes: the read-count-then-write isn't transactional — concurrent updates can double-fire (mitigated by Pattern 10.2); `regions` list edited mid-flight changes the denominator.
Interview angle: distributed consensus-lite; "how do you avoid alert flapping?"
Drill: compute the behavior for a 2-region monitor with 1 error report. (1 >= 2/2 → flips. Is that intended? Defend or critique.)

## Pattern 12: Upsert for last-writer-wins state

Problem it solves: per-(monitor, region) status needs exactly one row, updated frequently.
Real example: [alerting.ts:276-302](../../../apps/workflows/src/checker/alerting.ts#L276-L302) — `onConflictDoUpdate` on composite target.
Second example: subscriber merge path in [upsert.ts:119-146](../../../packages/services/src/page-subscriber/upsert.ts#L119-L146) (read-then-branch upsert, application-level).
Failure modes: LWW erases causality — a delayed "active" from a slow region can overwrite a fresh "error"; no version column to detect it.
Interview angle: upsert vs insert-then-update; when you need optimistic concurrency instead.
Drill: propose the minimal schema change to detect out-of-order writes (hint: compare `cronTimestamp`).

## Pattern 13: Zod at every boundary

Problem it solves: runtime data (HTTP, DB rows, queue payloads) can't be trusted by the type system.
Real examples: queue payload parse [checker/index.ts:96-99](../../../apps/workflows/src/checker/index.ts#L96-L99); **DB reads re-validated** `selectMonitorSchema.parse(row)` [create.ts:61](../../../packages/services/src/monitor/create.ts#L61); Tinybird responses typed by pipe schemas [client.ts:74-97](../../../packages/tinybird/src/client.ts#L74-L97); OpenAPI request validation [post.ts:15-42](../../../apps/server/src/routes/v1/monitors/post.ts#L15-L42).
Why it works: parse, don't validate — outputs are *typed values*, and DB rows with drifted data fail loudly (e.g. regions string → array transform).
Failure modes: parse cost on hot paths; schema drift between duplicated schemas (tRPC input vs service input — kept intentionally separate here, see [monitor.ts:39](../../../packages/api/src/router/monitor.ts#L39)).
Interview angle: "How do you get end-to-end type safety?" — the honest answer includes runtime validation, not just TS.
Drill: find a boundary *without* a parse (hint: Go checker's JSON marshal of assertions at [checker.go:98-104](../../../apps/checker/handlers/checker.go#L98-L104)) and describe the risk.

## Pattern 14: Wide-event canonical logging + tail sampling

Problem it solves: per-request observability without log spam or cost blowup.
General shape: one `event` object accumulated across middleware/handler, emitted once at request end; sampling decided *after* outcome is known (keep all errors/slow, sample 20% of the rest).
Real example: [server index.ts:109-179](../../../apps/server/src/index.ts#L109-L179), `shouldSample` at L90–100
Second example: [workflows index.ts:42-62](../../../apps/workflows/src/index.ts#L42-L62) (adds 4xx at 50%); handler enrichment at [checker/index.ts:153-162](../../../apps/workflows/src/checker/index.ts#L153-L162).
Why it works: tail sampling keeps 100% of what you'll actually query during an incident.
Failure modes: event mutation after emit; PII accumulating in the wide event; head-sampling by mistake (deciding before you know the outcome).
Interview angle: observability questions ("how would you know this broke?") — see [05-quality-engineering/06](../05-quality-engineering/06-observability-and-operations.md).
Drill: list which fields the `/updateStatus` handler adds to the wide event and why each helps an on-call engineer.

## Pattern 15: Retry with typed effects (Effect)

Problem it solves: transient failure handling with bounded, observable retries.
Real example: task creation [cron/checker.ts:185-212](../../../apps/workflows/src/cron/checker.ts#L185-L212) — `Effect.retry({ times: 3, exponential })`, expected-error escape hatch (`ALREADY_EXISTS`), `Effect.either` to collect instead of throw.
Second example: notification sends [alerting.ts:144-175](../../../apps/workflows/src/checker/alerting.ts#L144-L175).
Failure modes: retrying non-idempotent effects (safe here only because of Pattern 10); retry amplification under provider outage; Go side uses a different library (`backoff` at [checker.go:276](../../../apps/checker/handlers/checker.go#L276)) — two retry stacks to keep mentally aligned.
Interview angle: "retries done wrong" is a classic system-design probe (thundering herd, amplification).
Drill: compute worst-case attempts for one monitor's failing probe end to end (Go retries × task retries × notification retries).

## Pattern 16: Graceful noop fallback for optional infrastructure

Problem it solves: dev/test shouldn't require every SaaS dependency.
Real example: `NoopTinybird` when token/URL missing ([client.ts:18-35](../../../packages/tinybird/src/client.ts#L18-L35))
Second example: dev API-key bypass ([auth.ts:246-261](../../../apps/server/src/libs/middlewares/auth.ts#L246-L261)).
Why it works: the null-object pattern — same interface, empty behavior — beats `if (tinybird)` scattered everywhere.
Failure modes: noop silently active in production (misconfigured env = data quietly discarded); dev bypass semantics leaking into prod builds (guarded here by `NODE_ENV` check at L154).
Interview angle: null-object pattern + config-driven behavior.
Drill: what happens today if `TINY_BIRD_API_KEY` is unset in production? Trace [client.ts:20-28](../../../packages/tinybird/src/client.ts#L20-L28) and answer precisely — this is a great "read the code, don't guess" exercise.
