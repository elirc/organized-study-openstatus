# Capstone: at-least-once alerting

The single most valuable thing you can build in this repo. ADR-009 names the flaw (trigger row inserted before the provider send ⇒ a crash between them suppresses the alert forever), fabledocs2 Part 5 narrates the incident, and fire drill 1 makes you diagnose it. This capstone makes you **fix** it — the full arc: design doc, migration, dispatcher, fault-injection proof. Done well, this is a portfolio piece and the backbone of your "tell me about a hard reliability problem" story.

**Prerequisites:** fire drill 1 completed; ADR-009 read; you can run the service test suite (`turso dev`, migrate, seed, `pnpm test` — see repo CLAUDE.md).

**Target semantics:** at-least-once delivery with dedupe — a down customer may rarely get two pages, but never zero.

---

## Milestone 1 — Design doc (half day)

Write a 1-2 page RFC before touching code. Required sections:

1. **Current behavior**, evidenced: the exact write/send ordering in `apps/workflows/src/checker/alerting.ts:131-176`, the crash-consequence table (crash before insert / after insert before send / after send), and the retry paths that make it so (Effect retry at L163-168; Cloud Tasks redelivery).
2. **Two candidate designs**, honestly compared:
   - **A. Send-status on the trigger row.** Add `status` (`pending | sent | failed`) + `attempts` + `lastAttemptAt` to `notification_trigger`. Insert as `pending` in the same write as today; mark `sent` after provider success; a sweep job retries stale `pending`/`failed` rows.
   - **B. Full transactional outbox.** New `notification_outbox` table written *in the same transaction* as the incident insert; a dispatcher loop claims rows, sends, marks done.
3. **Your pick and why.** (A is the pragmatic choice here: it reuses the existing dedupe identity and avoids a new dispatcher deployable — but you must argue it, not assert it. Note that B is the only option that also fixes the incident-insert-then-crash-before-trigger-insert gap; quantify whether that gap matters.)
4. **New failure mode accepted:** duplicate sends. Where does dedupe move? (Provider-side idempotency keys where supported — PagerDuty `dedup_key`, email Message-ID — and attempt-count caps where not.)
5. **Rollout:** additive migration first (ADR-005's litany), flag-gated dual behavior, then flip.

**Gate:** a reviewer (or you, a day later, cold) can implement from the doc without asking questions. If the doc doesn't state what happens when the *sweep itself* crashes mid-batch, it's not done.

## Milestone 2 — Schema migration (half day)

Additive migration for your chosen design: new columns (or table) nullable/defaulted, `drizzle-kit generate`, and — the part everyone skips — update the sibling zod `validation.ts` in the same change. Review the generated SQL for SQLite table-rebuild surprises. Write the backfill statement for existing trigger rows (`sent`, presumably — argue why that's the safe default for history).

**Gate:** migration applies to a seeded DB; existing service tests still green; the zod schema round-trips a row with and without the new fields.

## Milestone 3 — The write path (1 day)

Rework `alerting.ts`: insert `pending` (keeping the unique dedupe identity exactly as-is — changing key granularity is out of scope and dangerous, see ADR-009's consequences), send, mark `sent` on success / `failed`+increment on error. Keep the wide-event enrichment: the send outcome must land on the request's canonical event, because "suppressed vs no-channels" indistinguishability (drill 1) is one of the things you're fixing.

**Gate:** happy path green; a provider error leaves a queryable `failed` row instead of a swallowed exception; no change to dedupe behavior on clean retries (prove with a test that re-fires the same transition and asserts exactly one row, still).

## Milestone 4 — The sweep (1 day)

A new workflows endpoint (CRON_SECRET-authed, same pattern as its siblings in `apps/workflows/src/index.ts`) that claims stale `pending`/`failed` rows (older than N minutes, attempts < cap), retries the send, updates status. Decisions you must make and defend in code review comments to yourself: claim mechanism (an `UPDATE … WHERE status='pending' AND lastAttemptAt < ?` marking a claim time — what happens with two concurrent sweeps?), attempt cap and what happens at the cap (dead-letter status + wide event), and pacing (token bucket, mirroring the email pattern in `cron/monitor.ts`).

**Gate:** kill-between-steps works end to end — see Milestone 5.

## Milestone 5 — Fault-injection proof (1 day)

The test that makes it real. This requires the workflows-testability groundwork from [lab 04](04-labs-pipeline-and-go.md) (injectable provider client + clock) — do that lab first if you haven't.

Write three tests:

1. **Crash after insert, before send** (provider client throws process-death simulation / send never called): assert the row is `pending`, then run the sweep with a fake clock advanced past the threshold, assert the provider was called and the row is `sent`. *This is the test that fails on `main` and passes on your branch — frame it exactly that way in the PR description.*
2. **Crash after send, before mark:** sweep retries, provider called twice, assert the idempotency key passed to the provider is identical both times.
3. **Poison alert:** provider fails permanently; assert the attempt cap dead-letters it with a queryable status and an emitted wide event, not an infinite retry loop.

**Gate / definition of done:** all three green; drill 1's RCA question ("can you prove which cause it was?") now has answer *yes, by querying trigger status*; and you've written the 90-second spoken version of this project — situation, the at-most-once insight, the chosen design, the duplicate-sends tradeoff you accepted, the failing-on-main test. That story slots directly into `../fabledocs3/interview-translation.md` ADR-009's follow-up.

## Stretch goals

- Implement design B's outbox for the incident-insert gap and measure the added complexity honestly — a paragraph comparing what A didn't cover.
- Add the "suppressed vs no-channels" disambiguation to the wide event schema and write the Axiom-style query that separates them (closes drill 1's observability gap and fabledocs2 Part 6's blind-spot table row).
