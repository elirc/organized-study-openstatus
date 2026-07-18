# Part 6 — Observability

## What it is and why it exists

The philosophy in one line: **one wide event per request, tail-sampled, shipped to Axiom.** Instead of scattering ten `log.info` calls through a handler, middleware accumulates a single canonical event — route, actor, workspace, duration, outcome, plus domain facts handlers append along the way — and emits it once at the end (`apps/server/src/index.ts:109-179`). Handlers enrich it: the status-update path appends affected regions (`apps/workflows/src/checker/index.ts:153-162`); auth middleware appends attribution (`apps/server/src/libs/middlewares/auth.ts:91-98`).

The supporting cast: **tail sampling** — keep 100% of 5xx/errors/slow(>2s), sample the boring rest at 20%, and workflows keeps 4xx at 100% too (`index.ts:90-100`, `apps/workflows/src/index.ts:42-62`); **shipping** via logtape → OTLP → Axiom, console JSON in dev (`index.ts:50-87`); **Sentry** on both Hono apps with `cause` preserved through the error adapters; **request correlation** via `requestId()` + `AsyncLocalStorage` (`index.ts:106-120`); a curious extra — per-monitor OTel metrics exported to the *customer's* OTLP endpoint (`checker.go:315-317`), observability as a product feature.

Why this design: a monitoring product's own telemetry is where you go during *your* worst day, which is usually also your customers' worst day (mass outage = peak traffic = peak log volume). Wide events keep queries cheap ("show me every failed /updateStatus for workspace X" is one filter, not a correlation job), and tail sampling keeps volume sane without ever losing an error.

## The hard decisions

### 1. Wide events over scattered logs — and over metrics

**Rejected alternative A:** conventional multi-line logging. Ten lines per request that you reassemble by requestId at 2am, with the interesting fact always in the line someone forgot to write. **Rejected alternative B:** a real metrics stack (Prometheus-style counters/histograms + dashboards). We query counters *out of* the wide events in Axiom instead. That second rejection is the tradeoff that still bothers me: there are **no metrics or dashboards in this repo**. Every "how often does X happen" question is an ad-hoc log query someone runs after getting suspicious, rather than a line on a dashboard that got visibly worse. Cheaper to build, more expensive to *notice* with.

### 2. Sample at the tail, decide after the response

Head-based sampling decides "keep this request?" before knowing how it ends — which means it throws away errors with the same probability as successes. Ours decides after: outcome-aware, so errors and slow requests are never lost (`index.ts:90-100`). **Cost:** every request pays full event-accumulation overhead even if it's ultimately dropped, and the sampler runs in-process (no central collector to retune rates without a deploy). Both fine at our volume. If you ever see head-sampling on an error path anywhere, flag it in review — it's the observability equivalent of catching and swallowing exceptions.

### 3. Correlation by requestId — which dies at the queue boundary

`requestId` + `AsyncLocalStorage` gives clean intra-service correlation, but the pipeline's interesting failures are *cross*-service: cron → Cloud Tasks → checker → /updateStatus. The requestId does not propagate through Cloud Tasks; the only cross-hop key is the `cronTimestamp` carried in payloads — a weak join at best. **Rejected (by inertia more than argument):** full distributed tracing with context propagation through task headers. Honest assessment: for a pipeline this shaped, OTel spans through the queue would be worth it today, and this is a "we'd do it differently now" item. When you debug your first cross-hop mystery, you'll feel exactly where the seam is.

### The blind-spot map (know these before you're on call)

| Flow | You'd know because… | You *wouldn't* know if… |
| --- | --- | --- |
| Cron fan-out | summary log with success/failed counts + Sentry on failures (`cron/checker.ts:225-242`) | **cron stops firing entirely** — absence of logs looks like health; no dead-man's-switch exists |
| Probe telemetry | Tinybird rows exist | TB send fails — logged in the checker and the row is simply lost (`checker.go:255-257`); no loss-rate metric, so our data plane's SLA is unknown |
| Alerting | wide event `notificationTriggered` + per-send error logs (`checker/index.ts:350-353`) | a send was suppressed by dedupe — indistinguishable in logs from "no channels configured" |
| Status page gating | `[proxy]` logs with action + reason (`proxy.ts:52-86`) | a schema-invalid page row fails open — silently, no counter |

The generalized lesson, and the hardest one in observability: **absence is the hardest signal.** Every periodic process needs a "this should have happened by now" watcher, and none is visible in this repo. Also the house rule from Part 2: *instrument the degradation, not just the failure* — the keyId-fallback warn (`auth.ts:106-114`) is the model; the system still works but attribution quality dropped, and someone will want to know when.

## War story

**Prevented: the 3am query that actually worked.** Mass-outage drill: a cloud provider region browns out, three hundred monitors flip at once, and support wants to know *which customers got notified* while it's still happening. Because every /updateStatus emits one wide event carrying monitorId, workspace, transition, affected regions, and `notificationTriggered`, that's a single Axiom filter with a group-by — during the highest-traffic minutes we'd ever seen, with tail sampling guaranteeing none of the error-path events were dropped. In the multi-line-logging counterfactual, that's a correlation job across four log streams under peak volume, and the answer arrives after the outage does. This design earned its keep in one query. The asterisk: the same drill exposed the dedupe-vs-no-channels blind spot in the table above — a wide event can only answer questions about fields someone appended. Widening the event is almost always the right fix; do it in the same PR as the feature, not after the incident.

## Before you move on

1. Cron silently stops firing at 02:00. Walk the detection story with today's tooling — when and how does anyone find out? Then design the dead-man's-switch: what emits the heartbeat, what watches it, what's the threshold?
2. Why is tail sampling strictly better than head sampling for error investigation, and what does it cost? Where in the code is the keep/drop decision made?
3. Design the five queries for the missing alerting-pipeline dashboard (from wide events alone). For each: the alert threshold, and — the senior test — which of them is an *absence* detector and which is a *degradation* detector.
