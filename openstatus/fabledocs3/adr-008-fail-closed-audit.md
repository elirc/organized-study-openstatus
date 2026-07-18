# ADR-008: Fail-closed, in-transaction audit log

**Status:** Accepted for user-facing mutations. Known asymmetry: pipeline and lifecycle mutations bypass it — system-actor audit rows are the standing follow-up decision.

## Context

A multi-tenant infrastructure product whose customers themselves face compliance questions ("who changed this monitor and when?") needs an audit trail that can be trusted *negatively* — the absence of a row must mean the action didn't happen, not that logging hiccuped. Most audit implementations are best-effort (emit after commit, swallow failures), which produces logs with holes exactly when things are going wrong.

## Decision

Every service-layer mutation emits an audit row **inside the same transaction** as the mutation, via `emitAudit(tx, ctx, entry)`. The emission is **fail-closed**: if the audit insert fails, the business mutation rolls back. Conventions ride along: action names follow `{entity}.{verb}` in a discriminated union; updates pass `before`/`after` snapshots and `changed_fields` is auto-diffed (no-op updates skipped); secrets are stripped from snapshots before emitting; the actor comes from the Actor union (ADR-007), so attribution covers users, keys, and machines uniformly.

## Evidence

- `packages/services/src/audit/emit.ts:92-149` — the fail-closed emit inside the caller's transaction; auto-diffing; L14-24's hand-rolled `deepEqual` (Edge-safety, ADR-002).
- `packages/db/src/schema/audit_logs/validation.ts` — the `{entity}.{verb}` discriminated union.
- `packages/services/src/integration/remove.ts` — the secret-stripping exemplar.
- `packages/services/test/helpers.ts` — `expectAuditRow(...)`: every entity's test suite asserts the audit side-effect, making the invariant test-pinned (ADR-010).
- The asymmetry: `apps/workflows/src/checker/index.ts:227-231` (status flips) and `apps/workflows/src/cron/monitor.ts:292-303` (free-plan auto-pause) mutate domain state with **no audit row** — they bypass the services layer.

## Alternatives considered

1. **Best-effort audit** (emit after commit, log failures). Rejected: an audit log with holes fails at its one job; "we couldn't record it, so it didn't happen" is the defensible posture for a product customers lean on for their own compliance.
2. **CDC / DB-level change capture** (triggers, WAL streaming). Rejected: SQLite/libSQL tooling is weak here (ADR-005), and row-level capture loses the *semantic* action name and actor attribution that `{entity}.{verb}` + Actor union preserve.
3. **Async audit via queue/outbox.** Rejected for this use: durability would depend on a second system, reintroducing the hole problem; in-transaction is the strongest guarantee available and the write volume is low enough to afford it.

## Consequences

**Positive:** the log is negatively trustworthy; attribution is uniform across seven actor kinds; auto-diffing gives support and customers field-level answers; the test helper makes forgetting the audit a failing test rather than a review catch.

**Negative:** availability coupling — an audit-table problem blocks all user writes (accepted deliberately; that outage is preferable to silent gaps). The invariant's precision cost: it holds only for *service-layer* mutations, so a support engineer investigating a pipeline-paused monitor finds nothing in `audit_log` — the actor union already includes `system`, making the fix cheap and the omission purely a prioritization decision. Snapshot hygiene (secret-stripping) is convention, not enforcement — a new entity with a credential column depends on the author remembering the pattern.
