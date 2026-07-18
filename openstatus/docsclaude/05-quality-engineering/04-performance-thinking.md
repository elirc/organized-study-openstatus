# 04 — Performance Thinking

Performance work usually fails because the engineer optimizes the wrong thing. The fix is: **measure first, then act**. This file gives you the toolkit to measure honestly and act on the right targets.

## The three perf domains in this stack

1. **Bundle size** — how much JS hits the browser. Affects first-paint and time-to-interactive.
2. **Render performance** — how long React takes to paint. Affects responsiveness during interaction.
3. **Backend performance** — how long the server takes to respond. Affects time-to-data and throughput.

Each has different tools. Each has different bug shapes. Don't confuse them.

## Backend: measure with logs and query plans

The simplest, fastest signal: log timings around suspect blocks.

```ts
const t0 = performance.now();
const result = await loadEverything();
console.log("[load] ms=", performance.now() - t0);
```

For a procedure, OpenStatus has the post-success `after()` hook ([`packages/api/src/trpc.ts:213-238`](../../packages/api/src/trpc.ts#L213-L238)) — you can add timing to it.

For deeper analysis:

- **Drizzle query log** — turn it on (`drizzle({ ..., logger: true })`). Every query logs with timing.
- **Turso/SQLite EXPLAIN** — `EXPLAIN QUERY PLAN SELECT ...` shows whether your query uses an index or does a full scan. If you see `SCAN`, you're missing an index.
- **Slow query log** — production tooling (Vercel logs, Datadog, etc.) usually captures slow queries.

### Finding N+1

Two ways:

1. Read the code. Any `for ... of ... await db.select()` is an N+1.
2. Look at logs. A request that hits the DB 100 times for a 100-row list is N+1.

OpenStatus avoids N+1 in [`enrichReportsBatch`](../../packages/services/src/status-report/list.ts#L72-L162) by batching. Use this pattern when you find one.

### Async serialization

Two await statements that don't depend on each other should run in parallel:

```ts
// ❌ Sequential, 200ms total
const a = await fetchA();   // 100ms
const b = await fetchB();   // 100ms

// ✅ Parallel, 100ms total
const [a, b] = await Promise.all([fetchA(), fetchB()]);
```

OpenStatus does this in [`packages/services/src/status-report/list.ts:184-202`](../../packages/services/src/status-report/list.ts#L184-L202) for count + rows queries. Look for opportunities elsewhere; they're common.

### Round-trips are the enemy

Each DB round-trip is a fixed cost (network + parse + queue). 10 cheap queries can be slower than 1 expensive one if each costs 5ms of overhead. Prefer:

- Joins over multiple selects when relations are tightly coupled.
- `inArray()` over multiple queries by id.
- `Promise.all` over serial awaits.
- A single tRPC procedure returning a composed payload, over multiple procedures the client orchestrates.

## React: measure with the Profiler

React DevTools' Profiler is the answer to "why is this slow."

1. Open DevTools → Profiler tab.
2. Click "Record."
3. Reproduce the slow interaction.
4. Click "Stop."
5. The flame chart shows each render with its time. Click a render to see which components rendered and why.

Look for:

- **Long renders** (> 16ms — that's one frame). Worth optimizing.
- **Cascades** — one state change causing many components to render. Often a context update or a parent re-render that should have been blocked by memo.
- **Repeated renders** — same component rendering 10 times in a second. Usually a state-update loop.

### Common React perf bugs

| Symptom | Cause | Fix |
| --- | --- | --- |
| List slow on scroll | Re-rendering every row | Memoize rows; virtualize long lists |
| Form slow on each keystroke | Top-level component watches all fields | Subscribe per-field; rhf handles this if you let it |
| Page slow on first render | Heavy compute in render | `useMemo`, or move to server component |
| App slow after a mutation | Cache invalidation refetches the world | Tighten invalidation; use `setQueryData` to patch |
| Hover causes re-render of unrelated components | Hover state lifted too high | Push state down |

### When to memoize

`useMemo`/`useCallback`/`React.memo` are tools, not defaults. Reach for them when:

- A child is wrapped in `React.memo` and you're passing it a fresh object/function each render.
- A `useMemo` value is expensive to compute and the deps stable.
- A `useCallback` is in the dep array of a `useEffect` and you don't want the effect to re-run.

Don't memoize:
- Cheap computations. The memo overhead is more than the saved work.
- Values whose identity doesn't matter. Memoizing for the sake of it adds noise.

If you can't articulate the concrete benefit, don't memoize.

## Bundle: measure with `@next/bundle-analyzer` or similar

```sh
ANALYZE=true pnpm build
```

You'll get an HTML report showing each chunk's contents. Look for:

- Unexpectedly large dependencies (date libraries, icon packs, lodash).
- Server-only code in client bundles (DB drivers, secrets — alarm bell).
- Duplicate dependencies (two versions of React, two versions of date-fns).

Reducing bundle size:

- **Dynamic imports** for large rarely-used components: `dynamic(() => import("./Heavy"), { ssr: false })`.
- **Tree shake** — make sure your imports are named (`import { format } from "date-fns"`, not `import * as df from "date-fns"`).
- **Replace heavy libs.** `date-fns/format` is lighter than full `moment.js`. `lucide-react`'s per-icon import is lighter than the bundle.

OpenStatus's edge bundle has a tight budget (Vercel Edge limit). Adding a heavy dep to an edge router can break deploys.

## Network: measure with the Waterfall

DevTools → Network → look at the waterfall. Each row is a request; the bar shows queue/send/wait/receive time.

Look for:

- **Long Time-To-First-Byte (TTFB)** — the server is slow. Backend perf, not client.
- **Sequential requests that should be parallel** — chained `await` in a hook somewhere.
- **Many small requests** — could be batched. tRPC's `httpBatchLink` solves this for tRPC.
- **Slow third-party scripts** blocking the main thread.

## CDN and caching

For data that doesn't change per-user:

- **Static assets** (CSS, JS bundles) — automatically CDN-cached by Vercel.
- **Public status page HTML** — could be CDN-cached for short TTLs if the team wanted. Currently dynamic.
- **API responses** — can use `Cache-Control` headers if the data is shareable across users.

OpenStatus's dashboard is per-user (cookies → not cacheable). The status page is per-domain. Adding caching headers to public status pages would dramatically reduce server load — at the cost of staleness.

## Time-to-Interactive (TTI) — the user-visible metric

TTI is "when can the user click and see a response." It's a function of:

- Time to first byte (server + network).
- Time to download HTML and CSS.
- Time to download and parse JS.
- Time to hydrate React.

The biggest lever is usually **bundle size**, because parse+hydrate dominate on slow devices. After that, **server response time**.

For OpenStatus's dashboard, the SSR prefetch pattern (server prefetches React Query cache, client hydrates) makes first-data-paint very fast — the user doesn't see a spinner while the client refetches.

## The 80/20 of perf work

The single most impactful perf intervention I've ever seen on a React app:

1. Move state down. (Re-renders the world → re-renders one component.)
2. Add an index to the slow query. (10s → 30ms.)
3. Parallelize independent awaits. (2x to 5x speedup for free.)

Almost never:
1. Replace React with a faster framework.
2. Hand-write JSX optimizations.
3. Micro-benchmark utility functions.

Do the cheap, obvious things first. Profile before going exotic.

## What "fast enough" means

User-perceived performance research:

- **< 100ms**: feels instant.
- **100ms – 1s**: feels responsive. Acceptable for most actions.
- **1s – 10s**: feels slow but tolerable with a spinner.
- **> 10s**: users assume it's broken.

Aim for < 100ms for clicks, < 1s for navigations, < 3s for page loads. Mutations that take longer should show progress.

Don't optimize past "fast enough." Engineering time spent shaving 200ms off a 50ms request is wasted.

## Pitfall checklist

1. **Optimizing without measuring.** "I think this is slow" → measure. Often you're wrong.
2. **Memoizing everything.** Adds overhead, no benefit.
3. **Premature caching.** Cache invalidation is the second-hardest problem in CS for a reason.
4. **Optimizing the wrong layer.** Frontend lag is sometimes a backend problem.
5. **Bundling server-only code into the client.** Hugely bloats the bundle.
6. **Heavy date libraries** (`moment.js`) when `date-fns` or native APIs suffice.
7. **Loading the entire icon set** instead of importing per-icon.
8. **Synchronous expensive work** in render. Move to `useMemo` or compute server-side.
9. **`Math.random()` / `Date.now()` in cache keys.** No cache, ever.
10. **Premature concurrency.** `Promise.all` over independent work — yes. `Promise.all` over chained dependencies — bugs.

## Drills

- Profile loading the status reports page. Identify the longest render. Predict why.
- Enable Drizzle's query logger. Run a public status page request. Count the queries. Estimate the speedup of batching one of them.
- Run the bundle analyzer on the dashboard. Find the three largest chunks. Are they all justified?
- Find a `useEffect` doing async work. Replace it with React Query if possible. Compare with the original via Profiler.
- Take a real Drizzle query in this repo. Generate the SQL (`.toSQL()`). Run `EXPLAIN QUERY PLAN` on it. Read the plan. Is every step indexed?
