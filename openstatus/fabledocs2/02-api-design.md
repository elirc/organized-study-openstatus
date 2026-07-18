# Part 2 — API design

## What it is and why it exists

Four front doors, each with a different contract-stability promise:

1. **tRPC** (`packages/api`) — the dashboard's private API. The `AppRouter` type (`packages/api/src/root.ts:5-6`) is "public" to exactly one client that deploys in lockstep with the server. Versionless on purpose.
2. **REST v1** (`apps/server/src/routes/v1/`) — the real public API. OpenAPI-documented, API-key authed (`v1/index.ts:142-161`). This is the one you can't casually break.
3. **ConnectRPC v2 + MCP + Slack** (`apps/server/src/index.ts:187-258`) — newer machine surfaces; MCP is where the AI-agent story lives.
4. **Machine endpoints** — `CRON_SECRET`-authed internals (`/updateStatus` and friends). Internal, changeable, but shared across independently-deployed apps, so "changeable" means "changeable carefully."

Why four instead of one: each surface has a different consumer, auth model, and versioning cost. Collapsing them into one REST API would have made the dashboard pay versioning tax it doesn't need, and made the public API carry dashboard-shaped endpoints it shouldn't expose.

## The hard decisions

### 1. tRPC for the dashboard instead of eating our own REST API

The purist argument — "dogfood the public API!" — is seductive and I rejected it. The dashboard needs dozens of UI-shaped queries (this table, that chart, prefetched together) that would be embarrassing as public endpoints and instantly ossify into contracts. tRPC gives end-to-end types with zero codegen, and because client and server ship in the same deploy, breaking changes are free.

**The tradeoff that bothers me:** it created *two* implementations of every capability, and for a while, two implementations of the business rules. That's why `packages/services` exists — the rules live once, and both tRPC and REST are supposed to be thin adapters over it. REST v1 hasn't finished the migration; the handlers still do inline Drizzle with duplicated quota logic (`apps/server/src/routes/v1/monitors/post.ts:44-90`), and the comment at `v1/index.ts:143-147` is me publicly admitting it. When you see two ways to do the same thing in this repo, look for the comment that says which one is the future.

### 2. Contract-first REST: zod `createRoute` schemas generate the OpenAPI docs

Every v1 route defines its request/response as zod schemas inside `createRoute` (`monitors/post.ts:15-42`); the OpenAPI document is *generated from the validators*. **Rejected:** hand-written OpenAPI YAML (drifts from reality within a month, ask anyone) and docs-from-annotations (same drift, different syntax). With this design, the docs literally cannot describe behavior the validator doesn't enforce. Validation errors go through one `defaultHook` (`v1/index.ts:26`) so every endpoint fails with the same shape.

**Wart:** zod's default `.parse` *strips* unknown fields silently. Send `{"nmae": "prod-api"}` and you get a 200 and a monitor with no name change — no error, no warning. Fixing it (`.strict()` on inputs) is a breaking change for any client that's been sending garbage successfully, which is exactly why you fix this in v1 of an API and not v1.4. We didn't. Learn from me.

### 3. Deliberately duplicated schemas at the tRPC↔service boundary

The wire schema in the router and the domain schema in the service are separate zod objects, maintained by hand (`packages/api/src/router/monitor.ts:39-61` vs the service's `Input`). Every service verb **re-parses its input at the top** (`packages/services/src/monitor/create.ts:22`) even though the router already validated. Juniors flag this as redundant in their first review. It isn't: the service is called from tRPC, REST, Slack, MCP, and background jobs — it cannot assume any transport validated anything. **Validation belongs to whoever owns the invariant**, and the domain owns domain invariants.

**Rejected:** sharing one schema between wire and domain. That welds the public contract to the internal model — every internal refactor becomes an API change and vice versa. The duplication is a *managed* decoupling. The part that bothers me: it's maintained by hand, and nothing detects drift except tests and review.

### Error translation, one direction only

Services throw `ServiceError` subclasses (`NotFoundError`, `ForbiddenError`); each transport translates at its own edge — tRPC via `toTRPCError` in `packages/api/src/service-adapter.ts:32-87`, REST to status codes. The dependency arrow never reverses: `TRPCError` does not appear under `packages/services/`, enforced by package boundaries. Seven kinds of caller funnel into one `Actor` union (`packages/services/src/context.ts:13-20`) so downstream code branches on *who you are*, never on *which route called me*.

## War story

**Caused: the phantom attribution.** API-key auth resolves a key to a workspace, and enriches every log line with the key's id for attribution. There's a fallback path where a key validates but no `keyId` is available — early on this was silent, and we spent an afternoon staring at an audit question ("which key created these 40 monitors?") that had no answer, because a batch client was coming through the fallback. The fix is instructive: we didn't block the request, we made the degradation *loud* — `apps/server/src/libs/middlewares/auth.ts:106-114` warns every time attribution quality drops. That's now house style: instrument the degradation, not just the failure (Part 6 returns to this). Related honesty: the dev-mode key bypass grants `*` scopes (`auth.ts:246-261`) — it's NODE_ENV-guarded (L154), but you should personally verify that guard the first time you touch this file, not trust me.

## Before you move on

1. Why does `createMonitor` parse its input twice — and which callers would be unprotected if the service-side parse were deleted?
2. A field rename in the monitor response: walk the blast radius for tRPC vs REST v1. Why is one free and the other a versioning event?
3. Where exactly does a `ForbiddenError` thrown in a service become an HTTP 403 for REST and a TRPC `FORBIDDEN` for the dashboard? Name the files.
