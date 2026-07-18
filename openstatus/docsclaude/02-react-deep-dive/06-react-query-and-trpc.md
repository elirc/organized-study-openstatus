# 06 — React Query and tRPC (Actual Mental Model)

These two libraries are doing the heavy lifting in the OpenStatus dashboard. Most "magic" reduces to two ideas: **the query cache is a keyed snapshot store** and **tRPC types travel from server to client via inference**. Once you see them clearly, you can debug them.

## The React Query mental model

There is one cache per app. The cache is a `Map<QueryKey, QueryState>` where `QueryKey` is a JSON-stable array (e.g., `["statusReport", "list", { pageId: 7 }]`) and `QueryState` includes `{ data, status, error, dataUpdatedAt, ... }`.

A **query** is an observer subscribed to a key. When you call `useQuery({ queryKey, queryFn })`:

1. Compute the key.
2. Look up cache entry.
3. If exists & fresh: return cached `data`, status `"success"`.
4. If exists & stale: return cached `data` (instant), trigger a background refetch.
5. If missing: status `"pending"`, run `queryFn`, store result.

That's it. Everything else (deduplication, retries, structural sharing, focus refetch, mutation invalidation) is built on top.

## `staleTime` vs `gcTime` (formerly `cacheTime`)

- **`staleTime`** — how long after a fetch the data is considered "fresh." Within this window, mounting a new consumer of the same key uses the cache without a refetch.
- **`gcTime`** — how long unused (no observers) cache entries are kept before being garbage-collected.

