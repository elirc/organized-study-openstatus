# System Map

## Shape: pnpm + Turborepo monorepo

Declared in [package.json](../../../package.json#L38): workspaces are `apps/*`, `packages/**/*`. Turbo ([turbo.json](../../../turbo.json)) orchestrates `build`/`dev`/`test`/`dx` with dependency edges (`dx → seed → migrate → env`).

```
                       ┌────────────────────────────────────────────────┐
                       │                  packages/                     │
                       │  db (Drizzle+libSQL)  services (biz logic)     │
                       │  api (tRPC)  tinybird  notifications  emails   │
                       │  assertions  regions  ui  utils  analytics …   │
                       └───────▲───────────▲───────────▲────────────────┘
                               │           │           │
   ┌───────────┐   tRPC   ┌────┴─────┐  ┌──┴───────┐  ┌┴─────────────┐
   │ dashboard │─────────▶│ pkg/api  │  │ server   │  │ workflows    │
   │ (Next.js) │          │ routers  │  │ (Hono,   │  │ (Hono, Bun)  │
   └───────────┘          └──────────┘  │  Bun)    │  │ cron + alert │
   ┌───────────┐                        │ REST v1  │  └──────▲───────┘
   │status-page│── proxy.ts ── db       │ MCP/RPC  │         │ /updateStatus
   │ (Next.js) │                        │ Slack    │   GCP Cloud Tasks
   └───────────┘                        └──────────┘         │
   ┌───────────┐                                       ┌─────┴────────┐
   │    web    │  marketing site                       │ checker (Go) │──▶ Tinybird
   └───────────┘                                       │ 28 regions   │
                                                       └──────────────┘
```

## Apps (deployables)

| App | Runtime | Role | Entry |
| --- | --- | --- | --- |
| `apps/dashboard` | Next.js (Node + Edge for tRPC) | Authenticated product UI: monitors, status pages, notifications, audit logs, chat agent | [layout/app router](../../../apps/dashboard/src/app) |
| `apps/status-page` | Next.js | Public status pages on subdomains/custom domains; password/SSO/IP gating in [proxy.ts](../../../apps/status-page/src/proxy.ts#L15-L96) | `src/app/(status-page)/[domain]/[locale]` |
| `apps/web` | Next.js | Marketing site + legacy app (not read in depth here) | — |
| `apps/server` | Bun + Hono | External API: REST v1, ConnectRPC v2, MCP, Slack agent, public endpoints | [index.ts](../../../apps/server/src/index.ts#L38-L258) |
| `apps/workflows` | Bun + Hono on Fly | Cron fan-out + alerting pipeline + lifecycle emails | [index.ts](../../../apps/workflows/src/index.ts#L102-L194) |
| `apps/checker` | Go (Gin) on Fly/Koyeb/Railway | Executes HTTP/TCP/DNS probes, writes results to Tinybird, reports status changes | [handlers/checker.go](../../../apps/checker/handlers/checker.go#L47) |
| `apps/private-location` / `apps/railway-proxy` / `apps/screenshot-service` / `apps/ssh-server` / `apps/docs` | Go/Node | Self-hosted probe agent, region proxy, screenshots, SSH demo, docs site | not covered in depth |

## Key packages

| Package | Owns | Must not own |
| --- | --- | --- |
| `packages/db` | Schema ([schema/](../../../packages/db/src/schema)), migrations, seed, the Drizzle client ([db.ts](../../../packages/db/src/db.ts#L6-L12)) | Business rules |
| `packages/services` | All workspace-scoped business logic, one verb per file; audit + scopes ([CLAUDE.md](../../../CLAUDE.md) codifies this) | HTTP/transport concerns |
| `packages/api` | tRPC routers (thin), context building ([trpc.ts](../../../packages/api/src/trpc.ts#L148-L241)), service adapter | Business logic (inline DB in routers is a documented smell) |
| `packages/tinybird` | Typed Tinybird pipe clients ([client.ts](../../../packages/tinybird/src/client.ts#L15-L50)) | Nothing else |
| `packages/notifications/*` | One sub-package per provider (slack, discord, pagerduty, email, sms, webhook, opsgenie…) | — |
| `packages/assertions` | Probe assertion schemas shared TS-side | Go evaluation (that's [checker.go](../../../apps/checker/handlers/checker.go#L334-L372)) |
| `packages/emails`, `packages/analytics`, `packages/upstash`, `packages/regions`, `packages/ui` | Email templates, product analytics, Redis, region dictionary, shadcn components | — |

## Public interfaces vs private internals

**Public (contracts you can't casually break):**
- REST v1 under `/v1` — OpenAPI-documented, API-key authed ([v1/index.ts](../../../apps/server/src/routes/v1/index.ts#L142-L161)).
- ConnectRPC v2 + MCP + Slack surfaces ([server index.ts](../../../apps/server/src/index.ts#L187-L258)).
- The status-page URL space (slugs + custom domains) and its subscriber endpoints.
- The tRPC `AppRouter` type ([root.ts](../../../packages/api/src/root.ts#L5-L6)) — "public" to the dashboard only, but versionless: server and client deploy together.

**Internal (can change freely if callers in-repo are updated):**
- `packages/services` function signatures, DB schema, Tinybird pipe names (versioned with `__v1` suffixes — note `ping_response__v8` in [checker.go](../../../apps/checker/handlers/checker.go#L50)), the `CRON_SECRET`-authed endpoints between workflows/checker.

**Ownership map for changes:**

| You want to change… | Touch | Review focus |
| --- | --- | --- |
| A dashboard screen | `apps/dashboard/src/app/(dashboard)/…` + maybe a router | UI state, cache invalidation |
| A business rule | `packages/services/src/<entity>/<verb>.ts` + tests | scope check, tx, audit, workspace filter |
| The wire contract (REST) | `apps/server/src/routes/v1/…` + OpenAPI schema | backward compatibility |
| The schema | `packages/db/src/schema/…` + `drizzle-kit generate` migration | rollout order, nullability, backfill |
| Alerting behavior | `apps/workflows/src/checker/…` | idempotency, duplicate notifications |
| Probe mechanics | `apps/checker/**/*.go` | timeout, retry, region semantics |

## Senior noticing

- Three separate HTTP servers share one database. The **boundary** between them is enforced only by convention + a shared `CRON_SECRET` ([checker/index.ts:86-91](../../../apps/workflows/src/checker/index.ts#L86-L91)) — there is no service mesh or per-service credential. Blast radius of a leaked secret: full alerting pipeline.
- The dashboard's tRPC runs on **Next.js Edge**, which is why `packages/services` must stay free of `node:*` imports — an invariant documented in code at [emit.ts:14-24](../../../packages/services/src/audit/emit.ts#L14-L24) (hand-rolled `deepEqual` instead of `node:util`).
- Legacy vs current: v1 REST routes still use inline Drizzle; the code comment at [v1/index.ts:143-147](../../../apps/server/src/routes/v1/index.ts#L143-L147) tells you the migration to services is in progress. When you see two ways to do the same thing, look for the comment that says which is the future.

## Drill

Without opening the docs again, draw this diagram from memory, then diff against the real one. Anything you forgot, open its entry file. Self-grade: **Basic** = apps only; **Solid** = apps + which package each depends on; **Strong** = also marked the three trust boundaries (session cookie, API key, CRON_SECRET) and where money flows (Stripe in dashboard, plan limits in workspace.limits).
