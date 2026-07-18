# 01 — Testing Strategy

Tests exist to give you the courage to change code. That's the whole purpose. Any test that doesn't enable confident change is a tax.

## The pyramid you should actually build

Forget the canonical "70% unit, 20% integration, 10% e2e." Real-world pyramid:

1. **Type tests** (the compiler) — fastest, broadest, lowest cost. TypeScript is your first line.
2. **Unit tests** of pure logic — fast, deterministic. Use for parsers, formatters, derivations.
3. **Service tests** with a real DB — *slightly* slower, very high value. The bulk of your test investment for an app like OpenStatus.
4. **Router tests** with constructed contexts — verify wire shape, auth, error mapping.
5. **End-to-end** (Playwright/Cypress) — sparse, expensive, brittle. Run for critical user journeys only.

OpenStatus's investment matches: ~no e2e in the OSS repo, heavy service + router tests, type checking everywhere.

## What goes at each layer

### TypeScript (free tests you didn't write)

A discriminated union with `assertNever` catches missing cases at compile time. A strict tsconfig (`strict: true`, `noUncheckedIndexedAccess: true`) catches null/undefined misuse. These are tests; they run on every keystroke.

If you find yourself writing a test that asserts "the function returns string when given X," ask if a stricter type would do it for free.

### Unit tests

For pure logic — derivations, formatters, parsers, comparators. Example targets:

