# ADR-003: Go probe fleet behind a data-only boundary, GCP Cloud Tasks as the spine

**Status:** Accepted, stable; the shared-secret authentication between hops is the acknowledged weak point.

## Context

The product's core promise is probing customer endpoints from many regions with tight control over timeouts, redirects, DNS, and TCP behavior — at a volume orders of magnitude above the CRUD traffic. Probes must be cheap to deploy anywhere (Fly, Koyeb, Railway) and must not share a failure domain with the dashboard. Between the cron scheduler and the probes sits a fan-out problem: every tick, thousands of monitor×region tasks with retries and throttling.

## Decision

Three coupled choices. (1) The prober is a **Go binary** (`apps/checker`, Gin) — static, small, with per-request `http.Client` control. (2) The TS↔Go boundary is **data-only**: workflows serializes a JSON payload, Go deserializes into its own structs; zero shared code, no codegen. (3) **GCP Cloud Tasks** is the delivery spine: cron enqueues one task per monitor×region with a **deterministic task name** (making double-enqueue a no-op), and the checker reports status transitions back through a queue-backed task to `/updateStatus`. Machine hops authenticate with a shared `CRON_SECRET`.

## Evidence

- `apps/checker/handlers/checker.go:47-79` — probe handler; per-request client honoring per-monitor timeouts (L76-79); Go-side payload structs.
- `apps/workflows/src/cron/checker.ts:245-356` — fan-out with deterministic task names, bounded concurrency (100), Effect-based retry on enqueue (L204-208).
- `apps/checker/checker/update.go:29-93` — queue-backed status-change report.
- `apps/workflows/src/checker/index.ts:86-91` + `checker.go:52` — the shared `CRON_SECRET` on both ends.

## Alternatives considered

1. **All-TypeScript probes** (Bun workers). Rejected: deployment footprint and per-request network control both favor Go; a monitoring vendor's probe quality *is* the product.
2. **Shared contract via protobuf/codegen.** Rejected (inferred): one JSON payload didn't justify a codegen toolchain spanning two languages. Cost accepted: drift between the TS payload builder and Go structs fails silently — no CI check exists.
3. **Self-hosted queue** (Redis streams, RabbitMQ). Rejected: retries, throttling, and named-task idempotency arrive free with Cloud Tasks; operating a broker while selling uptime is negative-margin work. Cost accepted: hard vendor coupling in the most critical path, and a complicated self-host story (compensated by `config.openstatus.yaml` / Coolify docs).
4. **Per-edge credentials or Cloud Tasks OIDC** instead of one static secret. Not adopted (yet): one leaked `CRON_SECRET` exposes the entire pipeline; OIDC is natively supported and remains the obvious successor decision.

## Consequences

**Positive:** probes deploy to any region in seconds; independent release cadence for prober vs product; the queue absorbs probe bursts (backpressure by architecture); deterministic task names give exactly-once *enqueue* semantics without any coordination.

**Negative:** three services + a binary share one database with only convention as the boundary; the JSON contract drifts silently without a schema check; request correlation dies at the queue (`requestId` doesn't propagate through Cloud Tasks — the only cross-hop key is `cronTimestamp`); and `/updateStatus` fan-in during a mass outage has no per-request backpressure — every monitor×region transitioning at once hits unbounded DB work per request.
