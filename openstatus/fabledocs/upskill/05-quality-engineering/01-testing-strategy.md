# Testing Strategy

## What actually exists here

| Layer | Runner | Where | Character |
| --- | --- | --- | --- |
| Service integration tests | bun:test | `packages/services/src/<entity>/__tests__/` | real seeded DB, per-test rollback transactions |
| Router tests | bun:test | `packages/api/src/router/*.test.ts` (e.g. [monitor.test.ts](../../../packages/api/src/router/monitor.test.ts)) | exercise tRPC procedures with inner-context injection (the test escape hatch at [trpc.ts:161-169](../../../packages/api/src/trpc.ts#L161-L169) exists for these) |
| REST route tests | bun:test | co-located `*.test.ts` beside every v1 route ([monitors/](../../../apps/server/src/routes/v1/monitors)) | HTTP-level via Hono |
| Go unit tests | go test | `apps/checker/**/*_test.go`, CI `go-tests.yml` | probe + assertion logic |
| Test-infra tests | bun:test | [with-test-transaction.test.ts](../../../packages/services/src/__tests__/with-test-transaction.test.ts) | the helpers are themselves tested |
| **Missing** | — | workflows (`cron/`, `checker/`) has no test files found; dashboard has no component test setup found | the highest-risk pipeline is the least tested — hold that thought for [06-contribution tickets](../06-contribution-practice/02-mid-level-feature-tickets.md) |

Environment: one seeded libSQL instance (CI: sqld container, [test.yml:15-27](../../../.github/workflows/test.yml#L15-L27)); seeds define fixture workspaces (team id 1 — [seed.mts:26-40](../../../packages/db/src/seed.mts#L26-L40); referenced as `SEEDED_WORKSPACE_TEAM_ID` in [monitor.test.ts:11-14](../../../packages/services/src/monitor/__tests__/monitor.test.ts#L11-L14)).

## What belongs at each layer

- **Service tests** own the invariants: tenancy rejection, quota enforcement, audit emission, scope rejection. They're integration tests by design — mocking Drizzle would test nothing real.
- **Route tests** own the contract: status codes, response shapes, auth failures. They should *not* re-test quota math.
- **Go tests** own probe semantics: assertion evaluation, redirect policy, timeout behavior — pure and fast.
- **What not to test:** Drizzle itself, zod itself, UI snapshot noise; and don't duplicate service-invariant tests at the route layer (drift cost, no signal).

## Isolation strategy (the interesting part)

The repo chose **shared DB + rollback transactions** over per-test DBs:
- `withTestTransaction` wraps the body, throws a sentinel, rolls everything back ([helpers.ts:37-50](../../../packages/services/test/helpers.ts#L37-L50)). Constraints documented in-code: no nesting (no savepoints over libsql HTTP), and all reads/writes inside must use `tx`.
- Where rollback isn't used, hygiene helpers compensate: `cleanQuotaGatedTables` exists because leftover rows made tight-quota tests "break randomly" ([helpers.ts:52-88](../../../packages/services/test/helpers.ts#L52-L88)); `clearAuditLog` per workspace (L90–100); unique `TEST_PREFIX` naming ([monitor.test.ts:31](../../../packages/services/src/monitor/__tests__/monitor.test.ts#L31)).
- Tradeoff to articulate: fast + real vs *ambient coupling* — a test that forgets `db: tx` writes committed rows and poisons someone else's run (that exact bug class is [debugging scenario 5](03-systematic-debugging.md)).

## Fixtures, time, randomness, flakes

- Fixtures: seeded workspaces by *stable id* + helper factories for contexts ([makeUserCtx/makeApiKeyCtx](../../../packages/services/test/helpers.ts#L121-L151)). Note the deliberate plan spread: team workspace for happy paths, free workspace for quota-rejection paths.
- Time: schema defaults use DB-side `strftime` ([monitor.ts:59-64](../../../packages/db/src/schema/monitors/monitor.ts#L59-L64)); business date math (`getScheduledTime`, [cron/monitor.ts:393-407](../../../apps/workflows/src/cron/monitor.ts#L393-L407)) is untested and non-injectable — a seam worth refactoring (clock parameter) before testing.
- Ordering: audit assertions sort by `id DESC` because same-millisecond rows tie on timestamps ([helpers.ts:206-211](../../../packages/services/test/helpers.ts#L206-L211)) — a tiny, quotable flake-prevention decision.
- `turbo test` is `cache: false` — correct, because tests aren't hermetic; the cost is linear CI time.

## Interview angle

- "Unit vs integration — where's the value?" → this repo's answer: integration-at-the-service-layer pins invariants; contrast with mock-heavy suites that pass while prod burns.
- "How do you keep DB tests fast and isolated?" → rollback transactions + their two documented constraints.
- "A test fails only in CI — go." → [scenario 5](03-systematic-debugging.md).

## Drill

Inventory one entity's suite (`packages/services/src/page/__tests__/` or similar): list which of the five standard cases exist — happy path, validation failure, cross-tenant rejection, quota rejection, read-only-scope rejection, audit assertion. Write the missing ones' `test("…")` names. Basic: inventory. Solid: correct gap list. Strong: you can defend which gaps are *worth closing* vs redundant with another layer.
