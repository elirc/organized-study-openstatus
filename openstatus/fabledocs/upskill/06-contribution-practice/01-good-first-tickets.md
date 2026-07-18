# Good First Tickets (Junior)

Sixteen tickets spread across the repo. Every one is real (anchored to code as it exists today), small blast radius, and follows an existing pattern. **Before starting any ticket:** re-read the anchors — line numbers were verified 2026-07-11 and drift.

Template fields are abbreviated; expand your own implementation plan before coding. All test commands are __inferred__ (see [command cheatsheet](../09-reference/command-cheatsheet.md)).

---

## Ticket J1: Add audit-row test for `updateMonitorRetry`
Difficulty: Easy — 1–2h. Skills: bun:test, test transactions, audit invariant.
Story: As a maintainer, I want every monitor verb's audit emission pinned by a test so regressions are caught.
Why good: pure test addition; copies an existing shape.
Acceptance criteria: new test in [monitor.test.ts](../../../packages/services/src/monitor/__tests__/monitor.test.ts) asserting `monitor.update` audit row after `updateMonitorRetry` ([update.ts:73-100](../../../packages/services/src/monitor/update.ts#L73-L100)); uses `withTestTransaction` + `expectAuditRow` ([helpers.ts:37-50, 240-264](../../../packages/services/test/helpers.ts#L37-L50)).
Read first: the happy-path test at [monitor.test.ts:79-108](../../../packages/services/src/monitor/__tests__/monitor.test.ts#L79-L108).
What could go wrong: forgetting `db: tx` in ctx/assert → reads miss in-tx writes.
Review questions: does the test assert `changedFields` includes `retry`?
Interview story potential: "I extended coverage of a fail-closed audit invariant" (STAR-S2).

## Ticket J2: `'rejects read-only actor'` coverage sweep
Difficulty: Easy — 2–3h. Skills: RBAC testing, grep-driven auditing.
Story: As a maintainer, I want every write verb to have the standard read-only-key rejection test (convention documented in [CLAUDE.md](../../../CLAUDE.md)).
Why good: convention already exists ([makeApiKeyCtx](../../../packages/services/test/helpers.ts#L132-L151) with `scopes: ["read"]`); you're closing gaps.
Plan: inventory write verbs (`grep -l requireScope packages/services/src/**/`), inventory tests containing `rejects read-only`, diff, fill.
What could go wrong: verbs where `requireScope` fires before DB lookup accept fake ids (that's by design — CLAUDE.md notes it).
Interview story potential: "I systematically verified an authz invariant across ~20 modules."

## Ticket J3: Structured logging for unknown assertion types (Go)
Difficulty: Easy — 1h. Skills: Go basics, logging hygiene.
Story: As an operator, I want unknown assertion types visible in log aggregation, not stdout noise.
Anchor: [checker.go:366-368](../../../apps/checker/handlers/checker.go#L366-L368) — `fmt.Println("unknown assertion type: ", …)` next to a TODO.
Plan: replace with `log.Ctx(ctx).Error().…` matching the zerolog style used at [checker.go:71](../../../apps/checker/handlers/checker.go#L71); include monitor id + type.
Checks: `go test ./...` in `apps/checker` (CI runs go-tests.yml).
Interview story: "smallest possible observability fix, correctly scoped."

## Ticket J4: Replace the monitors-list `return null` flash with the existing skeleton
Difficulty: Easy–Medium — 2–4h. Skills: React loading states, TanStack Query.
Anchor: [client.tsx:92](../../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L92) — `if (!monitors) return null;` blanks the page during load, while `MetricCardSkeleton` is already imported (L17).
Acceptance: loading renders skeletons; no layout shift on data arrival; error state distinct from loading (`useQuery` exposes both).
What could go wrong: `monitors` is also `undefined` on error — don't render skeletons forever; handle `isError`.
Interview story: "distinguishing loading/empty/error states" — a classic frontend-round probe.

## Ticket J5: Include `requestId` in requireScope denial logs
Difficulty: Easy — 1h. Skills: logging, tracing correlation.
Anchor: [require-scope.ts:37-45](../../../packages/services/src/auth/require-scope.ts#L37-L45) — the warn includes key/workspace but not `ctx.requestId` (available on ServiceContext, [context.ts:25](../../../packages/services/src/context.ts#L25)).
Acceptance: denial log carries requestId when present; existing tests still pass (`bun test src/auth` in packages/services).
Rejection risk: maintainer may prefer structured args over string interpolation — match the file's existing style.
Interview story: "correlating authz denials with request traces."

## Ticket J6: De-duplicate the doubled REMINDER comments in trpc.ts
Difficulty: Easy — 30m. Skills: reading before deleting.
Anchor: [trpc.ts:206-212](../../../packages/api/src/trpc.ts#L206-L212) — the same two REMINDER lines appear twice around the result check.
Acceptance: one copy remains, placed above the code it describes; no behavior change.
Why a maintainer accepts: comment discipline is an explicit repo value ([CLAUDE.md](../../../CLAUDE.md) "Comment Discipline").
Interview story: minor, but pairs with J5 as "I read the conventions doc and applied it."

## Ticket J7: Add a `latency` field guard test for `/updateStatus` payload
Difficulty: Easy–Medium — 2h. Skills: zod, Hono testing.
Anchor: [checker/index.ts:20-28](../../../apps/workflows/src/checker/index.ts#L20-L28) `payloadSchema`; 422 branch at L96–99.
Story: pin the wire contract between Go checker and workflows with tests (there are none for this route — verify before writing).
Acceptance: tests for valid payload (200), missing region (422), wrong auth (401). Use Hono's `app.request()` test style (see existing server route tests, e.g. `apps/server/src/routes/v1/monitors/get.test.ts`, for the idiom).
What could go wrong: module imports env at load — may need env vars in test setup (look at how workflows env is validated in [env.ts](../../../apps/workflows/src/env.ts)).
Interview story: "I pinned an untested cross-service contract."

## Ticket J8: Document `pnpm dx` in CONTRIBUTING
Difficulty: Easy — 1h. Skills: docs empathy, task-graph reading.
Anchor: [turbo.json](../../../turbo.json) (`dx` → seed → migrate → env), [CONTRIBUTING.MD](../../../CONTRIBUTING.MD).
Acceptance: a short "what dx actually does" section with the dependency chain and when to re-run it.
Rejection risk: duplicating README; link instead of copying.

## Ticket J9: Surface the 429 quota error as an upgrade prompt toast
Difficulty: Medium — 3–5h. Skills: tRPC error handling client-side, UX.
Anchor: `LIMIT_EXCEEDED → TOO_MANY_REQUESTS` at [service-adapter.ts:57-65](../../../packages/api/src/service-adapter.ts#L57-L65); monitor create mutation call sites under `apps/dashboard/src/components/forms/monitor/` (locate the create form; not read in these docs — verify).
Acceptance: hitting the monitor quota shows a plan-upgrade message, not a generic failure.
What could go wrong: matching on message string instead of `error.data.code` — match the code.
Interview story: "error taxonomy paying off end to end" (great frontend-round material).

## Ticket J10: Add `externalName` to the monitors data-table (hidden column)
Difficulty: Easy–Medium — 2–4h. Skills: TanStack Table, schema reading.
Anchor: column exists in schema ([monitor.ts:31](../../../packages/db/src/schema/monitors/monitor.ts#L31)); columns file at `apps/dashboard/src/components/data-table/monitors/columns.tsx` (verify shape there).
Acceptance: column available via column-visibility menu, off by default.
Rejection risk: if `externalName` isn't in the tRPC list output schema, that's scope growth — check first; if missing, downgrade ticket to "expose field in list schema" and ask a maintainer.

## Ticket J11: Test `deepEqual` date/array edge cases
Difficulty: Easy — 1–2h. Skills: unit testing pure functions.
Anchor: [emit.ts:25-60](../../../packages/services/src/audit/emit.ts#L25-L60); existing tests in `packages/services/src/audit/__tests__/` (check what's covered before adding).
Acceptance: cases for Date-vs-string, nested array order sensitivity, `undefined`-vs-`null` (interacts with `diffTopLevel` absent-handling L77-87).
Interview story: "testing the pure core of an audit diff engine."

## Ticket J12: Deprecated-region log level
Difficulty: Easy — 1h. Skills: log-level judgment.
Anchor: [cron/checker.ts:166-171](../../../apps/workflows/src/cron/checker.ts#L166-L171) — deprecated regions log `logger.error` every tick for every affected monitor; the comment says removal is pending.
Story: as an operator I don't want recurring expected conditions at error level (alert fatigue).
Plan: downgrade to `warn` with a dedupe note, or count occurrences into the cron summary log.
Rejection risk: maintainers may *want* it loud to force cleanup — ask in the PR description (this is the lesson).

## Ticket J13: Extract the thrice-duplicated notification retry block
Difficulty: Medium — 3–5h. Skills: refactoring, Effect.
Anchor: [alerting.ts:144-239](../../../apps/workflows/src/checker/alerting.ts#L144-L239) — alert/recovery/degraded branches are copy-paste except the method name.
Acceptance: one helper `sendWithRetry(provider, method, args)`; behavior identical (retry counts, log fields).
What could go wrong: `providerToFunction[provider][method]` loses type narrowing — keep the union typed (`"sendAlert" | "sendRecovery" | "sendDegraded"`).
Interview story: strong refactor story — "I removed 3× duplication in the alerting hot path with zero behavior change, proven by …" (write the proof plan!).

## Ticket J14: Add `whoami` scopes to the response
Difficulty: Medium — 2–4h. Skills: OpenAPI route, contract addition.
Anchor: `apps/server/src/routes/v1/whoami/` (verify current response shape); key scopes available at [auth.ts:115-119](../../../apps/server/src/libs/middlewares/auth.ts#L115-L119).
Story: As an API user I want to see which scopes my key holds.
Acceptance: additive field only; OpenAPI schema updated; test added beside existing whoami tests.
Rejection risk: leaking internal scope names — but they're already user-facing concepts.

## Ticket J15: Fix the fast-path `if (!monitors)` in `getMonitorListMetrics` inputs (empty vs loading)
Difficulty: Easy–Medium — 2h. Skills: distinguishing `undefined`/`[]`.
Anchor: [client.tsx:51-61, 92-97](../../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L51-L97) — `monitorsByType` computes on `monitors === undefined` as `[]`, disabling the metrics queries (`enabled: length > 0`), then L92 hides everything anyway. Untangle: loading → skeleton (J4), loaded-empty → empty state with CTA.
Pairs with J4; do them together as one PR.

## Ticket J16: Add JSON body size note to probe docs + constant
Difficulty: Easy — 1h. Skills: reading Go, doc precision.
Anchor: [checker.go:319-329](../../../apps/checker/handlers/checker.go#L319-L329) — `data=true` responses truncate body to 1000 bytes when > 1024 (note the 1000 vs 1024 asymmetry). Name the constant, document the behavior where the checker API is documented (apps/checker README or docs app; verify).
Interview story: "found an off-by-truncation quirk by reading, not running."

---

## How to pick

Do J1 or J2 first (test-only, zero risk), then one UI (J4/J15), one Go (J3), one cross-cutting (J13). That mix — test, frontend, second language, refactor — is deliberately shaped like a mid-level portfolio.

For every ticket, before opening the PR answer the five contribution-realism questions: useful why? blast radius why small? which existing pattern? how tested? what would make a maintainer reject it? Write the answers into the PR description ([template](../07-career-and-collaboration/02-writing-prs-and-rfcs.md)).
