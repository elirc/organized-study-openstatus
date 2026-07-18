# Framework Mental Models

## Next.js App Router: server-first, opt into the client

Mental model: everything is a Server Component until a file says `"use client"`. Server components fetch and render on the server (no bundle cost, no hooks); client components hydrate and own interactivity. The route tree *is* the layout tree.

Where the repo uses it well:
- Route groups organize without affecting URLs: `(dashboard)` in [apps/dashboard/src/app/(dashboard)](../../../apps/dashboard/src/app) — layout scoping for the authed shell; `(status-page)/[domain]/[locale]` in status-page nests tenant → locale.
- The list page splits `page.tsx` (server, params parsing) from `client.tsx` (interactivity) — see [monitors/(list)/](../../../apps/dashboard/src/app/(dashboard)/monitors/(list)) with `search-params.ts` shared between them (nuqs parsers used at [client.tsx:47](../../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L47)).
- Middleware as an access-control plane: the status-page [proxy.ts](../../../apps/status-page/src/proxy.ts#L15-L96) runs before any rendering — rewrite/redirect decisions never reach React.

Sharp edges:
- URL state (nuqs) vs component state vs server cache are three different state stores on one page ([client.tsx:47-49](../../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L47-L49) uses all three). Choosing wrong = lost state on refresh or stale UI.
- The `useEffect` at [client.tsx:82-90](../../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L82-L90) syncs URL → table state *once on mount* with suppressed exhaustive-deps and a TODO admitting the layout shift. Real-world compromise; know how you'd remove it (server-side `searchParamsCache`).

## React rendering: render vs commit, and query-driven UI

Mental model: render = compute the tree (pure, can be thrown away); commit = apply to DOM; effects run after commit. With TanStack Query, most "state management" becomes **server-cache management**: `data` is a cache read keyed by query key; mutations invalidate keys.

Where the repo uses it well:
- One query client per browser session, recreated per request on the server ([query-client via client.tsx:14-37](../../../apps/dashboard/src/lib/trpc/client.tsx#L14-L37)) — the comment explains the suspense-during-init reason. `staleTime: 60s` avoids refetch storms after SSR.
- Conditional dependent queries: metrics queries are `enabled` only when monitor ids exist ([client.tsx:64-78](../../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L64-L78)) — the standard "waterfall on purpose" pattern.

Sharp edges: `if (!monitors) return null` (L92) conflates loading/error/empty — tickets J4/J15 exist because of it. Derived arrays (`monitorsByType`, L51–60) recompute every render; fine here, `useMemo` when it feeds memoized children.

## tRPC: RPC with inferred contracts

Mental model: routers are plain functions; the *type* of the router flows to the client (`createTRPCContext<AppRouter>()`, [client.tsx:11-12](../../../apps/dashboard/src/lib/trpc/client.tsx#L11-L12)); there is no wire schema to maintain — which is exactly why it's for first-party clients only.

The middleware chain is the interesting part: `protectedProcedure` = session check → workspace resolution from cookie → context enrichment → post-response analytics via `after()` ([trpc.ts:148-241](../../../packages/api/src/trpc.ts#L148-L241)). Meta-driven event tracking (`.meta({ track: Events.DeleteMonitor })`, [monitor.ts:79](../../../packages/api/src/router/monitor.ts#L79)) keeps instrumentation declarative.

## Hono: the fetch-standard microframework

Mental model: `app.use`/`app.route` over the WHATWG fetch types; runs identically on Bun/Node/Workers. Middleware composes via `await next()` — which enables the wide-event pattern (start event → `await next()` → finalize + emit) at [server index.ts:109-179](../../../apps/server/src/index.ts#L109-L179).

Two Hono flavors here: plain Hono (workflows) and `OpenAPIHono` ([v1/index.ts:25-27](../../../apps/server/src/routes/v1/index.ts#L25-L27)) where zod schemas *are* the OpenAPI spec (`createRoute` at [post.ts:15-42](../../../apps/server/src/routes/v1/monitors/post.ts#L15-L42)) — contract-first without writing YAML.

## Drills

1. In the monitors list page, classify each piece of state (URL / React / server-cache) and say what survives a refresh. Verify against [client.tsx:43-97](../../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L43-L97).
2. Sketch the middleware onion for a `POST /v1/monitor` request (requestId → sentry → wide event → auth → scope → route) from [server index.ts](../../../apps/server/src/index.ts#L105-L179) + [v1/index.ts](../../../apps/server/src/routes/v1/index.ts#L142-L161).
3. Predict what happens to the monitors list UI when a delete mutation succeeds but no invalidation runs. Then find how the repo invalidates (search `invalidate` under `apps/dashboard/src/components/data-table/monitors/` — unverified here on purpose; go look).

## Interview angle

- "Server vs client components" / "where does data fetching live" → [08/02 Q1–Q4](../08-interview-prep/02-frontend-framework-questions.md).
- "How does tRPC give type safety, what are its limits" → 08/02 Q6.
- "Design an HTTP middleware stack" → 08/03 Q8; the wide-event onion is your concrete example.
