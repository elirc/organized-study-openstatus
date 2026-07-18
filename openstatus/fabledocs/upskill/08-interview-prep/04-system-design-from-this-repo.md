# System Design From This Repo

The whiteboard exercise: **"Design an uptime-monitoring platform with public status pages — think StatusPage + Pingdom."** You have studied a production implementation; answer with its choices, its tradeoffs, and where you'd deviate. Cross-reference: [03-architecture-and-patterns/06-architecture-critique.md](../03-architecture-and-patterns/06-architecture-critique.md) is the critique twin of this file.

Time-box a full run: 35–40 minutes aloud.

## Step 1 — Requirements (5 min)

Functional: create monitors (HTTP/TCP/DNS), probe from N regions on a schedule, detect down/degraded, alert via many channels, public status pages (custom domains, access control), subscriber notifications, REST API + dashboard.
Non-functional: probe volume >> user traffic (write-heavy telemetry); alert latency ~1 probe interval; false-positive control (one bad region ≠ incident); multi-tenant isolation; self-hostable.

*What this repo chose:* exactly this scope; multi-cloud regions are a differentiator ([README.md](../../../README.md#L35-L42)).

- **Junior answer sounds like:** jumps straight to tables and endpoints.
- **Mid adds:** separates telemetry volume from CRUD volume; asks about alert latency targets.
- **Senior includes:** "the hard part is alert *correctness* (flapping, partial outages), everything else is CRUD."

## Step 2 — API sketch (5 min)

Two client surfaces: an authenticated app API and a public API.
*Repo's choice:* tRPC for the first-party dashboard ([root.ts](../../../packages/api/src/root.ts#L5-L6)) — type-safe, versionless, deploys with its only client; OpenAPI REST for external users ([v1/index.ts](../../../apps/server/src/routes/v1/index.ts#L25-L27)) — versioned, documented, key-authed with scopes ([auth.ts](../../../apps/server/src/libs/middlewares/auth.ts#L124-L261)). Business logic shared underneath via a service layer ([create.ts](../../../packages/services/src/monitor/create.ts#L16-L63)) so the two surfaces can't drift.
*Stronger/simpler alternative:* one REST API for both — simpler, but loses dashboard iteration speed; GraphQL — flexible reads but overkill for this write-shaped domain.

- **Junior:** lists endpoints.
- **Mid:** justifies tRPC-for-internal/REST-for-external split; mentions input validation at the edge.
- **Senior:** raises contract governance — who may break what, and how the service layer prevents drift; notes error-code consistency across surfaces (this repo has a live 402-vs-429 inconsistency to cite — [post.ts:59-64](../../../apps/server/src/routes/v1/monitors/post.ts#L59-L64) vs [service-adapter.ts:57-65](../../../packages/api/src/service-adapter.ts#L57-L65)).

## Step 3 — Data model (7 min)

Core entities: workspace (tenant, plan/limits), monitor, per-region monitor_status, incident, page, page_component, status_report(+updates), subscriber, notification(+monitor join), audit_log.

*Repo's choices worth naming aloud:*
1. **Two data planes.** Relational state in SQLite-compatible libSQL ([db.ts](../../../packages/db/src/db.ts#L6-L12)); high-volume probe results in Tinybird (append-only, ClickHouse-family) — [checker.go:255](../../../apps/checker/handlers/checker.go#L255). *Why:* millions of pings/day would drown an OLTP DB; status pages need fast aggregates (uptime %, p95 latency). *Alternative:* Timescale/pg partitions (one system, more ops burden).
2. **Aggregate vs evidence.** `monitor.status` (one value) vs `monitor_status` per region (evidence) — [monitor.ts](../../../packages/db/src/schema/monitors/monitor.ts#L23), [alerting.ts:276-302](../../../apps/workflows/src/checker/alerting.ts#L276-L302).
3. **Soft delete + partial index** — `deletedAt` with `WHERE deleted_at IS NULL` index ([monitor.ts:66-72](../../../packages/db/src/schema/monitors/monitor.ts#L66-L72)).
4. **Plan limits as a JSON column** on workspace ([workspace.ts:22](../../../packages/db/src/schema/workspaces/workspace.ts#L22)) — flexible, but unconstrained by the DB.
5. Denormalized sins to confess if asked: `regions` as comma-joined text ([monitor.ts:26](../../../packages/db/src/schema/monitors/monitor.ts#L26)), nullable `workspaceId` on monitor (L37).

- **Junior:** tables + FKs.
- **Mid:** separates telemetry store; explains soft delete and the partial index.
- **Senior:** talks retention/rollups in the analytics plane, and which columns are *load-bearing invariants* (workspaceId scoping) vs conveniences.

## Step 4 — The probe pipeline (10 min — spend your time here)

*Repo's architecture:*

```
cron (per periodicity) ──▶ select active monitors (skip maintenance)
     └─▶ Cloud Task per (monitor × region), name = monitor-{id}-{region}-{ts}   ← idempotent enqueue
              └─▶ Go checker in-region: probe w/ timeout+retries, assertions
                       ├─▶ Tinybird (every result)
                       └─▶ if state changed: Cloud Task → /updateStatus
                                └─▶ upsert region status → quorum ≥ half → flip monitor.status
                                     → create/resolve incident → fan out notifications
                                        (dedupe: unique notificationTrigger insert)
```

Anchors: fan-out [cron/checker.ts:65-243](../../../apps/workflows/src/cron/checker.ts#L65-L243); probe [checker.go:120-332](../../../apps/checker/handlers/checker.go#L120-L332); quorum + incidents [checker/index.ts:86-356](../../../apps/workflows/src/checker/index.ts#L86-L356); dedupe [alerting.ts:131-139](../../../apps/workflows/src/checker/alerting.ts#L131-L139).

Design points to hit, each with the repo's answer:
- **Why a queue between cron and probes?** Burst absorption, per-region routing (provider headers — [cron/checker.ts:325-335](../../../apps/workflows/src/cron/checker.ts#L325-L335)), retry semantics owned by infra.
- **Idempotency:** deterministic task names; ALREADY_EXISTS treated as success ([cron/checker.ts:192-202](../../../apps/workflows/src/cron/checker.ts#L192-L202)).
- **Flap control:** edge-triggered updates (only on state *change*), quorum before aggregate flip ([checker/index.ts:216](../../../apps/workflows/src/checker/index.ts#L216)), retries inside the probe before declaring failure.
- **Weakness to volunteer (senior move):** the quorum read + `monitor.status` write is not transactional; correctness leans on the notification-dedupe insert. Fix: transaction or per-monitor serialization (queue key), or CAS on `monitor.status`.
*Simpler alternative:* single-region checks with in-process scheduler (node-cron) — fine to say for v1, then explain what multi-region adds and costs.

- **Junior:** "a cron pings the URLs and sends alerts."
- **Mid:** queue + idempotency + quorum, with the actual mechanisms.
- **Senior:** failure-mode enumeration: GCP down (checks stop — silent), TB write fails (data loss, logged only — [checker.go:255-257](../../../apps/checker/handlers/checker.go#L255-L257)), duplicate /updateStatus (dedupe insert), notification provider down (3 retries then give up — no DLQ). Then proposes an outbox/DLQ and says what it buys.

## Step 5 — Status pages + tenancy (5 min)

*Repo's choice:* separate Next.js app; middleware resolves host → page row → access action (public/password/SSO-domains/IP) — [proxy.ts:15-96](../../../apps/status-page/src/proxy.ts#L15-L96). Public data via output-schema-constrained tRPC ([statusPage.ts:64-113](../../../packages/api/src/router/statusPage.ts#L64-L113)).
Talk through: custom domains (host-header routing), caching (per-page revalidation; Tinybird pipe caching [client.ts:11-13](../../../packages/tinybird/src/client.ts#L11-L13)), and the defense-in-depth question (gate at edge only vs also at data layer).

## Step 6 — Scaling & variations (5 min)

Prepared answers for the classic follow-ups:

1. **"10× monitors"** — pressure lands on the fan-out loop (tasks per tick = monitors × regions) and Tinybird ingest. Shard cron by periodicity (already done — queue per periodicity), then by hash range; batch TB events (Go checker sends per-probe today).
2. **"Real-time status page updates"** — poll → SSE/WebSocket; publish on the `/updateStatus` path; careful: that path must stay fast, so publish async.
3. **"Add multi-tenancy"** — already deep: workspace scoping by construction ([internal.ts:28-47](../../../packages/services/src/monitor/internal.ts#L28-L47)), plan limits as data, audit per workspace. Talk isolation testing.
4. **"On-prem/private network monitoring"** — repo answer exists: private locations (self-hosted checker agent, `apps/private-location`) — inverted connectivity, agent polls for jobs.
5. **"Alert routing / on-call schedules"** — new bounded context; keep provider fan-out, add routing rules before it; don't bloat `triggerNotifications`.

## Grading rubric (self-assess a recorded run)

| Dimension | Basic | Solid | Strong |
| --- | --- | --- | --- |
| Requirements | lists features | separates telemetry from CRUD | quantifies probe volume, names alert correctness as the crux |
| Data | tables | two data planes + soft delete | invariants + retention story |
| Pipeline | cron+queue | idempotency + quorum mechanics | failure enumeration + concrete fix proposals |
| Communication | needs prompting | structured, anchored examples | drives the conversation, volunteers tradeoffs |

Practice drill: run Steps 1–6 aloud in 35 minutes, recorded. Once with the repo open (cite files), once without. The second run is your interview answer.
