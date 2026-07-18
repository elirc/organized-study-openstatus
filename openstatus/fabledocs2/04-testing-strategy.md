# Part 4 — Testing strategy

## What it is and why it exists

The pyramid here is deliberately integration-heavy:

| Layer | Runner | Where | What it owns |
| --- | --- | --- | --- |
| Service integration tests | bun:test | `packages/services/src/<entity>/__tests__/` | **the invariants**: tenancy rejection, quota, audit emission, scope rejection — against a real seeded DB |
| Router tests | bun:test | `packages/api/src/router/*.test.ts` | the contract: shapes, auth failures, error mapping |
| REST route tests | bun:test | co-located beside every v1 route (`apps/server/src/routes/v1/monitors/`) | HTTP-level status codes and payloads via Hono |
| Go unit tests | go test | `apps/checker/**/*_test.go` | probe semantics: assertions, redirects, timeouts — pure and fast |
| Test-infra tests | bun:test | `packages/services/src/__tests__/with-test-transaction.test.ts` | yes, the helpers are themselves tested |

Environment: one seeded libSQL instance (CI runs a sqld container, `.github/workflows/test.yml:15-27`), with fixture workspaces at stable ids (`packages/db/src/seed.mts:26-40`). The seed is deliberate about plan spread: a team workspace for happy paths, a free workspace so quota-rejection tests exist naturally.

Why integration-heavy: the things that kill a multi-tenant product — cross-tenant leaks, quota bypasses, missing audit rows — live in the interaction between service code, SQL, and the schema. A mocked-Drizzle unit test passes while all three burn. So service tests hit the real database, and we accept the cost.

## The hard decisions

### 1. Shared database + rollback transactions, not per-test databases

Each test body runs inside `withTestTransaction`, which throws a sentinel at the end and rolls everything back (`packages/services/test/helpers.ts:37-50`). **Rejected alternatives:** per-test SQLite files (slow setup, and the HTTP driver made it awkward), full truncate-between-tests (slow and destroys the seed), and mocking the DB (tests nothing real, see above). The rollback approach is fast and real, with two constraints documented in the helper itself: **no nesting** (no savepoints over libsql HTTP) and **everything inside must use `tx`**, not the default client.

**The tradeoff that still bothers me:** that second constraint is *ambient coupling* — nothing in the type system forces `db: tx`, and the failure mode is nasty (see the war story). Where rollback isn't usable we compensate with hygiene: `cleanQuotaGatedTables` (`helpers.ts:52-88`), per-workspace `clearAuditLog` (L90-100), and `TEST_PREFIX` naming so leaked rows are at least identifiable.

### 2. Pin invariants once, at the service layer — don't re-test them per transport

Every entity suite is expected to cover the same five cases: happy path, validation failure, cross-tenant rejection, quota rejection, read-only-scope rejection — plus an audit assertion via `expectAuditRow` (`helpers.ts`). Route tests do **not** re-test quota math; they test that the route translates errors and shapes correctly. **Rejected:** the "test everything everywhere" instinct, which produces suites that drift into disagreement and triple the cost of every behavior change with zero added signal. One small decision I'm proud of: audit assertions sort by `id DESC` because same-millisecond rows tie on timestamps (`helpers.ts:206-211`) — that one line killed a whole class of flakes.

### 3. `turbo test` runs with `cache: false`

Turborepo would love to cache test results. We disabled it, on purpose: the tests aren't hermetic (shared DB, seed state), so a cache hit can report green for code whose environment changed. **Cost:** linear CI time forever. I'll take honest-slow over fast-lying.

### The hole I'm not proud of

`apps/workflows` — the cron fan-out and the `/updateStatus` state machine, the highest-risk code in the product — **has no test files**. Neither does the dashboard (no component test setup). The most-tested code (service CRUD) is the code least likely to page anyone; the least-tested code is the alerting pipeline. Classic. The blocker is real (the pipeline's collaborators are Cloud Tasks, Tinybird, and time — all currently non-injectable; `getScheduledTime` at `apps/workflows/src/cron/monitor.ts:393-407` mixes ms/seconds and can't even take a fake clock), but "the blocker is real" has been the excuse for too long. If you want to make a name here, refactor a clock parameter into that date math and write the first workflows test. I will approve that PR the hour it opens.

## War story

**Caused: the test that poisoned other people's runs.** A contributor wrote a service test that built its entities with the default `db` client instead of the `tx` handle. Their test passed. The rollback rolled back nothing, committed rows accumulated in the shared instance, and *other* tests — quota tests, which count rows — started failing "randomly," but only when run after that suite, and at first only in CI. Two engineers lost most of a day bisecting test *ordering* instead of test *code*. That incident is why `cleanQuotaGatedTables` exists (the helper's own comments tell this story, `helpers.ts:52-88`) and why "did every write inside the transaction use `tx`?" is a standing review-checklist item. It's the sharpest lesson this suite has to teach: **isolation by convention fails silently, and it fails in someone else's test.**

## Before you move on

1. Why would mocking Drizzle in service tests be worse than useless here? Name two specific invariants that would go unverified.
2. Your new test passes alone but a quota test fails when the suite runs together. What's your first hypothesis, and which helper confirms it?
3. Which five standard cases should `packages/services/src/page/__tests__/` contain? Go inventory the real suite and list the gaps — then argue which gaps are worth closing vs redundant with the route layer.
