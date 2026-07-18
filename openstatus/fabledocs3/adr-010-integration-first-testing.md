# ADR-010: Integration-first testing on a shared seeded DB with rollback isolation

**Status:** Accepted for the service/API layers. Known deliberate-by-neglect gap: `apps/workflows` (the highest-risk code) has no tests; supersession there requires a clock/collaborator injection refactor first.

## Context

The failures that kill a multi-tenant product — cross-tenant leaks, quota bypasses, missing audit rows, scope bypasses — live in the *interaction* between service code, real SQL, and the schema. Mocked-ORM unit tests pass while all three burn. But real-database tests need an isolation strategy, and the libSQL HTTP driver supports no savepoints (no nested transactions).

## Decision

Test the invariants **at the service layer against a real seeded database** (bun:test; CI runs a sqld container). Isolation is **shared DB + rollback transactions**: each test body runs inside `withTestTransaction`, which throws a sentinel at the end and rolls everything back. Two constraints are documented in the helper itself: no nesting, and every read/write inside must use the `tx` handle. Where rollback can't be used, hygiene helpers compensate (`cleanQuotaGatedTables`, per-workspace `clearAuditLog`, `TEST_PREFIX` naming). Each layer owns distinct assertions — services own invariants (tenancy, quota, audit via `expectAuditRow`, scope via a standard `'rejects read-only actor'` case); route tests own contract shape and error mapping and do *not* re-test quota math; Go tests own probe semantics. `turbo test` runs `cache: false` because the tests aren't hermetic. Fixtures are seeded workspaces at stable ids with a deliberate plan spread (team workspace for happy paths, free workspace so quota-rejection paths exist naturally).

## Evidence

- `packages/services/test/helpers.ts:37-50` — `withTestTransaction` with its two in-code constraints; L52-88 — `cleanQuotaGatedTables` (its comments record the incident that motivated it: leftover committed rows made tight-quota tests "break randomly"); L206-211 — audit assertions sort `id DESC` because same-millisecond timestamps tie (flake-prevention as a one-liner).
- `packages/services/src/__tests__/with-test-transaction.test.ts` — the test infrastructure is itself tested.
- `.github/workflows/test.yml:15-27, 51-57` — sqld container; migrations applied before tests; `packages/db/src/seed.mts:26-40` — stable fixture ids.
- The gap: no test files under `apps/workflows/src/` (cron, `/updateStatus`, alerting); the collaborators (Cloud Tasks, Tinybird, wall-clock — e.g. non-injectable date math at `apps/workflows/src/cron/monitor.ts:393-407`) are currently un-mockable.

## Alternatives considered

1. **Mock the ORM, unit-test the verbs.** Rejected: the highest-value assertions (does this query actually filter by workspace? does the audit row actually commit?) are precisely what mocks fake.
2. **Per-test databases** (file-per-test SQLite or truncate-between-tests). Rejected: setup cost per test, awkward with the HTTP driver, and truncation destroys the seed. Rollback is fast and real.
3. **Savepoint-based nesting** (the textbook Rails/Django pattern). Unavailable: no savepoints over libsql HTTP — which is why the "no nesting" constraint exists rather than being solved.
4. **Cache test results in Turbo.** Rejected: non-hermetic tests + cached greens = lying CI. Honest-slow over fast-lying.

## Consequences

**Positive:** the product's core invariants are pinned by tests that exercise real SQL; every entity suite follows the same five-case template (happy path, validation, cross-tenant, quota, read-only scope) plus audit assertion, so coverage gaps are visible by inspection; the infrastructure being itself tested means isolation bugs surface as helper failures, not mystery flakes.

**Negative:** isolation is *conventional* — a test that writes with the default `db` instead of `tx` passes locally while committing rows that poison other suites' runs (the exact failure that motivated `cleanQuotaGatedTables`); CI time grows linearly (no caching); and the inverted risk profile stands: the best-tested code is CRUD, while the untested alerting pipeline is where the product actually gets hurt. The named precondition for fixing it: inject a clock and task/telemetry clients into workflows, then test the `/updateStatus` state machine with concurrent-report cases (the ADR-009 race).
