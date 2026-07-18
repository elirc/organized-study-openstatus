# Part 1 — Architecture

## What it is and why it exists

A pnpm + Turborepo monorepo (`package.json` workspaces: `apps/*`, `packages/**/*`; pipeline in `turbo.json`) with five deployables that matter and a dozen shared packages:

- **`apps/dashboard`** — Next.js, the authenticated product UI. Talks tRPC.
- **`apps/status-page`** — Next.js, the *public* surface. Subdomains and custom domains, with password/SSO/IP gating decided in `apps/status-page/src/proxy.ts:15-96`.
- **`apps/server`** — Bun + Hono. The external API: REST v1 (OpenAPI), ConnectRPC v2, MCP, Slack. Entry: `apps/server/src/index.ts:38-258`.
- **`apps/workflows`** — Bun + Hono on Fly. The heart: cron fan-out, the `/updateStatus` state machine, alerting, lifecycle emails. Entry: `apps/workflows/src/index.ts:102-194`.
- **`apps/checker`** — Go (Gin). Executes the actual HTTP/TCP/DNS probes from ~28 regions, writes results to Tinybird, reports status transitions back to workflows. Entry: `apps/checker/handlers/checker.go:47`.

Why this shape: monitoring is two workloads wearing one product. There's a **CRUD workload** (users editing monitors and pages — low volume, needs transactions and auth) and a **firehose workload** (probes × regions × frequency — high volume, append-only, latency-sensitive). Trying to serve both from one Next.js app is how you get a status-page product whose own status page goes down. So the CRUD side is TypeScript on managed platforms, and the probe side is a Go binary you can deploy to any region cheaply.

The packages that carry the architecture: `packages/services` (all business logic, one verb per file — this is load-bearing, see CLAUDE.md), `packages/db` (Drizzle schema + client, no business rules), `packages/api` (thin tRPC routers), `packages/tinybird` (typed pipe clients).

## The hard decisions

### 1. Polyglot: Go for the checker, TypeScript for everything else

**Rejected alternative A:** all-TypeScript, run probes from the Bun services. We'd have one language, shared types, one deploy story. I said no because probe workers need to be *tiny and everywhere* — a static Go binary deploys to Fly/Koyeb/Railway regions in seconds with a single-digit-MB footprint, and Go's `net` stack gives us the timeout/redirect/DNS control a monitoring product lives or dies on (see the per-request client honoring per-monitor timeouts, `checker.go:76-79`).

**Rejected alternative B:** share the contract via codegen (protobuf everywhere). Too much machinery for what is, honestly, one JSON payload. Instead the TS↔Go boundary is **data-only**: workflows builds a JSON body (`apps/workflows/src/cron/checker.ts:249-320`), Go types it independently in its own structs. No shared code at all.

**What still bothers me:** that boundary is enforced by *discipline, not tooling*. If you add a field to the payload in workflows and forget the Go struct, nothing fails at build time — the field silently drops. Deploy order across the two apps has no canary. It's on my "we'd do this differently today" list: a JSON-schema check in CI on both sides would cost a day and has never been done.

### 2. GCP Cloud Tasks as the queue, not a self-hosted broker

The cron tick fans out one Cloud Task per monitor×region (`cron/checker.ts:245-356`); the checker posts status transitions back through a queue-backed task (`apps/checker/checker/update.go:29-93`). **Rejected:** Redis streams or a self-hosted broker, which the "we're open source, everything must self-host" instinct wanted. I chose the managed queue because it gave us retries, throttling, and deterministic task names (our idempotency layer — task name = identity, so a double-enqueued cron tick is a no-op) for free, and queues are precisely the component you don't want to operate yourself while also selling uptime.

**The tradeoff that bothers me:** it complicates self-hosting — hence the compensating machinery in `config.openstatus.yaml` and the Coolify docs. And I accepted a hard vendor coupling in the most critical path of the product. I'd make the same call again, but I'd make it *behind an interface*; today the Cloud Tasks client is called directly.

### 3. `packages/services` must run on Next.js Edge

The dashboard's tRPC executes on the Edge runtime, so every service file must avoid `node:*` imports. You can see the scar tissue: `packages/services/src/audit/emit.ts:14-24` hand-rolls `deepEqual` instead of importing `node:util`. **Rejected:** running tRPC on Node only. Edge gave us cheap, fast, globally-distributed reads for a dashboard whose users are, by definition, having a bad day and refreshing a lot. **Cost:** an invisible invariant. Nothing *stops* you importing `node:crypto` into a service verb; you find out when the Edge bundle fails, or worse, at runtime. This is convention enforced by code review, and it has been violated in PRs more than once.

### The one that keeps me up: a single machine secret

Three HTTP services share one database, and the machine-to-machine hops (cron→workflows, workflows→checker, checker→workflows) all authenticate with the same static `CRON_SECRET` (`apps/workflows/src/checker/index.ts:86-91`, `checker.go:52`). No rotation, no per-edge scoping. Leak it once and the blast radius is the entire alerting pipeline. Cloud Tasks supports OIDC tokens natively; we've just never done the work. If you want a high-leverage first infrastructure PR, this is it.

## War story

**Prevented: the flapping pager.** Early design had the checker flip `monitor.status` directly on any failed probe. Play that forward: one region has a transient network blip, the monitor flips to error, alerts fire, next probe from another region flips it back, recovery notifications fire — your customer gets paged four times for zero real downtime, at 3am, and churns. The shipped design separates **evidence from verdict**: each region writes its own `monitor_status` row (evidence), and the global `monitor.status` only flips when a quorum of regions agrees (`apps/workflows/src/checker/index.ts:216`). Anti-flapping is structural, not a debounce timer someone tunes forever. The remaining wart — the quorum check itself is a non-transactional read-check-write across awaits — is Part 3's problem.

## Before you move on

1. A request to change a monitor's name, and a probe result from Frankfurt: list every process and package each one touches, in order.
2. Why can't `packages/services` import `node:util`, what breaks if it does, and at what point in the lifecycle do you find out?
3. The `CRON_SECRET` leaks. What can an attacker do, concretely — which endpoints, which writes? Sketch the fix using Cloud Tasks OIDC.
