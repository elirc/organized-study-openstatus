# Mid-Level Feature Tickets

Ten cross-layer tickets. House rule for all of them: **write the design note first** (½ page: approach, files, risks, test plan, rollback), pretend-submit it for review, then implement. Each includes risk + rollback sections because that's what distinguishes mid from junior work.

---

## Ticket M1: Migrate `GET /v1/monitor/:id` from inline Drizzle to the services layer
Difficulty: Medium — 1–2 days. Skills: incremental migration, contract preservation.
Story: As a maintainer, I want v1 routes on the services layer so scope checks and audit happen uniformly (the code announces this plan at [v1/index.ts:143-147](../../../apps/server/src/routes/v1/index.ts#L143-L147)).
Read first: [get.ts](../../../apps/server/src/routes/v1/monitors/get.ts) + its test; service twin [list.ts / getMonitor](../../../packages/services/src/monitor/list.ts); adapter needs: build a `ServiceContext` from `c.get("workspace")` + `c.get("apiKey")` (actor `{type:"apiKey", keyId, scopes}` — [context.ts:13-20](../../../packages/services/src/context.ts#L13-L20)).
Design note must cover: response-shape equality (byte-level JSON comparison test between old and new), error-code mapping (OpenStatusApiError vs ServiceError), keeping the route-level `requireWriteScope` as defense-in-depth.
Risk: silent contract drift. Mitigation: snapshot the current responses in tests *before* refactoring.
Rollback: revert commit; no schema involvement.
Interview story potential: "I executed one step of a documented legacy migration without breaking a public API" — the single best mid-level story shape.

## Ticket M2: Merge the doubled global-metrics queries
Difficulty: Medium — 1–2 days. Skills: tRPC/Tinybird, measurement-driven change.
Anchor: the "HMM: why do we need two queries?" comment at [client.tsx:63-78](../../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L63-L78); the `trpc.tinybird.globalMetrics` procedure (in [router/tinybird](../../../packages/api/src/router/tinybird)).
Design note: option A — one procedure accepting `{http: ids, tcp: ids}`; option B — keep two, dedupe client-side. Choose with evidence (pipe shapes, payload sizes).
Acceptance: one network request; identical rendered metrics; loading state unified.
Risk: pipe parameter limits on id lists. Rollback: revert; purely additive procedure can ship beside the old one.
Interview story: "removed a self-documented wart with a measured, backward-compatible API change."

## Ticket M3: System-actor audit rows for pipeline mutations (risk R5)
Difficulty: Medium–Hard — 2–3 days. Skills: invariants, cross-app reasoning.
Story: As a support engineer, when a monitor was paused or status-flipped by the system, I want audit_log to say so.
Anchors: pause write [cron/monitor.ts:292-303](../../../apps/workflows/src/cron/monitor.ts#L292-L303); status flip [checker/index.ts:227-231](../../../apps/workflows/src/checker/index.ts#L227-L231); emitAudit + `system` actor already exist ([context.ts:18](../../../packages/services/src/context.ts#L18), [makeSystemCtx](../../../packages/services/test/helpers.ts#L195-L204)).
Design note must cover: new action names added to the validation union (CLAUDE.md requirement); *volume* — status flips are frequent; decide flip-audit vs pause-audit-only, justify; workflows currently doesn't import `@openstatus/services` for this (check dependency direction).
Risk: audit table growth; fail-closed semantics in a pipeline that must not block (consider fail-open here — and defend the difference from the user-path choice!).
Rollback: feature-flag by env var; actions are additive.
Interview story: "extended an audit invariant to system actors, consciously choosing different failure semantics per path" — extremely strong tradeoff story.

## Ticket M4: Rate-limit subscriber self-signup
Difficulty: Medium — 2 days. Skills: abuse prevention, Redis, Edge constraints.
Anchors: [upsert.ts](../../../packages/services/src/page-subscriber/upsert.ts#L52-L239); Upstash client `packages/upstash`; existing `lib/rate-limit` in dashboard (inventory it first).
Design: sliding window per (ip, pageId) and per email; 429 with Retry-After; count only paths that send email.
Risk: blocking legit corporate NATs — start log-only (shadow mode), then enforce. That two-phase rollout *is* the interview story.
Rollback: kill-switch env var.

## Ticket M5: Dead-man's switch for cron
Difficulty: Medium — 1–2 days. Skills: observability, absence detection.
Story: If `/cron/30s` stops firing, someone must find out from us, not from customers.
Anchors: cron routes ([workflows index.ts:190](../../../apps/workflows/src/index.ts#L190)); summary logs [cron/checker.ts:225-231](../../../apps/workflows/src/cron/checker.ts#L225-L231).
Design: per-periodicity heartbeat (Redis timestamp or external service like a self-monitored OpenStatus monitor — dogfooding!); alert when now - last > 2×period.
Interview story: "designed absence detection for a scheduler" — rare and memorable.

## Ticket M6: Implement JSON body assertions in the Go checker
Difficulty: Medium–Hard — 3–5 days. Skills: Go, cross-language contracts.
Anchor: the TODO at [checker.go:364-365](../../../apps/checker/handlers/checker.go#L364-L365); TS schema `jsonBodyAssertion` already exists ([router/monitor.ts imports](../../../packages/api/src/router/monitor.ts#L1-L7), definition in `packages/assertions`).
Design note: JSONPath subset supported; document divergence risk between TS validation and Go evaluation (the *same* assertion must mean the same thing in both languages — propose a shared test-vector file both suites consume).
Risk: silently accepted-but-unevaluated assertions today become behavior-changing — existing monitors with JSON assertions start failing. Migration: evaluate in shadow mode first (log mismatch, don't fail).
Rollback: config flag in the checker.
Interview story: "closed a cross-language contract gap with shared test vectors and a shadow rollout" — senior-flavored.

## Ticket M7: Paginate the audit-log list end to end
Difficulty: Medium — 2–3 days. Skills: cursor pagination, UI tables.
Anchors: [audit list service](../../../packages/services/src/audit/list.ts), [auditLog router](../../../packages/api/src/router/auditLog.ts), dashboard `audit-logs` route. First: verify current behavior (limit? unbounded?).
Design: cursor = `(id)` descending (monotonic per [helpers.ts:206-211](../../../packages/services/test/helpers.ts#L206-L211) reasoning); keep filters (entityType/action) composable.
Risk: infinite-scroll cache invalidation on new writes. Rollback: additive params, old callers unaffected.

## Ticket M8: `Retry-After` + structured quota errors across surfaces
Difficulty: Medium — 2 days. Skills: API contracts, error taxonomy.
Story: clients should programmatically distinguish "upgrade plan" from "slow down."
Anchors: [service-adapter.ts:57-65](../../../packages/api/src/service-adapter.ts#L57-L65), [errors.ts:58-62](../../../packages/services/src/errors.ts#L58-L62), v1's 402 usage [post.ts:59-64](../../../apps/server/src/routes/v1/monitors/post.ts#L59-L64).
Design: add `limit`, `max` fields to the error payload (LimitExceededError already carries them — surface, don't invent); tRPC `data` extension; v1 response body extension **without changing status codes** (contract).
Interview story: "made quota errors machine-readable across two API surfaces without breaking either."

## Ticket M9: Component-scoped subscriber emails
Difficulty: Hard — 3–5 days. Skills: joins, batch email, product thinking.
Story: subscribers who picked components should only get reports affecting those components.
Anchors: scope rows `pageSubscriberToPageComponent` ([upsert.ts:126-135](../../../packages/services/src/page-subscriber/upsert.ts#L126-L135)); status-report → components join (`statusReportsToPageComponents`, [statusPage.ts:91](../../../packages/api/src/router/statusPage.ts#L91)); find where report-published emails are sent (status-report service / workflows emails — inventory first, it's part of the ticket).
Design: set-intersection in SQL, not JS; empty scope = all components (verify existing semantics!).
Risk: silently *not* notifying someone is worse than over-notifying — add an audit/metadata trail of recipient selection.
Interview story: "shipped a filtering feature where the failure mode was silence, and designed for auditability of who was excluded."

## Ticket M10: Per-monitor serialization for `/updateStatus` (risk R1, step 1)
Difficulty: Hard — 3–5 days. Skills: concurrency control.
Story: eliminate the interleaving window on `monitor.status`.
Anchors: [checker/index.ts:122-348](../../../apps/workflows/src/checker/index.ts#L122-L348); Redis available.
Design options to compare in the note: (a) Redis lock per monitorId (TTL, lost-lock semantics), (b) optimistic CAS: `UPDATE monitor SET status=? WHERE id=? AND status=?` and branch on affected-rows, (c) single transaction. Recommend (b) — no new infra, no lock liveness issues; the write itself becomes the arbiter.
Test plan: two concurrent requests in a Hono test; exactly one notification trigger row.
Rollback: pure code change; revert restores current behavior.
Interview story: the best concurrency story in the repo — "I found a read-modify-write race in an alerting pipeline and fixed it with compare-and-swap, proving it with a concurrent test."

---

Sequencing advice: M1 → M3 → M10 is a coherent arc (adopt the layer, extend its invariant, harden its weakest consumer) that yields three interlocking interview stories.
