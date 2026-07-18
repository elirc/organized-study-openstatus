# Systematic Debugging

Method first, then five repo-specific scenarios. The method is what interviews grade in debugging rounds; the scenarios are your practice reps.

## The method

1. **Reproduce** — before any theory, make it fail on demand (or capture the exact evidence if you can't).
2. **Narrow** — binary-search the pipeline: which *boundary* did good data last cross? This repo's boundaries are gift-wrapped for this: browser → tRPC → service → DB, and cron → queue → checker → /updateStatus → notifications.
3. **Hypothesize** — one falsifiable statement at a time.
4. **Test cheaply** — logs you already have (wide events!), a targeted query, a unit test — before attaching debuggers or redeploying.
5. **Fix the root cause** — not the symptom.
6. **Add regression coverage** — the bug class, not just the instance.

Actual tools here: wide-event logs in Axiom (every request has `request_id`, `duration_ms`, `outcome` — [server index.ts:109-179](../../../apps/server/src/index.ts#L109-L179)); Sentry on both Hono apps; `drizzle-kit studio` for DB inspection; Tinybird pipes for probe history; `bun test --watch` for cheap hypothesis tests; React Query devtools + the tRPC `loggerLink` (enabled in dev, [client.tsx:44-48](../../../apps/dashboard/src/lib/trpc/client.tsx#L44-L48)).

---

## Scenario 1: "I got paged twice for the same outage"

Reproduction: hard to reproduce on demand — start from evidence: two provider messages, same monitor, ~same time.
First question: same **cronTimestamp** or different? (Pull both `notificationTrigger` rows.)
Narrowing path:
1. Same cronTimestamp → dedupe insert failed to dedupe → inspect unique constraint on `notification_trigger` (schema file — *investigate*), and the catch-all at [alerting.ts:131-139](../../../apps/workflows/src/checker/alerting.ts#L131-L139).
2. Different cronTimestamps → two *transitions* fired → check `monitor_status` history and the non-transactional flip at [checker/index.ts:216-348](../../../apps/workflows/src/checker/index.ts#L216-L348): did two regions' `/updateStatus` calls interleave (both read stale `monitor.status`)?
3. Confirm via wide events: filter Axiom by monitorId, look at `status_update` payloads ([checker/index.ts:153-162](../../../apps/workflows/src/checker/index.ts#L153-L162)) and their request timestamps.
Useful probes: `SELECT * FROM notification_trigger WHERE monitor_id = ? ORDER BY cron_timestamp DESC LIMIT 10`.
Likely root causes: interleaved handlers passing the same guard; or flapping (error→active→error across two ticks — check whether that's a "bug" at all).
Regression test: simulate two concurrent `/updateStatus` posts in a Hono test and assert one trigger row.
Senior lesson: dedupe at the *effect* is the last line, not the design; serialization belongs at the state transition.
Interview version: narrate steps 1–3 as "I distinguish duplicate-effect from duplicate-cause before touching code."

## Scenario 2: "Monitor shows error in dashboard but the endpoint is fine"

Reproduction: create a monitor against your own endpoint; make it fail once; fix endpoint; wait a tick.
First question: is the bug in **probing** (checker) or **state** (workflows)?
Narrowing path:
1. Tinybird rows for the monitor: are recent probes succeeding? If yes → state problem, go to 3.
2. If probes still fail: assertions? `EvaluateHTTPAssertions` treats status-only when no assertions ([checker.go:334-338](../../../apps/checker/handlers/checker.go#L334-L338)); a stale body assertion fails a healthy endpoint. Also `degradedAfter`/timeout config.
3. State: recovery requires an `/updateStatus` with `active` — sent only when `req.Status != "active"` ([checker.go:231-241](../../../apps/checker/handlers/checker.go#L231-L241)) where `req.Status` is the status *at enqueue time* ([cron/checker.ts:157-158](../../../apps/workflows/src/cron/checker.ts#L157-L158)). Check `monitor_status` rows per region: did enough regions send recovery to pass quorum ([checker/index.ts:216](../../../apps/workflows/src/checker/index.ts#L216))? A monitor whose failing region was *removed from its regions list* can strand the count.
Likely root causes: stale assertion; quorum arithmetic with edited region list; missed recovery task.
Regression test: unit-test the quorum branch with a region list that shrank.
Senior lesson: edge-triggered systems strand state when the edge is missed; level-triggered reconciliation (periodic "recompute from evidence") is the structural fix.
Interview version: a perfect "debug a distributed state machine" narration.

## Scenario 3: "Creating a monitor 500s, but only from the dashboard — the REST API works"

Reproduction: create via UI (fails), same payload via `POST /v1/monitor` (works).
First question: which layer differs? (Same service? No — v1 doesn't use the service! [post.ts:44-90](../../../apps/server/src/routes/v1/monitors/post.ts#L44-L90) is inline.)
Narrowing path:
1. tRPC error in browser devtools — `zodError` present? Input schema mismatch ([monitor.ts:50-61](../../../packages/api/src/router/monitor.ts#L50-L61)) vs service schema re-parse ([create.ts:22](../../../packages/services/src/monitor/create.ts#L22)).
2. No zod error → check the adapter: which ServiceError code → which TRPC code ([service-adapter.ts:32-87](../../../packages/api/src/service-adapter.ts#L32-L87)). INTERNAL means an unexpected throw — Sentry has the `cause`.
3. Reproduce in a service test with the exact input — cheapest isolation.
Likely root causes: schema drift between the duplicated input schemas; audit action missing from the validation union (Drill 2's failure mode — creates throw only on the *service* path).
Regression test: shared fixture pushed through both surfaces.
Senior lesson: duplicated contracts need contract tests; "works on surface A" localizes instantly if you know the layering.

## Scenario 4: "Status page shows a blank page for one customer's custom domain"

Reproduction: `curl -H "x-forwarded-host: status.customer.com" localhost:3003/...` locally with their row seeded.
First question: routing (proxy) or rendering (page)?
Narrowing path:
1. Proxy logs — the `[proxy] request` / `[proxy] action` console lines ([proxy.ts:52-86](../../../apps/status-page/src/proxy.ts#L52-L86)) tell you which action fired and why.
2. `passthrough` on validation failure (L40–42) is **fail-open**: a page row failing `selectPageSchema` silently skips rewriting → generic 404/blank. Check: does their row parse? (New enum value? Bad locale?)
3. If action is `rewrite`: the target route + `statusPage.get` — does `lower(customDomain)` match exactly (L34)? Trailing dot? Port in host?
Likely root causes: schema-invalid page row (fail-open), case/format mismatch in domain, locale override.
Regression test: `composePageAction` unit tests for each accessType — it's pure, test it exhaustively.
Senior lesson: middleware that fails open needs *visibility* (a metric/log for validation failures) or an on-call will chase ghosts.

## Scenario 5: "CI tests pass locally but fail on main with 'monitors limit reached'"

Reproduction: run the suite twice locally without reseeding.
First question: is the failure in the code under test or in **shared test state**?
Narrowing path:
1. The failing test uses the seeded free workspace whose quota is 1-ish; leftover rows from earlier tests/aborted runs consume it.
2. This exact class is documented: `cleanQuotaGatedTables` exists *because* "tight-plan negative tests break randomly otherwise" ([helpers.ts:52-88](../../../packages/services/test/helpers.ts#L52-L88)).
3. Check whether the failing suite (a) wraps in `withTestTransaction` (rolls back, no pollution) or (b) writes committed rows without cleanup.
Likely root causes: a new test committing rows on the free workspace without `beforeAll(cleanQuotaGatedTables)`.
Regression fix: adopt the helper, or better, create-and-delete a dedicated workspace (the helpers' stated "prevailing convention", [helpers.ts:102-106](../../../packages/services/test/helpers.ts#L102-L106)).
Senior lesson: shared-fixture suites accumulate coupling; the fix is convention + helpers, and knowing *why* the repo already grew them.
Interview version: "flaky test" questions — answer with state-leak taxonomy: order dependence, shared fixtures, time, randomness, concurrency.

---

Drill: pick scenario 2 or 4 and write the full narration (5 min, out loud) as if in a debugging round: symptom → boundary question → two probes → root cause → regression test. Self-grade — Basic: reaches a plausible cause. Solid: probes are cheap-first and each eliminates half the space. Strong: ends with the *class* fix and what telemetry would have made it a 5-minute bug.
