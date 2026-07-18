# Language & Runtime Model

## The event loop, concretely

Mental model: one call stack; `await` parks the current function and returns control to the loop; microtasks (promise continuations) drain before the next macrotask (timers, I/O callbacks). A "single-threaded" server is concurrent because **handlers interleave at every await**.

Where this repo exercises it well:
- Bounded fan-out instead of naive `Promise.all`: `Effect.forEach(taskInputs, …, { concurrency: 100 })` at [cron/checker.ts:185-211](../../../apps/workflows/src/cron/checker.ts#L185-L211). An unbounded map over thousands of monitors×regions would spike memory and hammer GCP.
- `Promise.allSettled` for independent user workflows at [cron/monitor.ts:146-153](../../../apps/workflows/src/cron/monitor.ts#L146-L153) — one user's failure must not abort the batch; results are then counted for the summary log.
- Rate limiting inside a loop: `await limiter.removeTokens(1)` at [cron/monitor.ts:140-144](../../../apps/workflows/src/cron/monitor.ts#L140-L144) — an await used purely as backpressure.

Sharp edge (memorize this): interleaving-at-await means read-check-write sequences on shared state race *without threads*. The `/updateStatus` handler reads `monitor.status`, awaits several times, then writes it ([checker/index.ts:128-298](../../../apps/workflows/src/checker/index.ts#L128-L298)). Two requests interleave; both pass the same guard. Single-threadedness protects statements, not sequences.

Pitfall checklist:
- [ ] Any `for` loop of awaits that could be `allSettled`/bounded-concurrent (or must stay serial — say why)
- [ ] Any `Promise.all` where one rejection should not cancel siblings
- [ ] Any read-modify-write across an await on shared/durable state
- [ ] Fire-and-forget promises (unhandled rejection risk) — the repo's tRPC analytics uses Next's `after()` to be explicit about post-response work ([trpc.ts:213-238](../../../packages/api/src/trpc.ts#L213-L238))

## The five runtimes

| Runtime | Here | Constraint that bites |
| --- | --- | --- |
| Browser | client components ([client.tsx](../../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L1)) | no secrets, no Node APIs |
| Node (Next server) | RSC, NextAuth, route handlers | server-only code imported into client crashes builds |
| **Edge** | dashboard tRPC edge router | no `node:*` — the reason `deepEqual` is hand-rolled ([emit.ts:14-24](../../../packages/services/src/audit/emit.ts#L14-L24)) |
| Bun | apps/server, apps/workflows | top-level await OK ([index.ts:64](../../../apps/server/src/index.ts#L64)); `bun:test`; fetch-handler export shape ([index.ts:273-275](../../../apps/server/src/index.ts#L273-L275)) |
| Go | apps/checker | contract with JS is JSON only; types duplicated on both sides |

Transferable rule: **a package's runtime is the intersection of every environment that imports it.** `packages/services` runs on Edge and Bun → lowest common denominator wins. That's an architectural constraint expressed as an import discipline.

## Closures and long-lived state

The Go probe op ([checker.go:111-274](../../../apps/checker/handlers/checker.go#L111-L274)) and the JS retry closures both capture mutable outer variables (`called`, `result`). Same concept as JS closures/stale state in React (see [02-framework-mental-models.md](02-framework-mental-models.md)). Being able to point at a closure in Go and a stale closure in React and say "same mechanism" is a strong interview move.

## Drills

1. Rewrite (on paper) [cron/monitor.ts:140-146](../../../apps/workflows/src/cron/monitor.ts#L140-L146) without the rate limiter using batched `allSettled` chunks — then argue which is better for the Resend API and why the original picked token-bucket.
2. Find every `await` between the `monitor.status` read and write in [checker/index.ts](../../../apps/workflows/src/checker/index.ts#L128-L298). Count the interleaving windows.
3. Explain to a rubber duck why `withTestTransaction`'s `let result!: T` needs the definite-assignment assertion ([helpers.ts:39-50](../../../packages/services/test/helpers.ts#L39-L50)).

## Interview angle

- "Explain the event loop / microtasks" → answer with the interleaving-race example (Q7 in [08/01](../08-interview-prep/01-js-ts-node-deep-dive.md)).
- "Serial vs parallel awaits" → Q1.
- "How do you rate-limit or bound concurrency?" → Q11.
- "What's different about Edge runtimes?" → Q9.
