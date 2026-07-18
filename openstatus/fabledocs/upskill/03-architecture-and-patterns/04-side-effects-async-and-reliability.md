# Side Effects, Async & Reliability

## The side-effect map

| Effect | Triggered from | Anchor | Durability of intent |
| --- | --- | --- | --- |
| Probe task enqueue (GCP) | cron tick | [cron/checker.ts:245-356](../../../apps/workflows/src/cron/checker.ts#L245-L356) | task name = idempotent |
| Tinybird ping event | Go checker per probe | [checker.go:255-257](../../../apps/checker/handlers/checker.go#L255-L257) | **none** — failure logged, row lost |
| Status-change task | Go checker on transition | [update.go:29-93](../../../apps/checker/checker/update.go#L29-L93) | queue-backed |
| monitor_status upsert, monitor.status flip, incident insert/resolve | /updateStatus | [checker/index.ts:122-348](../../../apps/workflows/src/checker/index.ts#L122-L348) | direct writes, no tx |
| Provider notifications (slack/pagerduty/sms/…) | alerting | [alerting.ts:144-239](../../../apps/workflows/src/checker/alerting.ts#L144-L239) | trigger row before send |
| Emails (lifecycle, verification) | workflows / subscriber flows | [cron/monitor.ts:425-444](../../../apps/workflows/src/cron/monitor.ts#L425-L444) | Redis key before send |
| Audit rows | every service mutation | [emit.ts:102-149](../../../packages/services/src/audit/emit.ts#L102-L149) | in-transaction (strongest) |
| Analytics events | tRPC `after()` | [trpc.ts:213-238](../../../packages/api/src/trpc.ts#L213-L238) | post-response, best-effort |
| Stripe | dashboard billing | `packages/api/src/router/stripe/` (not read) | — |

## Concepts, defined on this repo

**Idempotency** — same operation twice, one effect. Three implementations compared in [Pattern 10](05-pattern-catalog.md): deterministic task names (enqueue), unique-insert trigger rows (send), Redis keys with TTL (email). Note what each key *encodes* and its failure directions: too-coarse suppresses real events, too-fine duplicates.

**Retries** — three stacks: Go `backoff` around probes ([checker.go:276](../../../apps/checker/handlers/checker.go#L276)), Effect exponential on enqueue ([cron/checker.ts:204-208](../../../apps/workflows/src/cron/checker.ts#L204-L208)) and on sends ([alerting.ts:163-168](../../../apps/workflows/src/checker/alerting.ts#L163-L168)). Retries are only safe *because* of the idempotency layer beneath them — always say these two words together.

**Outbox (absent)** — the classic pattern writes intent to the DB in the same tx as the state change, and a separate dispatcher delivers. Here, the state change and the sends happen in the *same request*: if the process dies between the incident insert and the provider call, the alert is lost (the trigger row may or may not exist — read the order at [alerting.ts:131-176](../../../apps/workflows/src/checker/alerting.ts#L131-L176): trigger inserted **before** send, so a crash after insert = alert *suppressed forever*, because retry hits the dedupe). That's the sharpest reliability finding in this repo — verify it, then own it in interviews.

**Compensation** — incidents auto-resolve on recovery ([checker/index.ts:51-84](../../../apps/workflows/src/checker/index.ts#L51-L84), `autoResolved: true`) — a compensating write, not a rollback.

**Backpressure** — bounded concurrency (100) on enqueue; token-bucket on emails; GCP queue absorbs probe bursts. What has *no* backpressure: `/updateStatus` fan-in during a mass outage (every monitor × region transitioning at once) — the alerting queue helps, but the handler's per-request DB work is unbounded.

**Timeouts** — per-monitor probe timeout honored by a per-request Go client ([checker.go:76-79](../../../apps/checker/handlers/checker.go#L76-L79)); note there's no visible timeout on the notification provider calls (*investigate* per provider package).

**Failure visibility** — every pipeline stage logs structured wide events; failures also Sentry ([cron/checker.ts:232-242](../../../apps/workflows/src/cron/checker.ts#L232-L242)). But visibility ≠ recovery: a logged lost Tinybird row stays lost.

## Side effects in risky places (flagged)

1. **Trigger-then-send ordering** (above) — durable dedupe recorded before the effect happens. Swap risk: send-then-record duplicates on crash instead of suppressing. Pick your failure mode consciously; today's choice = at-most-once.
2. **Writes inside an auth middleware** — `lastUsedAt` debounced update during key validation ([auth.ts:184-190](../../../apps/server/src/libs/middlewares/auth.ts#L184-L190)). Adds a write to every (debounce-expired) read request.
3. **Email sends inside cron steps with date math** — `getScheduledTime` mixes ms/seconds conventions ([cron/monitor.ts:393-407](../../../apps/workflows/src/cron/monitor.ts#L393-L407)); off-by-unit here means emails at wrong times. Worth a unit test (ticket M-candidates).
4. **Fire-and-forget analytics in `after()`** — correct place (post-response), but errors there vanish silently; acceptable for analytics, never copy the pattern for domain effects.

## Drill

Draw the crash-consequence table for the alert path: mark an ✗ at each step (before trigger insert / after insert before send / after send) and write what the user experiences on restart. Then redesign with an outbox: which writes join the incident transaction, what does the dispatcher loop look like, and what new failure mode do you accept (duplicate sends → need provider-side dedupe keys)?
Self-grade — Basic: correct crash table. Solid: outbox sketch with dispatcher. Strong: articulates at-most-once vs at-least-once as a *product* decision ("would this customer rather get two pages or zero?").

## Interview angle

This file is the ammunition for "how do you make X reliable?" questions: [08/03 Q9](../08-interview-prep/03-api-and-data-modeling-questions.md) (webhooks/idempotency), [08/04 Step 4](../08-interview-prep/04-system-design-from-this-repo.md) (pipeline failure enumeration). The trigger-before-send analysis is a genuinely senior observation — rehearse saying it in under 60 seconds.
