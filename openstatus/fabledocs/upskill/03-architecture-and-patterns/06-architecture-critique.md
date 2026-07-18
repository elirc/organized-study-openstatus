# Architecture Critique

The judgment file. Confirmed observations cite lines; everything else is labeled hypothesis. This doubles as system-design interview material — the walkthrough version is [08/04](../08-interview-prep/04-system-design-from-this-repo.md).

## Strongest design choices (steal these)

1. **The services layer with fail-closed audit.** One verb per file, uniform signature, tenancy + quota + audit as invariants enforced in one place ([create.ts](../../../packages/services/src/monitor/create.ts#L16-L63), [emit.ts:92-149](../../../packages/services/src/audit/emit.ts#L92-L149)). This is what "business logic layer" looks like when it's real, not aspirational.
2. **Two data planes.** OLTP state in libSQL, probe telemetry in Tinybird ([checker.go:255](../../../apps/checker/handlers/checker.go#L255), [client.ts](../../../packages/tinybird/src/client.ts#L15-L50)). Right-sized: the write-heavy plane never threatens the transactional plane.
3. **Evidence vs verdict separation + quorum.** Per-region `monitor_status` accumulates evidence; `monitor.status` flips on quorum ([checker/index.ts:216](../../../apps/workflows/src/checker/index.ts#L216)). Structurally correct anti-flapping.
4. **Idempotency by deterministic identity** at three layers ([Pattern 10](05-pattern-catalog.md)) — the pipeline tolerates retries and double-fires by construction.
5. **Wide-event observability with tail sampling** ([server index.ts:109-179](../../../apps/server/src/index.ts#L109-L179)) — one canonical line per request, errors/slow always kept.
6. **Contract-first REST** — zod `createRoute` schemas generating OpenAPI ([post.ts:15-42](../../../apps/server/src/routes/v1/monitors/post.ts#L15-L42)); docs can't drift from validation.

## Risks and tradeoffs (prioritized)

Full table with likelihood/impact in the [risk register](../09-reference/risk-register.md). The top five, argued:

1. **Non-transactional status transition (R1).** Read-check-write on `monitor.status` across awaits, guarded only by downstream dedupe ([checker/index.ts:128-348](../../../apps/workflows/src/checker/index.ts#L128-L348)). Hypothesis: duplicate or missed notifications under concurrent regional reports. Migration path: wrap flip+incident+trigger in one transaction, or serialize per-monitor (queue key = monitorId). Test strategy: concurrent-request integration test asserting exactly one trigger row (see [debugging scenario 1](../05-quality-engineering/03-systematic-debugging.md)).
2. **Trigger-before-send = at-most-once alerting (R2).** Crash between trigger insert and provider call suppresses the alert permanently ([alerting.ts:131-176](../../../apps/workflows/src/checker/alerting.ts#L131-L176)). For a paging product, at-least-once with provider-side dedupe is the defensible default. Migration: outbox table + dispatcher; or move send success/failure onto the trigger row (status column) with a retry sweep. Test: kill-between-steps fault injection.
3. **Silent telemetry loss (R3).** TB send failures only logged ([checker.go:255-257, 300-302](../../../apps/checker/handlers/checker.go#L255-L302)). Uptime *data* — the product — has weakest durability. Migration: local buffer + retry, or accept and *measure* loss rate (metric on failed sends) so the SLA is known.
4. **Single shared machine secret (R4).** `CRON_SECRET` authenticates cron→workflows, workflows→checker, checker→workflows. No rotation/scoping visible. Migration: per-edge secrets, or OIDC tokens from Cloud Tasks (GCP supports this natively).
5. **Audit coverage asymmetry (R5).** Pipeline and lifecycle mutations bypass `audit_log` ([checker/index.ts:227-231](../../../apps/workflows/src/checker/index.ts#L227-L231), [cron/monitor.ts:292-303](../../../apps/workflows/src/cron/monitor.ts#L292-L303)) — a support engineer asking "why is this monitor paused?" finds nothing. Migration: system-actor audit rows for `monitor.pause`/`monitor.status-flip` (cheap; the actor union already has `system`).

Tradeoffs I'd *defend* as-is: GCP Cloud Tasks dependency (huge operational win vs self-hosting purity — but it does complicate self-host; note `config.openstatus.yaml`/Coolify docs offer alternatives); schema duplication between tRPC and services (managed decoupling during migration); SQLite/libSQL choice (fits the read-mostly OLTP shape; revisit only with write-contention evidence).

## "What I'd change owning this for 3 months"

Month 1 — *make invariants true everywhere*: finish v1→services migration (the code already announces it, [v1/index.ts:143-147](../../../apps/server/src/routes/v1/index.ts#L143-L147)); add system-actor audit rows (R5); add the concurrent-transition test (R1 evidence-gathering before fixing).
Month 2 — *reliability*: outbox or status-on-trigger for notifications (R2); TB loss metric (R3); per-monitor serialization for `/updateStatus` (R1 fix).
Month 3 — *product hardening*: member roles for user actors (the documented gap in [require-scope.ts:19](../../../packages/services/src/auth/require-scope.ts#L19)); defense-in-depth on status-page data reads (R6); NOT NULL migration for `monitor.workspace_id`.

Each change: additive first, feature-flagged where behavior shifts, with the test named before the code (per-change test strategies above and in the risk register).

## Confirmed vs hypothesis ledger

Confirmed by reading (line-anchored above): no tx around status flip; trigger inserted before send; TB failures logged-only; scope no-op for user actors; v1 bypasses audit.
Hypotheses needing a repro or a maintainer's word: actual duplicate-notification frequency; libsql-HTTP tx isolation strength (R7); whether statusPage.get leaks gated-page data (R6); whether deleted monitors are excluded from cron selection (the query at [cron/checker.ts:98-108](../../../apps/workflows/src/cron/checker.ts#L98-L108) filters `active` and maintenance but — check — not `deletedAt`; *if* soft-deleted rows keep `active=true` this schedules ghost probes. Investigate before claiming.)

## Drill

Write your own top-3 risk list *before* reading this file's (cover it). Compare. Where you disagree with mine, argue it in writing — disagreement with justification is exactly what "architecture review" interviews want. Strong = your argument names evidence you'd gather, not just opinions.