OpenStatus sets `staleTime: 60 * 1000` ([`apps/dashboard/src/lib/trpc/client.tsx:14-23`](../../apps/dashboard/src/lib/trpc/client.tsx#L14-L23)):

```ts
function makeQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000,   // 60s — match SSR hydration expectations
      },
    },
  });
}
```

Why? With SSR + hydration, you don't want the client to *immediately* refetch the same data the server just rendered. `staleTime: 60s` keeps it fresh long enough that the user's first interaction doesn't trigger a redundant fetch.

**Senior aside:** *Most real-world bugs in React Query come from misjudging `staleTime`. Too low and you hammer your API. Too high and you show stale data after a mutation that didn't properly invalidate. The fix is rarely `staleTime`; it's correct invalidation.*

## Query keys — naming things hard

A query key uniquely identifies a "logical query." Conventions:

```ts
["statusReport", "list", { pageId: 7 }]
["statusReport", "get",  { id: 42 }]
["page", "list"]
```

The tRPC integration generates these for you. `trpc.statusReport.list.queryOptions({ pageId: 7 })` returns `{ queryKey: ["statusReport", "list", { pageId: 7 }], queryFn: ... }`. You don't usually write keys by hand.

You invalidate by **prefix**:

```ts
queryClient.invalidateQueries({ queryKey: trpc.page.list.queryKey() });
// Invalidates anything whose key starts with ["page", "list", ...]
```

That's why the create flow can say "after creating a status report, also invalidate the page list" with one line ([`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:46-49`](../../apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx#L46-L49)):

```tsx
refetch();
queryClient.invalidateQueries({ queryKey: trpc.page.list.queryKey() });
```

`refetch()` is bound to *this* query (the `list` for this `pageId`). The `invalidateQueries` is broader — every mounted consumer of any page-list cache refetches.

## Mutations

```ts
const mutation = useMutation({
  mutationFn: async (input) => { ... },
  onSuccess: (data, variables) => { ... },
  onError:   (err,  variables) => { ... },
});
```

- `mutate(input)` — fire-and-forget; results handled in callbacks.
- `mutateAsync(input)` — returns a promise; await it inline.

Optimistic updates:

```ts
onMutate: async (newItem) => {
  await queryClient.cancelQueries({ queryKey });
  const prev = queryClient.getQueryData(queryKey);
  queryClient.setQueryData(queryKey, (old) => [...old, newItem]);
  return { prev };
},
onError: (err, _vars, ctx) => queryClient.setQueryData(queryKey, ctx.prev),
onSettled: () => queryClient.invalidateQueries({ queryKey }),
```

Cancel in-flight queries, snapshot the old data, write the optimistic value, restore on error, refetch on settle. OpenStatus doesn't use optimistic updates heavily — the mutations are fast enough — but you should know the pattern.

## SSR hydration with React Query

The server-side proxy ([`apps/dashboard/src/lib/trpc/server.tsx:23-57`](../../apps/dashboard/src/lib/trpc/server.tsx#L23-L57)) creates a *per-request* query client (`getQueryClient = cache(makeQueryClient)`). React's `cache()` ensures the same client is reused across the same request — so a layout and a page in the same render share one cache.

Server components prefetch into this cache. Then `HydrateClient` ([`apps/dashboard/src/lib/trpc/server.tsx:59-67`](../../apps/dashboard/src/lib/trpc/server.tsx#L59-L67)) serializes the cache and ships it to the client:

```tsx
export function HydrateClient(props) {
  const queryClient = getQueryClient();
  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      {props.children}
    </HydrationBoundary>
  );
}
```

On the browser, the matching `QueryClient` (created in `client.tsx`) absorbs the dehydrated state. Client `useQuery` calls find cache hits immediately. The user sees fully populated UI on first paint.

The pattern of `fetchQueryOrNotFound` ([`apps/dashboard/src/lib/trpc/server.tsx:103-117`](../../apps/dashboard/src/lib/trpc/server.tsx#L103-L117)) is layered on top:

```ts
export async function fetchQueryOrNotFound<...>(queryOptions) {
  const queryClient = getQueryClient();
  try {
    return await queryClient.fetchQuery(queryOptions);
  } catch (error) {
    if (error instanceof TRPCClientError && error.data?.code === "NOT_FOUND") {
      notFound();   // Next.js renders the not-found page
    }
    throw error;
  }
}
```

Use this in layouts when the resource must exist for the children to make sense. It both populates the cache and aborts rendering with a 404 if the resource is missing.

## tRPC — the typed pipe

tRPC takes a server-side router definition and exposes it as a typed client.

Server-side, you compose procedures into a router (see [`packages/api/src/edge.ts:25-49`](../../packages/api/src/edge.ts#L25-L49)). The router's *type* (`AppRouter`) is exported:

```ts
import type { AppRouter } from "@openstatus/api";
```

Client-side, you create a typed proxy:

```ts
export const { TRPCProvider, useTRPC, useTRPCClient } =
  createTRPCContext<AppRouter>();
```

`useTRPC()` returns an object whose shape mirrors the router. Every leaf has helpers — `queryOptions`, `mutationOptions`, `queryKey`, `mutate` (via vanilla client), etc. — all typed exactly to the procedure's input/output.

No code generation, no schema files, no manual updating. You change the server, the client types change on save.

## How tRPC routes split — edge vs lambda

[`apps/dashboard/src/lib/trpc/shared.ts:38-77`](../../apps/dashboard/src/lib/trpc/shared.ts#L38-L77) implements a custom link:

```ts
const lambdas = ["stripeRouter", "emailRouter", "apiKeyRouter", "integrationRouter", "blob"];
// ...
return (ctx) => {
  const path = ctx.op.path.split(".") as [string, ...string[]];
  const endpoint = lambdas.includes(path[0]) ? "lambda" : "edge";
  return endpoint === "edge" ? edgeLink(newCtx) : lambdaLink(newCtx);
};
```

When you call `trpc.statusReport.create.mutate(...)`, the procedure path is `statusReport.create`. The first segment, `statusReport`, isn't in `lambdas`, so the request goes to `/api/trpc/edge`. When you call `trpc.emailRouter.sendStatusReport.mutate(...)`, the first segment `emailRouter` is in `lambdas`, so it goes to `/api/trpc/lambda`.

This split exists because edge runtime can't run Stripe or Resend SDKs (Node-only). The hardcoded list is a fragility — adding a new Node-only router means *two* edits (lambda router + this array). Keep them in sync.

## Errors travel typed too

A procedure throws a `TRPCError({ code: "NOT_FOUND" })`. The client receives a `TRPCClientError` with `.data.code === "NOT_FOUND"`. You can branch on it:

```ts
if (isTRPCClientError(err) && err.data?.code === "CONFLICT") {
  form.setError("title", { message: err.message });
}
```

OpenStatus's adapter ([`packages/api/src/service-adapter.ts:32-87`](../../packages/api/src/service-adapter.ts#L32-L87)) maps service-layer errors (`NotFoundError`, `ForbiddenError`, `ConflictError`, ...) to the right tRPC error codes:

```ts
case "NOT_FOUND":  throw new TRPCError({ code: "NOT_FOUND", message: err.message });
case "FORBIDDEN":  throw new TRPCError({ code: "FORBIDDEN", message: err.message });
case "CONFLICT":   throw new TRPCError({ code: "CONFLICT", message: err.message });
case "LIMIT_EXCEEDED": throw new TRPCError({ code: "TOO_MANY_REQUESTS", message: err.message });
// ...
```

The codes are not arbitrary — they map to HTTP status codes, which downstream observability tools categorize. `TOO_MANY_REQUESTS` (429) for quota exhaustion is semantically right; `FORBIDDEN` (403) would be misleading.

## Common patterns you'll see in this repo

### `enabled` to gate queries

```ts
const { data } = useQuery({
  ...trpc.tinybird.globalMetrics.queryOptions({ monitorIds, type: "http" }),
  enabled: monitorIds.length > 0,
});
```

The query doesn't fire until the condition is true. Used heavily in [`apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx:64-78`](../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L64-L78) so HTTP and TCP metric queries don't fire when there are no monitors of that type.

### `refetch()` + `invalidateQueries` after mutation

The pattern from the status-report page is canonical. Use it whenever a mutation might affect data shown elsewhere.

### Hydration boundary at the layout

Prefetch in the layout, hydrate down, *then* render the page. The page calls `useQuery` and finds the cache populated. No double-fetch.

## Pitfall checklist

1. **Unstable query keys.** A `new Date()` or `Math.random()` in the key — every render is a new query.
2. **Reaching for `useEffect` to fetch.** That's React Query's job; let it do it.
3. **Optimistic updates without `onError` revert.** UI shows wrong data after failure.
4. **Forgotten invalidation.** Mutation succeeds, the list shows stale data, the user thinks the create failed.
5. **`staleTime: Infinity`** on data that does change. Manual cache-busting forever.
6. **`enabled: condition && query`** — boolean coercion gotcha; use `enabled: !!condition` or `Boolean(condition)`.
7. **Calling `useQuery` conditionally.** Hook rules apply; use `enabled` instead.
8. **Subscribing to a query in 10 places when one place would do.** Move the hook to a shared parent and pass data down — or accept the duplication if it actually helps colocation. React Query dedupes the *fetch*, but each hook is still its own subscription.
9. **Mutations that don't return new data.** If `mutationFn` returns `void`, `onSuccess` has no payload to write into the cache; you'll have to refetch.
10. **Mixing `mutate` and `mutateAsync` semantics.** `mutate` doesn't return a promise; awaiting it does nothing.

## Drills

- Open the React Query DevTools (it's typically wired into the dev build). Watch the cache as you navigate.
- After clicking Create Status Report, identify every cache key that should be invalidated. Compare with what the code actually invalidates ([`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:46-49`](../../apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx#L46-L49)). What did the author miss, if anything?
- Inspect the network tab. After navigating to a status reports page that was prefetched server-side, do you see a refetch on mount? Why or why not? (Hint: `staleTime`.)
- Add a `staleTime: 0` override to one query. Click around. Notice the difference.
- Read [`apps/dashboard/src/lib/trpc/shared.ts:38-77`](../../apps/dashboard/src/lib/trpc/shared.ts#L38-L77). Add a fake router name to `lambdas` and predict what happens when you call any procedure with that prefix. What error?
