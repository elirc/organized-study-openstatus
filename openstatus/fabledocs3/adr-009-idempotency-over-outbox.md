# ADR-009: Pipeline reliability via layered idempotency — no transactions, no outbox

**Status:** Accepted, **under strain**. The two sharpest known risks in the codebase (non-transactional status flip; trigger-before-send at-most-once alerting) are direct consequences; both have identified supersession paths.

## Context

The alerting pipeline (probe result → regional evidence → quorum verdict → incident → notifications) runs at firehose volume across process and vendor boundaries (Cloud Tasks, notification providers). Full transactionality is impossible across those boundaries and was distrusted even within the database (HTTP-driver isolation, ADR-005). Retries exist at every layer — and retries without idempotency multiply effects.

## Decision

Consistency in the pipeline rests on **deterministic identity at three layers** instead of transactions: (1) probe enqueue — Cloud Task names are derived deterministically, so double-enqueue is a no-op; (2) notification send — a `notification_trigger` row is inserted (unique) **before** the provider call, so retries dedupe; (3) lifecycle email — Redis keys with TTL. Status transitions separate **evidence from verdict**: per-region `monitor_status` rows accumulate evidence; global `monitor.status` flips only on quorum — structural anti-flapping. Recovery is **compensation, not rollback**: incidents auto-resolve with `autoResolved: true`. `/updateStatus` performs its upsert → read → count → conditional update → incident insert as separate statements with **no transaction**.

## Evidence

- `apps/workflows/src/cron/checker.ts:245-356` — deterministic task names; Effect retry on enqueue (L204-208).
- `apps/workflows/src/checker/alerting.ts:131-176` — trigger row inserted before the send; retry on sends (L163-168).
- `apps/workflows/src/checker/index.ts:122-348` — the untransacted state machine; quorum check at L216; auto-resolve at L51-84.
- `apps/checker/handlers/checker.go:276` — Go-side probe retry with backoff.

## Alternatives considered

1. **Transactional outbox** (state change + send-intent in one transaction; separate dispatcher delivers, provider-side dedupe keys absorb duplicates). Rejected at the time as "more moving parts." This is the decision most worth revisiting: the chosen ordering makes alerting **at-most-once** — a crash after the trigger insert but before the provider call means the retry hits its own dedupe row and the alert is *suppressed forever*. For a paging product, at-least-once with dedupe is the defensible default; the customer would rather get two pages than zero.
2. **Send-then-record.** Rejected: crash-after-send duplicates pages. A reasonable fear — but the failure mode chosen instead (silent suppression) is strictly worse for this product. Supersession path: send status on the trigger row + a retry sweep, or a real outbox.
3. **Wrap the status flip in a transaction / serialize per-monitor.** Not adopted: the quorum flip is a read-check-write across awaits, so two regions reporting concurrently can both read pre-flip state → duplicate or missed notifications (risk R1). Fix candidates: one transaction around flip+incident+trigger, or queue-keying `/updateStatus` by monitorId.

## Consequences

**Positive:** every layer tolerates retries and double-fires by construction; anti-flapping is structural rather than a tuned debounce; the pipeline needs no broker of its own; compensation semantics (`autoResolved`) preserve honest history.

**Negative:** at-most-once alerting (above) — the sharpest known flaw; the quorum race under concurrent regional reports; idempotency-key granularity is a silent correctness parameter (too-coarse suppresses real events, too-fine duplicates); and observability can't distinguish "suppressed by dedupe" from "no channels configured" (see `../fabledocs2/06-observability.md`) — the schema records that a trigger existed, not that a provider accepted the page.
