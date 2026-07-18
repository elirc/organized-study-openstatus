# 04 — Derived Data, Read Models, and Caching

The hardest part of caching isn't caching — it's *invalidation*. Phil Karlton's famous quote: "There are only two hard things in Computer Science: cache invalidation and naming things." This file is about how OpenStatus handles both.

## Derived data: compute, don't store

If you can derive a value from existing data, compute it. Storing it creates a synchronization burden: every write to the source must update the derived value. Forget once, and you have inconsistency forever.

OpenStatus encodes this well. The public status page status (operational/degraded/down) isn't a column in any table — it's computed from monitors, incidents, and reports at query time ([`packages/api/src/router/statusPage.ts:136-180`](../../packages/api/src/router/statusPage.ts#L136-L180)). The `hasUnresolvedIssue` flag isn't stored — it's `.some(r => r.status !== "resolved")` at render time.

The cost: every read recomputes. For most things, that cost is acceptable. When it isn't, you cache the *result*, not the intermediates — and invalidate on the events that change the inputs.

## Read models — the senior pattern

A read model is a *view* of data optimized for a specific consumer. It's different from your write tables. Examples:

- **Public status page** — reads from `page`, `pageComponent`, `monitor`, `incident`, `statusReport`, `maintenance`, `statusReportUpdate` and produces a single `PublicPage` object with computed fields (`openEvents`, `lastEvents`, `trackers`, `pageStatus`). [`packages/api/src/router/statusPage.ts:65-400`](../../packages/api/src/router/statusPage.ts#L65-L400).
- **Dashboard status report list** — reads from `statusReport`, enriches with `statusReportUpdate`, `pageComponent`, `page`. [`packages/services/src/status-report/list.ts:66-207`](../../packages/services/src/status-report/list.ts#L66-L207).

Two patterns to recognize:

**1. Server-side computation, ship clean data.** The public status page does all the event computation server-side and ships a fully-formed object the client just renders. The client doesn't run sort, filter, or derive — it maps to JSX.

**2. Wide joins in one query, not N+1.** `enrichReportsBatch` loads updates + components + pages in three batched queries, not N+1.

When you build a new endpoint:
- Sketch the consumer's needs first. What does the UI render?
- Build the read model to *match* those needs, not the write tables' shape.
- One query (or a few parallel ones) with all the data, not N+1.

## When the query is too slow

You'll hit cases where a read model is genuinely expensive. The fix order:

1. **Index**. Most "slow query" problems are missing-index problems.
2. **Reduce columns**. `.select({ id, title })` is faster than `select()` if the table has wide rows.
3. **Reduce joins**. Sometimes you don't need *every* relation; lazy-load expensive ones on demand.
4. **Cache the result**. React Query on the client; a Redis layer on the server for shared computation.
5. **Materialize**. Pre-compute and store the derived data; update via triggers or background jobs.

Step 5 is last. Materializing introduces all the consistency problems we said to avoid. Only do it when steps 1–4 aren't enough.

The public `statusPage.get` has a TODO ([`packages/api/src/router/statusPage.ts:49-55`](../../packages/api/src/router/statusPage.ts#L49-L55)) about SQL performance. Currently it loads broad relations and computes in memory. A senior optimizing this would profile first, then likely split the query (shell vs events vs trackers), then cache the shell at the CDN level if traffic warrants.

## Cache invalidation strategies

Three strategies in increasing strength:

### 1. TTL (time-to-live)

"Cache for 60s, then refetch." Simple, fits most cases. React Query's `staleTime` is this.

Pros: dead simple, no coordination needed.
Cons: window of staleness on every change.

OpenStatus uses 60s `staleTime` ([`apps/dashboard/src/lib/trpc/client.tsx:14-23`](../../apps/dashboard/src/lib/trpc/client.tsx#L14-L23)) for client-side caches.

### 2. Explicit invalidation by key

"After a mutation, invalidate cache key X." Surgical but requires knowing which keys are affected.

Example: after creating a status report, invalidate `["page", "list", ...]` because the page-list summary might change.

```ts
queryClient.invalidateQueries({ queryKey: trpc.page.list.queryKey() });
```

This is the pattern OpenStatus uses heavily on the client. The cost: the developer has to know what to invalidate. Miss one and you get stale UI.

### 3. Event-driven (pub/sub)

"On mutation X, publish event Y; subscribers update their caches." Used at scale (Redis pub/sub, WebSockets) when many clients care.

OpenStatus doesn't need this — single-user dashboards. A monitoring SaaS at scale (with team dashboards updated in real time) would.

## The staleness-correctness tradeoff

Every cache is a deliberate decision to tolerate staleness in exchange for performance. Make the decision explicitly:

- "60s stale is fine for the page list — the user adds a page and refreshes the page they're on, which fetches fresh."
- "0s stale is required for the audit log — every entry must be visible immediately."
- "Infinity stale is fine for the workspace plan — it never changes mid-session, and we invalidate explicitly on plan-change events."

OpenStatus tends toward fresh reads with explicit invalidation on mutation. That's a defensible default for a control-room product where stale data is alarming.

## Optimistic updates — pretend the mutation succeeded

For UX, sometimes you want the UI to update immediately without waiting for the server. Pattern:

```ts
const mutation = useMutation({
  mutationFn: api.deleteReport,
  onMutate: async ({ id }) => {
    await queryClient.cancelQueries({ queryKey });
    const prev = queryClient.getQueryData(queryKey);
    queryClient.setQueryData(queryKey, (old) => old.filter((r) => r.id !== id));
    return { prev };
  },
  onError: (err, _vars, ctx) => queryClient.setQueryData(queryKey, ctx.prev),
  onSettled: () => queryClient.invalidateQueries({ queryKey }),
});
```

OpenStatus doesn't use this widely — mutations are fast enough that the spinner is brief. For slower mutations or higher-frequency interactions (drag-and-drop reordering), optimistic updates would pay off.

The cost of optimistic updates: every mutation needs its own `onMutate` rollback. That's not free — adds code and bug surface. Use only when the UX win is meaningful.

## Cache shape: structural sharing

React Query uses **structural sharing**: if a refetch returns data that's structurally equal to the cached value, it keeps the *same reference*. That matters because:

- React's `===` re-render check sees no change.
- `useMemo` deps with the data object don't re-fire.
- Selectors and derived values don't recompute.

If you transform the data inside `useQuery`'s `select` function, structural sharing applies to the selected slice too. This is why writing transforms in `select` (or in render with memoization) is preferred over storing the transformed data in another state.

## Server-side caching with Next.js

Next.js has its own caches:

- **Request memoization** (`cache()` from `react`): dedupes calls within a single render.
- **Data cache**: caches `fetch` responses across requests (controlled by `revalidate`, `cache: "no-store"`).
- **Full route cache**: static pages cached at build.
- **Router cache**: client-side cache of route segments.

OpenStatus's dashboard is dynamic (per-user, per-request), so the data cache is mostly bypassed (`credentials: "include"` + cookies → not cacheable). The status page is also dynamic. The web site (marketing) can use the full route cache aggressively.

If you're optimizing a Next route, walk through which caches apply. Misusing one for sensitive data leaks across users.

## Cache keys you can reason about

A query key is a serialization of *what data is being requested*. If two callers should see the same data, their keys must match exactly. If they shouldn't (different workspace, different filter), keys must differ.

OpenStatus's tRPC integration produces keys like `["statusReport", "list", { pageId: 7 }]`. The workspace is *implicit* (server-side from the session) — it doesn't appear in the key. That's fine because users only have one active workspace at a time.

If you ever needed to support a user switching workspaces in the same React Query client (rare), you'd want workspace id in the key to prevent cache poisoning across workspaces.

## Pitfall checklist

1. **Derived data stored as a column** that's not kept in sync. Lock-in for life.
2. **N+1 in a list.** Batch with `inArray` + map.
3. **Missing `invalidateQueries`** after a mutation. UI shows stale data.
4. **Invalidating too aggressively.** Every mutation invalidates everything → no caching benefit.
5. **`staleTime: Infinity`** on data that does change. Forever stale, manual cache-busting forever.
6. **Cache keys with unstable values** (`Date.now()`, `Math.random()`). Every render = new key.
7. **Optimistic update without rollback.** Wrong UI after failure.
8. **Server-side cache leaking across users.** Sensitive data shown to wrong session.
9. **Materializing prematurely.** "We might need it" turns into "we maintain a denormalized table forever."
10. **Caching a slow query without measuring.** You can't know what cache to add without knowing where time is actually spent.

## Drills

- Find every `invalidateQueries` in [`apps/dashboard/src/`](../../apps/dashboard/src/). For each, ask: is this the minimum set of keys? Could it be tighter?
- Compute the round-trip count for `statusPage.get`. (Hint: `db.query.page.findFirst({ with: {...} })` is one query with joins; the parsing and post-processing add no DB hits.)
- Sketch a materialized table for "the latest status of each monitor." What columns? What updates it (trigger, app code, background job)? Defend or attack the design.
- Pick a mutation. Walk through every place the changed data could appear (dashboard list, sidebar, public page, search results). Confirm each is either invalidated or out of scope (different cache).
- Read the dashboard's status reports `onSuccess` ([`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:36-51`](../../apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx#L36-L51)). What does it invalidate? What does it miss? (Hint: think about other tabs in another browser window.)
