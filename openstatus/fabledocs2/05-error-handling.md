# Part 5 — Error handling & reliability

## What it is and why it exists

Error handling here is three distinct regimes, and confusing them is the classic new-hire mistake:

1. **Request-path errors** — `ServiceError` subclasses thrown from services, translated per-transport (`packages/api/src/service-adapter.ts:32-87`), `cause` preserved all the way to Sentry. Clean, boring, done.
2. **Consistency errors** — handled by transactions where they exist (user mutations) and by idempotency backstops where they don't (the pipeline).
3. **Delivery errors** — retries wrapped around side effects, safe only because of the idempotency layer beneath them. Say those two words together, always: *retries + idempotency*. Retries without idempotency is how you page a customer six times.

The idempotency layer is worth memorizing because it appears three times with three mechanisms: **deterministic task names** for probe enqueue (double-enqueue = no-op, `apps/workflows/src/cron/checker.ts:245-356`), **unique-insert trigger rows** for notifications (`apps/workflows/src/checker/alerting.ts:131-176`), and **Redis keys with TTL** for lifecycle emails (`apps/workflows/src/cron/monitor.ts:425-444`). Each key encodes an identity; too-coarse a key suppresses real events, too-fine duplicates them.

## The hard decisions

### 1. Fail-closed audit: a failed audit insert rolls back the business mutation

`emitAudit` runs inside the same transaction as the write it describes (`packages/services/src/audit/emit.ts:92-149`). If the audit insert fails, the user's mutation fails. **Rejected:** best-effort audit (log-and-continue). That's the popular choice and it's wrong for us: an audit log with holes is not an audit log, it's a vibe. For a product whose customers use audit trails for their own compliance, "we couldn't record it, so it didn't happen" is the defensible failure mode. **Cost I accepted:** availability coupling — an audit-table problem can block all writes. It has never happened, and if it does, I'd rather have that outage than silent gaps.

### 2. Trigger-before-send: we chose at-most-once alerting, and I regret it

Read the ordering at `alerting.ts:131-176` carefully: the notification-trigger row (the dedupe record) is inserted **before** the provider call. Now run the crash scenarios:

- Crash *before* the insert → retry re-runs everything → fine.
- Crash *after* send, had we recorded after → duplicate page → annoying.
- Crash *after* the insert, *before* the send → retry hits the dedupe and skips → **the alert is suppressed forever.** Nobody is paged. For a paging product.

**Rejected at the time:** send-then-record (risks duplicates) and a proper outbox (state change + intent written in one transaction, separate dispatcher delivers — "more moving parts"). I picked insert-first because duplicate pages felt like the embarrassing failure. That was the wrong instinct and I'll say it plainly: **ask the customer whether they'd rather get two pages or zero.** At-least-once with provider-side dedupe keys is the defensible default for alerting. The migration path is an outbox table or a status column on the trigger row plus a retry sweep; the test is kill-between-steps fault injection. This is the sharpest known flaw in the codebase — I'd rather you hear it from me than find it in an incident.

### 3. Compensation over rollback for cross-plane state

You can't roll back a Slack message or a Tinybird row, so recovery flows use **compensating writes**: incidents auto-resolve on recovery with `autoResolved: true` (`apps/workflows/src/checker/index.ts:51-84`) rather than pretending the incident never happened. **Rejected:** trying to make the pipeline transactional across SQL + Tinybird + providers — distributed transactions across third parties is a research project, not a feature. The flag matters: humans can distinguish "system decided it recovered" from "operator resolved it."

### Two smaller regimes to know before they surprise you

- **Fail-open vs fail-closed is chosen per surface, not globally.** The status page proxy fails *open* on a schema-invalid page row (public availability wins); scope checks fail *closed* (`packages/services/src/auth/require-scope.ts:25-48`). Both are correct *for their surface* — but the fail-open path has no counter, so we'd never know it's happening (Part 6).
- **404 vs 403 is deliberately inconsistent.** Cross-tenant fetches return NOT_FOUND to avoid leaking existence (`getMonitorInWorkspace`), but tag ownership validation returns FORBIDDEN (`packages/services/src/monitor/internal.ts:63-84`). The first is policy; the second predates the policy. Follow the 404 pattern for anything new.

## War story

**Caused (by construction): the alert that never came.** Trace it yourself, because this is the one you must believe with your own eyes: monitor goes down, quorum agrees, incident row inserted, trigger row inserted at `alerting.ts:131` — and the workflows process is OOM-killed before the PagerDuty call at L176. Fly restarts the process; the retry arrives; the dedupe check sees the trigger row and concludes *this alert was already handled*. The customer's site is down, our database says everything worked, and the only queryable truth — a trigger row with no corresponding provider success — is a distinction our schema doesn't record. The incident review writes itself, which is why Parts 5 and 6 of your onboarding both point at the same fix: put send status on the trigger row, sweep for stuck ones, alert on the sweep.

## Before you move on

1. For each of the three idempotency mechanisms, name the identity the key encodes and one failure that a too-coarse version of that key would cause.
2. Draw the crash-consequence table for the alert path (crash before trigger insert / after insert before send / after send): what does the user experience on process restart in each case?
3. Argue both sides in two sentences each: audit fail-closed but status-page fail-open. Why are opposite choices both correct?
