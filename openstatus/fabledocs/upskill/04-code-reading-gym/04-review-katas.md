# Review Katas

Eight fake PRs. For each: read the intent + diff summary, write your review (findings graded **Blocking / Important / Optional**), then check the expected findings. All diffs are invented; the "files this resembles" anchors are real.

Grading yourself: Basic = catches the blocking issue. Solid = + the important ones with *reasons*. Strong = + suggests the smaller/safer version of the PR and phrases everything kindly and specifically.

---

## Kata 1: "Add `description` to monitor update"

Author intent: let users edit monitor descriptions from the dashboard.
Fake diff summary: adds `description` to the tRPC input, passes it into `tx.update(monitor).set({...})` in `updateMonitorGeneral`; no schema change (column exists); no test changes.
Files this resembles: [update.ts:29-71](../../../packages/services/src/monitor/update.ts#L29-L71), [monitor.ts router](../../../packages/api/src/router/monitor.ts#L63-L75).
Expected findings — Blocking: service input schema (`UpdateMonitorGeneralInput`) not updated — the `Input.parse` will strip the field silently; the feature no-ops. Important: no test asserting the audit diff includes `description`; tRPC input and service schema must both change (documented duality). Optional: dashboard form field + character limit mirroring `text` column expectations.
Good review comment example:
> The router accepts `description` but `UpdateMonitorGeneralInput.parse` (service side) will strip it, so the update silently no-ops — can you add it to the service schema and a test that `changedFields` contains `description`? (See how `name` flows through update.ts L44–67.)

## Kata 2: "Speed up /updateStatus by removing the extra queries"

Author intent: performance — drop the `affectedRegions` query and flip `monitor.status` immediately on any error report.
Fake diff summary: deletes the count query and the quorum condition; every error report flips status and notifies.
Files this resembles: [checker/index.ts:137-216](../../../apps/workflows/src/checker/index.ts#L137-L216).
Expected findings — Blocking: removes the anti-flapping quorum invariant — one flaky region pages every customer on that monitor. Important: the perf claim is unmeasured (two indexed queries); wide-event `duration_ms` would show this handler isn't the bottleneck. Optional: if perf is real, cache `monitor.regions` count, don't change semantics.
Lesson: performance PRs that change *behavior* need the behavior called out; reviewers exist to catch invariant deletion dressed as optimization.

## Kata 3: "Add Discord notifications for maintenance windows"

Author intent: notify channels when maintenance starts.
Fake diff summary: in the maintenance create verb, after the insert, `await sendDiscord(...)` directly inside `withTransaction`.
Files this resembles: [alerting.ts provider dispatch](../../../apps/workflows/src/checker/alerting.ts#L144-L176); service verb shape [create.ts](../../../packages/services/src/monitor/create.ts#L24-L62).
Expected findings — Blocking: external HTTP call **inside a DB transaction** — provider latency holds the tx open; provider failure rolls back a valid maintenance (or worse: tx retried → duplicate sends). Important: no idempotency key; no retry policy; provider fan-out belongs in the alerting layer, not a CRUD verb. Optional: audit metadata for the notification attempt.
This kata encodes the repo's most transferable reliability rule: **effects after commit** (and the honest note that the repo itself lacks an outbox — see [03/04](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md)).

## Kata 4: "Refactor: share one HTTP client in the Go checker"

Author intent: reuse `http.Client` across probe requests for efficiency.
Fake diff summary: hoists the client to package level, removes per-request construction.
Files this resembles: [checker.go:76-95](../../../apps/checker/handlers/checker.go#L76-L95).
Expected findings — Blocking: the per-request client is deliberate — the comment says "We need a new client for each request to avoid connection reuse" (L76); connection reuse would distort latency **measurements** (the product's core metric) via warm connections; also per-request `Timeout` differs per monitor. Important: redirect policy is configured per request from `req.FollowRedirects` (L81–94) — a shared client races on it. Optional: if allocation cost is proven, use `http.Transport` per config bucket with `DisableKeepAlives`.
Lesson: "obvious" efficiency refactors can invert a domain requirement. Read the comment above the code you delete.

## Kata 5: "Add rate limiting to subscriber signup"

Author intent: stop signup abuse.
Fake diff summary: in-memory `Map<ip, count>` in the tRPC public procedure, resets on deploy, limit 5/day.
Files this resembles: [statusPage.ts public router](../../../packages/api/src/router/statusPage.ts#L64), [upsert.ts](../../../packages/services/src/page-subscriber/upsert.ts#L52).
Expected findings — Blocking: in-memory state on Edge/serverless = per-instance counters — limit is ~meaningless and resets constantly; use the existing Upstash Redis (`packages/upstash`, used at [cron/monitor.ts:28](../../../apps/workflows/src/cron/monitor.ts#L28)) or the dashboard's `lib/rate-limit` (exists — verify shape). Important: keying by IP alone punishes corporate NATs; key by (ip, pageId). Optional: return 429 with Retry-After; count only *new-row* paths, not idempotent re-signups.

## Kata 6: "Make audit logging async for performance"

Author intent: mutations shouldn't wait for audit inserts.
Fake diff summary: replaces `await emitAudit(tx, …)` with a fire-and-forget `queueMicrotask` using the global `db`.
Files this resembles: [emit.ts:92-149](../../../packages/services/src/audit/emit.ts#L92-L149), [create.ts:53-59](../../../packages/services/src/monitor/create.ts#L53-L59).
Expected findings — Blocking: breaks the fail-closed invariant (mutation can commit with audit lost) *and* writes outside the tx (orphan audit rows for rolled-back mutations — both directions of inconsistency). Important: the perf premise is unmeasured — one extra insert inside an already-open tx. Optional: if audit volume ever matters, the correct shape is an outbox in-tx + async shipper, keeping atomicity.
This kata = interview gold: "what invariant does this PR delete?"

## Kata 7: "Unify quota errors to 403"

Author intent: consistency — all quota rejections return FORBIDDEN.
Fake diff summary: changes `LIMIT_EXCEEDED → TOO_MANY_REQUESTS` mapping to FORBIDDEN in service-adapter; changes v1's 402 to 403.
Files this resembles: [service-adapter.ts:57-65](../../../packages/api/src/service-adapter.ts#L57-L65), [post.ts:59-64](../../../apps/server/src/routes/v1/monitors/post.ts#L59-L64).
Expected findings — Blocking (v1 part): changing a public API's status code breaks clients switching on 402 — needs versioning or changelog + deprecation. Important: 403 is semantically wrong ("you can never" vs "you've hit a cap") — the current comment in the adapter documents the reasoning; consistency should converge on the *documented* choice (429), not the newest opinion. Optional: emit a `X-Quota-Limit` header.
Lesson: consistency PRs must state *which* standard wins and why, and treat public codes as contracts.

## Kata 8: "Test: assert monitors list renders 100 rows fast"

Author intent: add a perf regression test to the dashboard.
Fake diff summary: jest test rendering the monitors table with 100 fake rows, asserting render < 50ms via `performance.now()`.
Files this resembles: [client.tsx](../../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L43-L97).
Expected findings — Blocking: wall-clock thresholds in CI = guaranteed flakes (shared runners); Important: this repo has no jest setup — dashboard tests would need infra decisions first (its test culture is bun:test on services/server); measure render *counts* (React profiler) not ms if you must. Optional: the real perf risk in that component is the unmemoized derived arrays — a targeted approach is a profiling session, documented, not a CI test.
Lesson: reviews evaluate *test design* too — a flaky test is negative value.

---

Meta-drill: across all eight, tally which findings you caught by **reading the diff** vs by **knowing the repo's invariants** (audit atomicity, quorum, per-request client, error contracts). That ratio is your review maturity: seniors catch most issues from invariant knowledge, which is why this curriculum front-loads them.
