# Key Flows

Seven end-to-end traces. The rest of the curriculum rotates through these — drills in module 04, tests in module 05, and interview cards in module 08 each pick different flows. Line anchors verified 2026-07-11.

---

## Flow 1: Create a monitor (dashboard → tRPC → service → audit)

Why this flow matters: it is the repo's canonical mutation. Every other write (pages, notifications, maintenance…) is a variation of this shape.

Open these files first:
- [monitor.ts (router)](../../../packages/api/src/router/monitor.ts#L50-L77) — tRPC input schema preserving the wire contract
- [create.ts (service)](../../../packages/services/src/monitor/create.ts#L16-L63) — the whole verb on one screen
- [emit.ts](../../../packages/services/src/audit/emit.ts#L102-L149) — the audit side-effect

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Browser | [client.tsx](../../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L44-L46) (list page shows the hook style) | `useTRPC()` + TanStack Query issue the call | superjson-encoded input | stale cache after mutation |
| 2 | Edge (Next) | [trpc.ts](../../../packages/api/src/trpc.ts#L148-L200) | `enforceUserIsAuthed`: session → `resolveActiveWorkspace` via `workspace-slug` cookie | `ctx.user`, `ctx.workspace` | wrong-workspace bugs if cookie stale |
| 3 | Router | [monitor.ts](../../../packages/api/src/router/monitor.ts#L77-L93) | validate input (zod), call service, map errors | `CreateMonitorInput` | drift between tRPC schema and service schema |
| 4 | Adapter | [service-adapter.ts](../../../packages/api/src/service-adapter.ts#L18-L24) | `toServiceCtx`: workspace + `{type:"user"}` actor | `ServiceContext` | — |
| 5 | Service | [create.ts](../../../packages/services/src/monitor/create.ts#L21-L28) | `requireScope` (no-op for users) → parse → `withTransaction` → **quota check** `countMonitorsInWorkspace` | count vs `workspace.limits.monitors` | count-then-insert race (*investigate*: libSQL HTTP tx isolation) |
| 6 | Service | [create.ts](../../../packages/services/src/monitor/create.ts#L34-L51) | insert with defaults (regions/periodicity from plan) | `monitor` row; regions serialized `join(",")` | comma-joined text column: no FK, no validation at DB level |
| 7 | Service | [create.ts](../../../packages/services/src/monitor/create.ts#L53-L59) | `emitAudit` in same tx, `after` snapshot + metadata | `audit_log` row | fail-closed: audit failure rolls back the create |
| 8 | Router→Client | [service-adapter.ts](../../../packages/api/src/service-adapter.ts#L32-L87) | ServiceError → TRPCError (e.g. LIMIT_EXCEEDED → 429) | — | leaking internals (guarded at L79–86) |

Validation and authorization: zod at router **and** service ([create.ts:22](../../../packages/services/src/monitor/create.ts#L22) re-parses — defense in depth); authn in middleware; tenancy by construction (`workspaceId: ctx.workspace.id` at insert, [create.ts:44](../../../packages/services/src/monitor/create.ts#L44)).

Persistence and side effects: one insert + one audit row, both in one transaction. No external calls.

Tests that cover it: [monitor.test.ts](../../../packages/services/src/monitor/__tests__/monitor.test.ts#L79-L108) — happy path + audit assertion inside a rolled-back transaction.

What juniors usually miss: the input is parsed **twice** (router + service) on purpose — the service can't trust every transport (Slack, MCP, jobs also call it).

What seniors notice: the quota check and the insert are in the same transaction, but over libSQL HTTP the isolation guarantee for two *concurrent* transactions is worth verifying before relying on it (flagged in the [risk register](../09-reference/risk-register.md)).

Interview angle: "How do you enforce plan limits?" — mid-level answer names the check location, the error mapping to 429, and the race caveat.

Drill: without looking, write the 6 steps of a service verb in order (scope → parse → tx → fetch/guard → write → audit). Then diff against [create.ts](../../../packages/services/src/monitor/create.ts#L16-L63).
Self-grade — Basic: steps in order. Solid: + which errors each step throws. Strong: + why audit is fail-closed and what UX cost that choice has.

---

## Flow 2: Scheduling fan-out (cron → Cloud Tasks → per-region probes)

Why this flow matters: it's the background/async backbone — quotas, retries, idempotency, and region routing in one file.

Open these files first:
- [cron/checker.ts](../../../apps/workflows/src/cron/checker.ts#L65-L125) — selection query
- [cron/checker.ts](../../../apps/workflows/src/cron/checker.ts#L245-L356) — task construction

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Scheduler (external) | mounted at [index.ts:190](../../../apps/workflows/src/index.ts#L190) | `/cron/:periodicity` hit per interval, `CRON_SECRET`-authed | — | shared secret |
| 2 | Workflows | [cron/checker.ts:77-108](../../../apps/workflows/src/cron/checker.ts#L77-L108) | select active monitors of this periodicity, **excluding those under active maintenance** (subquery join) | monitor rows | maintenance linkage only via page components |
| 3 | Workflows | [cron/checker.ts:127-151](../../../apps/workflows/src/cron/checker.ts#L127-L151) | batch-fetch all per-region statuses (comment: "N+1 fix") | `Map<monitorId, status[]>` | — |
| 4 | Workflows | [cron/checker.ts:153-183](../../../apps/workflows/src/cron/checker.ts#L153-L183) | build TaskInput per (monitor × region); 30s periodicity double-schedules with +30s offset | `TaskInput` | deprecated regions skipped with error logs |
| 5 | Workflows | [cron/checker.ts:185-212](../../../apps/workflows/src/cron/checker.ts#L185-L212) | Effect.forEach concurrency 100, 3 retries exp backoff; `ALREADY_EXISTS` (code 6) treated as success | — | retry storm if GCP degraded |
| 6 | GCP | [cron/checker.ts:336-352](../../../apps/workflows/src/cron/checker.ts#L336-L352) | task name `monitor-{id}-{region}-{timestamp}` = **natural idempotency key**; provider-specific region header (fly/koyeb/railway) | base64 JSON payload | payload/Go struct skew across deploys |
| 7 | Checker | [checker.go:52-67](../../../apps/checker/handlers/checker.go#L52-L67) | secret check; fly-replay to correct region if misrouted | `HttpCheckerRequest` | — |

Validation and authorization: `Authorization: Basic ${CRON_SECRET}` end to end ([cron/checker.ts:343](../../../apps/workflows/src/cron/checker.ts#L343), [checker.go:52](../../../apps/checker/handlers/checker.go#L52)).

Persistence and side effects: none in SQL here — the side effect *is* enqueueing.

Tests that cover it: none found for `sendCheckerTasks` (no direct coverage found in `apps/workflows` — evidence: no test files under `apps/workflows/src/cron/`).

What juniors usually miss: idempotency comes from the deterministic **task name**, not from any lock. Re-running the cron can't double-schedule because GCP rejects duplicates (code 6), which the code deliberately swallows.

What seniors notice: scheduling reads `monitor.regions` as authoritative but quorum later divides by `monitor.regions.length` — a monitor edited mid-cycle can briefly disagree with in-flight tasks.

Interview angle: "Design a cron that doesn't double-fire" — answer with deterministic task naming + dedupe-on-enqueue, citing this file.

Drill: predict what happens if the same cron tick runs twice (two schedulers). Then verify via L192–202. Basic: "GCP dedupes". Solid: names the task-name key. Strong: also notes the 30s double-schedule interaction and the maintenance-exclusion subquery.

---

## Flow 3: Probe → status change → incident → notifications (the alerting pipeline)

Why this flow matters: this is the product's core promise ("get notified when down") and the richest reliability material in the repo.

Open these files first:
- [checker.go](../../../apps/checker/handlers/checker.go#L120-L332) — probe, retries, assertions
- [update.go](../../../apps/checker/checker/update.go#L29-L93) — status-change enqueue
- [checker/index.ts](../../../apps/workflows/src/checker/index.ts#L86-L356) — the alerting brain
- [alerting.ts](../../../apps/workflows/src/checker/alerting.ts#L17-L255) — provider fan-out

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Checker (Go) | [checker.go:120-183](../../../apps/checker/handlers/checker.go#L120-L183) | run probe with retry closure (`backoff`); evaluate assertions | `checker.Response` | JSON body assertion is a TODO (L364–365) |
| 2 | Checker | [checker.go:255-257](../../../apps/checker/handlers/checker.go#L255-L257) | send `PingData` to Tinybird `ping_response__v8`; failure only logged | analytics row | **silent data loss** on TB failure |
| 3 | Checker | [checker.go:205-253](../../../apps/checker/handlers/checker.go#L205-L253) | if state differs from `req.Status`, call `UpdateStatus` (error/degraded/active transitions) | `UpdateData` | fires per region |
| 4 | Checker | [update.go:29-93](../../../apps/checker/checker/update.go#L29-L93) | enqueue Cloud Task → workflows `/updateStatus` (queue `alerting`) | JSON | ordering not guaranteed across regions |
| 5 | Workflows | [checker/index.ts:86-126](../../../apps/workflows/src/checker/index.ts#L86-L126) | auth, zod parse, **upsert per-region status** | `monitor_status` row | — |
| 6 | Workflows | [checker/index.ts:137-165](../../../apps/workflows/src/checker/index.ts#L137-L165) | count affected regions with same status | `affectedRegionCount` | read-after-write across requests |
| 7 | Workflows | [checker/index.ts:216-348](../../../apps/workflows/src/checker/index.ts#L216-L348) | **quorum**: `affected >= regions/2 or single-region` → flip `monitor.status`, create/resolve incident | `incident` row | read-modify-write of `monitor.status` with **no transaction** — two concurrent `/updateStatus` calls can both pass the `monitor.status === "error"` guard (*possible risk*) |
| 8 | Workflows | [alerting.ts:70-139](../../../apps/workflows/src/checker/alerting.ts#L70-L139) | per channel: SMS quota check; **dedupe by inserting `notificationTrigger`** — duplicate insert throws → skip | trigger row | dedupe keyed to cronTimestamp (*investigate exact unique index*) |
| 9 | Workflows | [alerting.ts:144-239](../../../apps/workflows/src/checker/alerting.ts#L144-L239) | provider send with Effect retry ×3 exp backoff; failures logged, not re-queued | — | notification can be *recorded as sent* (trigger row) even if all sends fail |
| 10 | Workflows | [checker/index.ts:301-327](../../../apps/workflows/src/checker/index.ts#L301-L327) | on error-flip: `findOpenIncident` else insert incident + audit + notify | incident | TOCTOU between find and insert |

Validation and authorization: `CRON_SECRET` at [checker/index.ts:86-91](../../../apps/workflows/src/checker/index.ts#L86-L91); zod payload parse L96–99 returns 422.

Persistence and side effects: SQL writes (monitor_status upsert, monitor.status, incident), Tinybird events, provider webhooks/emails/SMS — **all without an outbox**; the trigger row is the only durable send-intent record.

Tests that cover it: no direct coverage found for `/updateStatus` (evidence: no test files under `apps/workflows/src/checker/`).

What juniors usually miss: state is compared against `req.Status` — the status *the task was enqueued with* ([cron/checker.ts:157-158](../../../apps/workflows/src/cron/checker.ts#L157-L158)) — not a fresh DB read. The pipeline is edge-triggered, not level-triggered.

What seniors notice: the quorum check counts rows in `monitor_status` (updated by concurrent requests), then branches on the *stale* `monitor.status` it selected earlier — the idempotency of the whole flow rests on the notificationTrigger insert, which is a good last line of defense but a thin one.

Interview angle: this flow *is* a system-design interview: "Design an uptime monitor with multi-region checks and alerting." See [08-interview-prep/04-system-design-from-this-repo.md](../08-interview-prep/04-system-design-from-this-repo.md).

Drill: build the trace table for the **recovery** path (error → active) yourself, from [checker/index.ts:218-249](../../../apps/workflows/src/checker/index.ts#L218-L249). Basic: steps listed. Solid: + `autoResolved` flag and where it's set ([checker/index.ts:67-75](../../../apps/workflows/src/checker/index.ts#L67-L75)). Strong: + explains why recovery only resolves incidents when the previous status was `error` (L233) and what happens on degraded→active.

---

## Flow 4: Public status page delivery (domain → access control → data)

Why this flow matters: it's the only *unauthenticated* read surface, so every mistake is public. Multi-tenancy, custom domains, and four access modes meet in one middleware.

Open these files first:
- [proxy.ts](../../../apps/status-page/src/proxy.ts#L15-L96) — the whole gate
- [statusPage.ts](../../../packages/api/src/router/statusPage.ts#L64-L130) — the public data query

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Edge middleware | [proxy.ts:18-28](../../../apps/status-page/src/proxy.ts#L18-L28) | resolve route from `x-forwarded-host` + path | `{prefix,…}` | host-header trust (Vercel-verified per comment L47–48) |
| 2 | Middleware | [proxy.ts:30-42](../../../apps/status-page/src/proxy.ts#L30-L42) | DB lookup: `lower(slug) = prefix OR lower(customDomain) = prefix` | page row | **DB query on every request** in middleware (latency + cost) |
| 3 | Middleware | [proxy.ts:64-80](../../../apps/status-page/src/proxy.ts#L64-L80) | `composePageAction`: password cookie / `?pw` param / SSO email / client IP → redirect, rewrite, or pass | action union | pure function = testable gate |
| 4 | Page (RSC) | `src/app/(status-page)/[domain]/[locale]/…` | render with data from public tRPC | — | — |
| 5 | tRPC public | [statusPage.ts:64-113](../../../packages/api/src/router/statusPage.ts#L64-L113) | `publicProcedure` fetches page + reports + maintenances + components + incidents in one relational query | `selectPublicPageSchemaWithRelation` | *possible risk*: this endpoint is keyed by slug only — whether password-gated pages leak data to a direct tRPC caller depends on the output schema stripping; unverified |
| 6 | Output guard | [statusPage.ts:76](../../../packages/api/src/router/statusPage.ts#L76) | `.output(selectPublicPageSchemaWithRelation.nullish())` — schema-enforced response shape | — | output schema is the real security boundary here |

Validation and authorization: access control lives in the **proxy layer**, not the data layer — a deliberate (documented-by-structure) choice with a defense-in-depth question attached (see risk register R6).

What juniors usually miss: `.output()` on a tRPC procedure is doing security work (field stripping), not just typing.

What seniors notice: password comparison flows through a cookie set from `createProtectedCookieKey(_page.slug)` — per-page cookie naming prevents one unlocked page from unlocking another.

Interview angle: "How would you add password protection to a multi-tenant public page?" — answer: gate at the edge, per-tenant cookie, and *also* enforce at the data layer; then note which half this repo implements.

Drill: list the four access types from [page.ts](../../../packages/db/src/schema/pages/page.ts#L33-L44) (password, accessType enum, authEmailDomains, allowedIpRanges) and where each is read in `composePageAction`'s inputs ([proxy.ts:64-80](../../../apps/status-page/src/proxy.ts#L64-L80)). Strong = you also explain why `sanitizeRedirectParam` exists (open-redirect defense).

---

## Flow 5: REST v1 request (API key → scopes → handler)

Why this flow matters: external contract + key-based authn/authz, and a live legacy-vs-modern seam.

Open these files first:
- [auth.ts](../../../apps/server/src/libs/middlewares/auth.ts#L53-L122) — authn
- [v1/index.ts](../../../apps/server/src/routes/v1/index.ts#L142-L161) — middleware order
- [post.ts](../../../apps/server/src/routes/v1/monitors/post.ts#L44-L90) — a handler

Trace:

| Step | Owner | File | What happens | Risk |
| --- | --- | --- | --- | --- |
| 1 | Middleware | [auth.ts:57-78](../../../apps/server/src/libs/middlewares/auth.ts#L57-L78) | read `x-openstatus-key`; `validateKey` | missing key → 401 |
| 2 | validateKey | [auth.ts:159-226](../../../apps/server/src/libs/middlewares/auth.ts#L159-L226) | custom key (prefix lookup + bcrypt-style hash verify + expiry + debounced lastUsed) → else Unkey → else super-admin sentinel → dev bypass in non-prod | Unkey legacy keys get blanket `write` scope |
| 3 | Middleware | [auth.ts:89-119](../../../apps/server/src/libs/middlewares/auth.ts#L89-L119) | load workspace; set `c.set("workspace")`, `c.set("apiKey")` with fallback placeholder + loud warn | attribution degradation logged |
| 4 | Scope gate | [v1/index.ts:143-147](../../../apps/server/src/routes/v1/index.ts#L143-L147) | `requireWriteScope()` — **primary** enforcement for v1 because handlers use inline Drizzle | comment documents the migration plan |
| 5 | Handler | [post.ts:44-90](../../../apps/server/src/routes/v1/monitors/post.ts#L44-L90) | manual quota checks (count, periodicity, regions) → 402 PAYMENT_REQUIRED | duplicates service logic; **no audit row** on this path |

What juniors usually miss: the same conceptual check (write allowed?) exists twice — route-level middleware here, `requireScope` in services — because v1 hasn't been migrated. Reading the comment at [v1/index.ts:143-147](../../../apps/server/src/routes/v1/index.ts#L143-L147) tells you which is intended to win long-term (defense-in-depth).

What seniors notice: 402 vs 429 inconsistency — v1 returns `PAYMENT_REQUIRED` for quota, tRPC maps `LIMIT_EXCEEDED` to `TOO_MANY_REQUESTS` ([service-adapter.ts:57-65](../../../packages/api/src/service-adapter.ts#L57-L65)). Two public meanings for the same domain event.

Interview angle: "Design API-key auth with rotation and least privilege" — custom-prefix + hash + expiry + scopes + debounced lastUsed is a complete concrete answer.

Drill: trace what a **read-only** key gets when it calls `POST /v1/monitor`. Which line rejects it? Verify against [require-scope.ts](../../../apps/server/src/libs/middlewares/require-scope.ts) (open it — not yet read in these docs, good practice).

---

## Flow 6: Subscriber self-signup (public mutation done carefully)

Why this flow matters: an unauthenticated write path — the hardest kind to get right.

Open first: [upsert.ts](../../../packages/services/src/page-subscriber/upsert.ts#L52-L239), then [statusPage.ts imports](../../../packages/api/src/router/statusPage.ts#L18-L25) to see the verbs the public router uses.

Trace:

| Step | Owner | File | What happens | Risk |
| --- | --- | --- | --- | --- |
| 1 | Public router | [statusPage.ts:18-25](../../../packages/api/src/router/statusPage.ts#L18-L25) | `upsertSelfSignupSubscriber` invoked from `publicProcedure` | no rate limit visible at this layer (*investigate*) |
| 2 | Service | [upsert.ts:59-71](../../../packages/services/src/page-subscriber/upsert.ts#L59-L71) | resolve page → workspace; **plan gate before any write** | free plan rejected |
| 3 | Service | [upsert.ts:76-90](../../../packages/services/src/page-subscriber/upsert.ts#L76-L90) | componentIds validated against the page (anti-IDOR for join rows) | — |
| 4 | Service | [upsert.ts:102-117](../../../packages/services/src/page-subscriber/upsert.ts#L102-L117) | already-verified → no-op return, **no audit** | response reveals subscription existence (mild enumeration) |
| 5 | Service | [upsert.ts:183-197](../../../packages/services/src/page-subscriber/upsert.ts#L183-L197) | new row: `crypto.randomUUID()` token + 7-day expiry | token = capability; emailed link |
| 6 | Service | [upsert.ts:211-223](../../../packages/services/src/page-subscriber/upsert.ts#L211-L223) | audit with `subscriber` actor; **token stripped from snapshot** | secrets-out-of-audit invariant |

What juniors usually miss: there's no `ctx` parameter — the function *builds its own* ServiceContext from the page's workspace because no authenticated caller exists (docblock L37–51 explains).

What seniors notice: audit for the pending-merge case passes `metadata: { componentIds }` because the row diff alone would be empty — the exact case the `emitAudit` no-op-skip exception was built for ([emit.ts:118-132](../../../packages/services/src/audit/emit.ts#L118-L132)).

Interview angle: "How do you secure an unauthenticated endpoint that writes to the DB?" — plan gate, input-scoped validation, capability tokens, expiry, no-op idempotency.

Drill: write the state machine (none → pending → verified → unsubscribed) and find the verb file for each transition in `packages/services/src/page-subscriber/`. Strong = you can say which transitions emit audit rows and which deliberately don't.

---

## Flow 7: Test run (CI and local)

Why this flow matters: knowing how tests *actually* execute is the difference between "tests pass on my machine" and diagnosing CI.

Trace:

| Step | Owner | File | What happens |
| --- | --- | --- | --- |
| 1 | CI | [test.yml:15-27](../../../.github/workflows/test.yml#L15-L27) | `sqld` (libSQL server) container on :8080; `DATABASE_URL=http://127.0.0.1:8080` |
| 2 | CI | [test.yml:51-57](../../../.github/workflows/test.yml#L51-L57) | `pnpm migrate` then `pnpm seed` in `packages/db` |
| 3 | Seed | [seed.mts](../../../packages/db/src/seed.mts#L26-L40) | workspace id 1 = `team` plan (`love-openstatus`); fixtures exported as `SEEDED_WORKSPACE_TEAM_ID` / `FREE_ID` ([monitor.test.ts:11-14](../../../packages/services/src/monitor/__tests__/monitor.test.ts#L11-L14)) |
| 4 | Turbo | root `pnpm test` | fans out to package `test` scripts (`bun test` in server/services), `cache: false` |
| 5 | Test | [helpers.ts:37-50](../../../packages/services/test/helpers.ts#L37-L50) | `withTestTransaction` wraps the body and **always rolls back** via a sentinel error |
| 6 | Assert | [helpers.ts:240-264](../../../packages/services/test/helpers.ts#L240-L264) | `expectAuditRow` reads audit_log through the same tx |

What juniors usually miss: tests share one seeded DB; isolation comes from the rollback wrapper + unique prefixes (`TEST_PREFIX` at [monitor.test.ts:31](../../../packages/services/src/monitor/__tests__/monitor.test.ts#L31)), not from a fresh DB per test.

What seniors notice: `withTestTransaction` documents its own constraint — no nesting, because libSQL HTTP lacks savepoints ([helpers.ts:29-35](../../../packages/services/test/helpers.ts#L29-L35)). Infrastructure limits shape test architecture.

Interview angle: "How do you keep integration tests fast and isolated without mocking the DB?" — rollback-transaction pattern, with its constraints.

Drill: read [with-test-transaction.test.ts](../../../packages/services/src/__tests__/with-test-transaction.test.ts) (the tests *of the test helper*) and summarize what guarantees it locks in. Strong = explain why testing your test infrastructure is worth the upkeep.