- `periodToSince(period)` ([`packages/services/src/status-report/list.ts:36-47`](../../packages/services/src/status-report/list.ts#L36-L47)): given a period string, returns a Date. Pure. Test all branches.
- Sort comparators in `statusPage.utils.ts`. Pure. Test edge cases (empty arrays, single elements, ties).
- Zod schemas with custom `.refine`s. Pure validation.

### Service tests with the real DB

The most important tests in this codebase. They exercise:

- DB writes and reads through the same path production uses.
- Transaction behavior.
- Audit emission.
- Cross-workspace rejection.
- Scope enforcement.

Pattern: `withTestTransaction` rolls back every test ([`packages/services/test/helpers.ts:37-50`](../../packages/services/test/helpers.ts#L37-L50)), so a seeded DB is reused without per-test cleanup.

### Router tests

Verify the boundary: wire shape acceptance, auth middleware, error code mapping, cross-workspace IDOR rejection. The status-report IDOR test ([`packages/api/src/router/statusReport.test.ts:77-100`](../../packages/api/src/router/statusReport.test.ts#L77-L100)) is the model.

```ts
const ctx = createInnerTRPCContext({
  req: undefined,
  session: { user: { id: "1" } },
  workspace: { id: 1 },
});
const caller = edgeRouter.createCaller(ctx);
await caller.statusReport.create({ pageId: otherWorkspacePageId, ... });
// expect TRPCError with code NOT_FOUND
```

Construct a context, create a caller, invoke a procedure, assert the result or error.

### E2E (Playwright)

OpenStatus's OSS surface doesn't ship e2e, but if it did, the highest-value tests would be:

- Signup → first monitor → first incident → public page shows it.
- Login → workspace switch → mutation respects new workspace.

Three e2e tests covering the golden paths beat fifty covering edge cases. Edge cases belong lower in the pyramid.

## What *not* to test

- **Implementation details.** "The function calls Drizzle three times" — fragile, changes when you optimize.
- **Framework behavior.** "useState updates state." Not your code.
- **Library code.** Don't test that Zod validates; test that *your* schema accepts/rejects what you want.
- **Trivial getters/setters.** No.
- **Snapshots of UI** that change every PR — they become noise.

If a test's failure doesn't tell you "this PR broke a real requirement," delete it.

## Test naming

Bad:
```
test("createStatusReport works")
test("test 1")
```

Good:
```
test("creates report + initial update + associations and emits audit")
test("rejects pageId from another workspace")
test("rejects when actor has read-only scope")
```

The name *is* the requirement. Read the failing test name; you should know what broke without opening the file.

## Test isolation in OpenStatus

The test stack (Bun's built-in test runner) plus `withTestTransaction` plus per-suite workspace scoping ([`packages/services/test/helpers.ts`](../../packages/services/test/helpers.ts)) plus seeded fixtures (`SEEDED_WORKSPACE_FREE_ID`, `SEEDED_WORKSPACE_TEAM_ID`) give you:

- A real DB shared by all tests.
- No persistent state changes per test (rollback).
- Per-suite cleanup for things rollback can't help (audit log, quota-gated tables).
- Predictable seeded data (workspaces 1 and 2).

This is the cheapest fast accurate test infrastructure you'll find. It works because every service threads `ctx.db` and the production code uses `withTransaction`. Skip either invariant and the test pattern breaks.

## Coverage — useful number, terrible target

100% coverage doesn't mean "well tested." It means "every line was executed during a test." A line can execute without any meaningful assertion.

Target instead:
- Every public service verb has at least one happy-path test and one rejection test (cross-workspace, scope mismatch, validation failure).
- Every audit-emitting mutation has a test that asserts the audit row.
- Every list/get endpoint has a test that asserts workspace scoping.

That's a *qualitative* target, not a number. Use coverage to find *holes* (this whole file has 0% — why?), not to chase 100%.

## Flaky tests are bugs

A test that fails intermittently is worse than no test — it teaches the team to ignore failures. When you see a flaky test:

1. Reproduce locally (run 10x).
2. Identify the source of non-determinism: time, network, ordering, shared state.
3. Fix the test or fix the code; don't `.skip()` it without a tracking issue.

If you must skip: comment with the issue link and a remove-by date. Otherwise it becomes folklore.

## Test data — fixtures vs builders

Two patterns:

**Fixtures:** seeded rows you reference by ID. Fast, predictable. OpenStatus uses `SEEDED_WORKSPACE_FREE_ID` and `SEEDED_WORKSPACE_TEAM_ID`.

**Builders:** factory functions that insert what each test needs. Independent, expressive, slower.

OpenStatus mixes them: seeded workspaces for plan/quota state, per-test inserts for pages/components/reports the test actually mutates. That's the right balance for this app.

## Mocking — minimal, deliberate

Mocking is necessary when:
- The dependency is external (an HTTP API, a queue) and you can't control it.
- The dependency is non-deterministic (random, time, network).

Mocking is *wrong* when:
- You're mocking your own code to make tests easier — that's a sign your code is hard to use.
- You're mocking the DB — tests now pass even when SQL changes.

OpenStatus mocks the notification dispatcher in service tests ([`packages/services/src/status-report/__tests__/status-report.test.ts:41-56`](../../packages/services/src/status-report/__tests__/status-report.test.ts#L41-L56)):

```ts
const subscriptionSpies = (globalThis as Record<string, unknown>)
  .__subscriptionSpies as { dispatchStatusReportUpdate: { mockClear, mock: { calls } } };
```

It does *not* mock Drizzle. The DB is real (libsql via the local Turso server). That's the right call.

## Time and randomness

If your code reads `Date.now()` or `Math.random()`, your tests can't be deterministic without injecting them. Patterns:

- Take a `clock` parameter: `function expire(now: Date) { ... }`.
- Or inject a function: `clock: () => Date`.
- Or use a library like `vitest`'s `vi.useFakeTimers()`.

OpenStatus's `periodToSince(period)` uses `Date.now()` directly. Hard to test with multiple clocks. A refactor would extract a `now()` helper. Not urgent, but a senior tightening the suite would do this.

## Pitfall checklist

1. **Tests that don't fail when the code is wrong.** Useless assertions, mock that fakes the bug.
2. **Mocking your own code.** Sign of bad design.
3. **Tests that depend on order.** `beforeEach` should reset state.
4. **Tests with hardcoded waits** (`await sleep(1000)`). Use `await waitFor(...)` or proper event waits.
5. **Untested error paths.** "It works" is half the test.
6. **Snapshot tests of churning output.** Noise, not signal.
7. **Tests that need network access.** Mock the boundary, not the world.
8. **Skipped tests with no issue link.** Folklore in waiting.
9. **`describe` blocks 500 lines deep.** Split.
10. **Tests for code that doesn't exist yet.** TDD is fine; commits with skipped placeholder tests are not.

## Drills

- Read [`packages/services/src/status-report/__tests__/status-report.test.ts`](../../packages/services/src/status-report/__tests__/status-report.test.ts). For each `test(...)`, name the production requirement it encodes.
- Find one untested service verb. Write a test using the existing helpers.
- Identify a test that you'd delete (low signal, high noise). Defend the deletion.
- Read [`packages/services/test/helpers.ts:37-50`](../../packages/services/test/helpers.ts#L37-L50). Explain in one paragraph why this pattern requires the production code to *not* open its own transactions.
- Sketch a test plan for a new feature ("auto-resolve reports after 24h of no updates"). What goes in unit, service, router, e2e?
