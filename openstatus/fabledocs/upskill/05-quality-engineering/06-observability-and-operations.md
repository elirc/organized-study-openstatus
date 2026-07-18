# Observability & Operations

## What exists

| Signal | Implementation | Anchor |
| --- | --- | --- |
| Canonical request logs (wide events) | one accumulated `event` per request, emitted at end | [server index.ts:109-179](../../../apps/server/src/index.ts#L109-L179) |
| Tail sampling | keep all 5xx/errors/slow(>2s), sample rest 20% (workflows keeps 4xx at 100%) | [index.ts:90-100](../../../apps/server/src/index.ts#L90-L100), [workflows index.ts:42-62](../../../apps/workflows/src/index.ts#L42-L62) |
| Log shipping | logtape → OTLP → Axiom; console JSON lines in dev | [index.ts:50-87](../../../apps/server/src/index.ts#L50-L87) |
| Errors | Sentry middleware both Hono apps; `cause` preserved through error adapters | [index.ts:105](../../../apps/server/src/index.ts#L105), [workflows index.ts:165-178](../../../apps/workflows/src/index.ts#L165-L178) |
| Request correlation | `requestId()` middleware + `AsyncLocalStorage` context | [index.ts:106-120](../../../apps/server/src/index.ts#L106-L120) |
| Domain event enrichment | handlers append domain facts to the wide event (e.g. `status_update` with affected regions) | [checker/index.ts:153-162](../../../apps/workflows/src/checker/index.ts#L153-L162), auth enrichment [auth.ts:91-98](../../../apps/server/src/libs/middlewares/auth.ts#L91-L98) |
| Health | `/ping` on both services | [index.ts:197-202](../../../apps/server/src/index.ts#L197-L202) |
| Ops audit trail | `checkerAudit.publishAuditLog` for monitor.failed/recovered/incident.* | [checker/index.ts:170-211](../../../apps/workflows/src/checker/index.ts#L170-L211) |
| User-facing OTel | per-monitor metrics export to the *customer's* OTLP endpoint | [checker.go:315-317](../../../apps/checker/handlers/checker.go#L315-L317) |

Gaps to note: no metrics/dashboards in-repo (counters live implicitly in logs); no tracing spans across the cron→checker→updateStatus hop (requestId doesn't propagate through Cloud Tasks — *verify*; the payloads carry `cronTimestamp` which serves as a weak correlation key).

## "How would I know this broke?" — per flow

| Flow | Detection today | Blind spot |
| --- | --- | --- |
| Monitor create (Flow 1) | wide event `outcome=error`, Sentry with cause | silent zod strip (field ignored) is invisible — no error, wrong behavior |
| Cron fan-out (Flow 2) | summary log with success/failed counts + Sentry message on failures ([cron/checker.ts:225-242](../../../apps/workflows/src/cron/checker.ts#L225-L242)) | **cron not firing at all** — absence of logs; needs a dead-man's-switch (external heartbeat) |
| Probe execution | TB rows exist; checker logs | TB send failure = logged in checker only; no loss-rate metric (risk R3) |
| Alerting (Flow 3) | wide event `status_update.notificationTriggered` + `notifications` list ([checker/index.ts:350-353](../../../apps/workflows/src/checker/index.ts#L350-L353)); per-send error logs | suppressed-by-dedupe sends look identical to "no channels configured" in logs; trigger row without provider success is unqueryable truth |
| Status page (Flow 4) | `[proxy]` console logs with action + reason ([proxy.ts:52-86](../../../apps/status-page/src/proxy.ts#L52-L86)) | schema-invalid page rows fail *open* silently — no counter |
| REST v1 (Flow 5) | wide events with `auth_method`, workspace enrichment | keyId-missing fallback is warned ([auth.ts:106-114](../../../apps/server/src/libs/middlewares/auth.ts#L106-L114)) — good example of *instrumenting a degradation* |
| Deploys | per-app GH Actions | cross-app contract skew (payload change deployed to workflows before checker) has no canary |

## Operational judgment to internalize

1. **Instrument the degradation, not just the failure** — the keyId-fallback warn is the model: the system still works, but attribution quality dropped, and someone will want to know when.
2. **Absence is the hardest signal** — cron silence, zero TB rows, zero notifications. Every periodic process needs a "this should have happened by now" watcher; none is visible here (good mid-ticket).
3. **Sampling must be outcome-aware** — deciding *after* the response (tail) is why errors are never lost here. If you ever see head-sampling on an error path, flag it.
4. **Rollback story:** deploys are per-app images; DB migrations are forward-only. So rollback = redeploy previous image, and schema changes must stay backward-compatible one version — which is *the* reason for the additive-migration discipline in [03/02](../03-architecture-and-patterns/02-data-model-and-persistence.md).

## Drill

Design the missing alerting-pipeline dashboard: five queries over the wide events (by name: e.g. `count status_update by status`, `p99 duration_ms of /updateStatus`, `notificationTriggered=false where status flipped`, cron success/failed ratio, TB send failures). For each: what page/alert threshold, and which incident it would have caught in [debugging scenarios 1–2](03-systematic-debugging.md).
Self-grade — Basic: 5 plausible queries. Solid: thresholds with reasoning. Strong: includes one absence-detector and one degradation-detector.

## Interview angle

"How do you monitor a distributed pipeline?" / "What's your observability philosophy?" — wide events + tail sampling + absence detection, all with anchors. This doubles as the "how would you know?" follow-up in system design ([08/04](../08-interview-prep/04-system-design-from-this-repo.md)).
