# JS / TS / Node Deep-Dive Question Cards

Fourteen cards. Each anchors to a place this repo genuinely exercises the concept — answer with the anchor, not a textbook definition. Format per the Interview Question Card template.

---

## Q1: Walk me through what the event loop does when a request handler awaits three sequential DB calls. How would you parallelize, and when shouldn't you?

Round: JS/TS deep-dive
What it's really testing: async mental model; serial vs parallel judgment.
Repo anchor: [checker/index.ts:122-148](../../../apps/workflows/src/checker/index.ts#L122-L148) — `/updateStatus` awaits upsert → monitor fetch → affected-regions query serially; the fetch depends on nothing but runs second anyway.
Junior answer sounds like: "await pauses the function; use Promise.all to go faster."
Mid-level adds: awaits yield to the loop so the server keeps serving; only *independent* calls parallelize — here the upsert must land before counting affected regions, but the monitor fetch could overlap with it.
Senior includes: parallelism vs correctness under read-your-writes (the count must see the upsert — over HTTP-libsql these are separate requests, so ordering is the only guarantee); also DB connection pressure — `Promise.all` over 100 items needs a concurrency cap (this repo uses `Effect.forEach(…, { concurrency: 100 })` at [cron/checker.ts:185-211](../../../apps/workflows/src/cron/checker.ts#L185-L211)).
Likely follow-ups: microtask vs macrotask ordering; what happens to errors in `Promise.all` (one rejection wins) vs `allSettled` ([cron/monitor.ts:146](../../../apps/workflows/src/cron/monitor.ts#L146) uses `allSettled` — say why: one user's failure shouldn't kill the batch).
Practice drill: 90 seconds aloud, tracing the anchor.

## Q2: What's a discriminated union and when have you used one for something real?

Round: JS/TS deep-dive
Repo anchor: [context.ts:13-20](../../../packages/services/src/context.ts#L13-L20) — the 7-variant `Actor` union; exhaustive switches at [context.ts:60-97](../../../packages/services/src/context.ts#L60-L97).
What it's really testing: whether your TS goes beyond annotations into *modeling*.
Junior: "a union with a `type` field; you switch on it."
Mid adds: narrowing per branch gives you exact fields (`actor.keyId` only exists on apiKey/mcp); exhaustive switch means adding `{type:"cli"}` breaks compilation at every dispatch site — the compiler becomes your TODO list.
Senior includes: unions as **contract design** — the union is the authorization surface (requireScope branches on it, [require-scope.ts:28-30](../../../packages/services/src/auth/require-scope.ts#L28-L30)); also when *not* to union (open-ended plugin sets → interfaces/registries).
Follow-ups: `never`-based exhaustiveness check; unions vs enums.
Drill: add a hypothetical actor type on paper and list the compile errors.

## Q3: `unknown` vs `any` — show me code where the difference mattered.

Round: JS/TS deep-dive
Repo anchor: [emit.ts:25-60](../../../packages/services/src/audit/emit.ts#L25-L60) — `deepEqual(a: unknown, b: unknown)` forces every access through narrowing (`typeof`, `Array.isArray`, `instanceof Date`); contrast the deliberate, commented casts policy in [CLAUDE.md](../../../CLAUDE.md) ("centralize the cast in a named helper… comment the runtime guarantee").
Junior: "any disables checking, unknown makes you check."
Mid adds: `unknown` at boundaries (JSON, catch clauses) + zod parse to get typed values; `any` spreads virally through inference.
Senior includes: cast *policy* — unavoidable casts get a named helper + a comment stating the runtime invariant that makes it safe; a scattered `as never` is a missing helper (repo's own words).
Drill: rewrite a `catch (e: any)` handler with `unknown` + narrowing, aloud.

## Q4: How does zod give you types, and where do you put parses in a layered app?

Round: JS/TS deep-dive
Repo anchor: input parse at service top ([create.ts:22](../../../packages/services/src/monitor/create.ts#L22)); **DB row re-parse** ([create.ts:61](../../../packages/services/src/monitor/create.ts#L61)); queue payload parse ([checker/index.ts:96-99](../../../apps/workflows/src/checker/index.ts#L96-L99)).
What it's testing: parse-don't-validate; trust boundaries.
Junior: "zod validates request bodies."
Mid adds: `z.infer` keeps types and validators in one artifact; parse at *every* trust boundary — including reads from your own DB when columns hold serialized data (regions text → array).
Senior includes: cost/ownership — duplicated schemas (tRPC input vs service input, [monitor.ts:39-61](../../../packages/api/src/router/monitor.ts#L39-L61)) are deliberate decoupling of wire contract from domain contract; error mapping (`ZodError` → BAD_REQUEST with field issues, [service-adapter.ts:34-40](../../../packages/api/src/service-adapter.ts#L34-L40)).
Drill: whiteboard the three parse sites for Flow 1 from memory.

## Q5: What can go wrong with `instanceof` in a monorepo, and what are the alternatives?

Round: JS/TS deep-dive
Repo anchor: [context.ts:33-38](../../../packages/services/src/context.ts#L33-L38) — comment explains drizzle's symbol-keyed `is()` is used because pnpm can load two copies of a class, breaking `instanceof`.
Junior: (usually hasn't seen it) "instanceof checks the prototype chain."
Mid adds: duplicate module instances (pnpm/npm dedupe failures, mixed CJS/ESM) make `instanceof` false negatives; alternatives: brand symbols, discriminant properties, `Symbol.hasInstance`.
Senior includes: this is why library authors ship `isX()` helpers; error classes across package boundaries have the same problem — check `err.code` not `err instanceof` at boundaries (this repo's ServiceError check at [service-adapter.ts:41](../../../packages/api/src/service-adapter.ts#L41) works because api and services share one install — fragile if split).
Drill: explain the failure to a junior in 60 seconds.

## Q6: Explain error propagation design across layers — not try/catch syntax, the *design*.

Round: JS/TS deep-dive
Repo anchor: [errors.ts](../../../packages/services/src/errors.ts#L1-L82) taxonomy → [service-adapter.ts:32-87](../../../packages/api/src/service-adapter.ts#L32-L87) mapping; deliberate idempotent swallow at [monitor.ts:87-92](../../../packages/api/src/router/monitor.ts#L87-L92) (delete of already-deleted returns success, preserving legacy behavior — with a comment).
Junior: "throw custom errors, catch at the top."
Mid adds: typed codes decouple services from HTTP; unknown errors get generic messages so internals don't leak (L79–86); `cause` preserved for Sentry.
Senior includes: error contracts are API contracts (a 404→500 regression breaks clients); swallowing is acceptable only when *documented and intentional* (the delete example); consistency across surfaces is a real cost (402 vs 429 here).
Drill: map all 8 ServiceError codes to HTTP statuses from memory.

## Q7: How do Node/Bun servers handle concurrent state without locks — and where does that bite?

Round: JS/TS deep-dive
Repo anchor: [checker/index.ts:128-348](../../../apps/workflows/src/checker/index.ts#L128-L348) — single-threaded JS doesn't save you: two `/updateStatus` *requests* interleave at every `await`, so read-check-write on `monitor.status` races across requests even with zero threads.
Junior: "Node is single-threaded so no race conditions."
Mid adds: races live between awaits (interleaved request handlers) and between *processes* (Fly runs multiple instances); need DB-level guarantees (transactions, unique constraints, CAS).
Senior includes: names the exact mitigations present (unique-insert dedupe at [alerting.ts:131-139](../../../apps/workflows/src/checker/alerting.ts#L131-L139)) and absent (no tx around the status flip); proposes per-key serialization.
Drill: narrate the interleaving that double-sends a notification, step by step.

## Q8: `AsyncLocalStorage` — what problem does it solve here?

Round: JS/TS deep-dive
Repo anchor: [server index.ts:1, 86, 113-120](../../../apps/server/src/index.ts#L109-L120) — logtape `withContext` binds request_id/method/url so any log inside the request carries them without parameter threading.
Junior: rarely knows it.
Mid adds: continuation-local context across awaits; the alternative (passing a ctx param everywhere) and why frameworks use ALS for request IDs/tracing.
Senior includes: pitfalls — context loss across unusual async boundaries (callbacks from native code, queues), memory retention, and why it's fine for logging but risky for authorization data.
Drill: explain how a log line inside `triggerNotifications` gets its request_id.

## Q9: Module systems: what breaks when a package must run on Next.js Edge?

Round: JS/TS deep-dive
Repo anchor: [emit.ts:14-24](../../../packages/services/src/audit/emit.ts#L14-L24) — hand-rolled deepEqual because `node:util` isn't available on Edge; CLAUDE.md states the service package must avoid `node:*`.
Junior: "Edge is like a browser environment."
Mid adds: Edge = V8 isolates, subset of Node APIs; a transitive `node:` import anywhere in the package poisons it; that's an *architectural constraint on dependencies*.
Senior includes: enforcing the invariant (lint rules, separate tsconfig/exports conditions, CI smoke on edge runtime) rather than relying on comments; splitting routers edge/lambda ([root.ts](../../../packages/api/src/root.ts#L5)) by runtime need.
Drill: list three Node APIs you must avoid and their Edge substitutes.

## Q10: Design a retry policy. When are retries harmful?

Round: JS/TS deep-dive
Repo anchor: three stacks: Go `backoff` around the probe ([checker.go:276](../../../apps/checker/handlers/checker.go#L276)), Effect exponential retry on enqueue ([cron/checker.ts:204-208](../../../apps/workflows/src/cron/checker.ts#L204-L208)) and on notification send ([alerting.ts:163-168](../../../apps/workflows/src/checker/alerting.ts#L163-L168)).
Junior: "retry 3 times with backoff."
Mid adds: only retry idempotent ops (or make them idempotent first — task names, trigger rows); classify errors (ALREADY_EXISTS is *success*, [cron/checker.ts:192-202](../../../apps/workflows/src/cron/checker.ts#L192-L202)); cap totals.
Senior includes: retry amplification across layers (multiply the three stacks); jitter; budgets/circuit breakers; and the observability requirement — a retried success should still be visible.
Drill: compute worst-case downstream calls for one failing probe.

## Q11: How do you keep a long array-processing loop from starving a server?

Round: JS/TS deep-dive
Repo anchor: [cron/checker.ts:185-211](../../../apps/workflows/src/cron/checker.ts#L185-L211) — bounded concurrency (100) over potentially thousands of task creations; rate limiter for email batch at [cron/monitor.ts:45, 140-144](../../../apps/workflows/src/cron/monitor.ts#L140-L144).
Junior: "use async so it doesn't block."
Mid adds: unbounded `Promise.all` = memory + downstream overload; bounded concurrency pools; CPU-bound loops need chunking/yielding (`setImmediate`) or workers.
Senior includes: backpressure end-to-end — the queue absorbs bursts so the web tier doesn't; what metric tells you the bound is wrong.
Drill: implement `mapWithConcurrency(items, n, fn)` on the whiteboard (classic!).

## Q12: What is superjson solving in a tRPC app?

Round: JS/TS deep-dive
Repo anchor: [trpc.ts:104-119](../../../packages/api/src/trpc.ts#L104-L119) — `transformer: superjson`.
Junior: "it serializes data."
Mid adds: JSON can't represent Date/Map/undefined/BigInt; superjson encodes them with type metadata so `createdAt` arrives as a real `Date` — schema fields like `createdAt: integer(..., { mode: "timestamp" })` ([monitor.ts:59-64](../../../packages/db/src/schema/monitors/monitor.ts#L59-L64)) surface as Dates client-side.
Senior includes: transformer cost on large payloads; contract implications (REST consumers don't get this — another reason wire schemas are duplicated).
Drill: name four values plain JSON mangles and how you've handled each.

## Q13: Bun vs Node here — what actually differs for application code?

Round: JS/TS deep-dive
Repo anchor: server runs `bun run --hot src/index.ts` (apps/server package.json); tests are `bun:test` imports ([monitor.test.ts:1](../../../packages/services/src/monitor/__tests__/monitor.test.ts#L1)); top-level await at [server index.ts:64](../../../apps/server/src/index.ts#L64).
Junior: "Bun is faster Node."
Mid adds: mostly Node-API compatible; built-in test runner/TS execution removes build steps; the `export default { port, fetch }` server shape ([index.ts:273-275](../../../apps/server/src/index.ts#L273-L275)) is Bun's fetch-handler convention (shared with edge runtimes).
Senior includes: runtime choice as a *portability contract* — Hono apps target the fetch standard so they run on Bun/Node/Workers; risk: subtle compat gaps, so CI must run on the production runtime.
Drill: explain why `showRoutes` + top-level `await configure(...)` would need care under serverless cold starts.

## Q14: Show me non-trivial generics you've actually benefited from.

Round: JS/TS deep-dive
Repo anchor: [context.ts:7-11](../../../packages/services/src/context.ts#L7-L11) — `DrizzleTx` derived via `Parameters<Parameters<DrizzleClient["transaction"]>[0]>[0]` because the lib doesn't export the type; also `withTransaction<T>` (L40) preserving the callback's return type.
Junior: "generics make functions reusable."
Mid adds: deriving types from values (`typeof db`), indexed access + `Parameters`/`ReturnType` to extract library internals without forking them.
Senior includes: the maintenance tradeoff — derived types break loudly on lib upgrades (good: they *tell* you), and the comment documents why the hack exists; knows when to stop (don't type-golf what a small interface states clearly).
Drill: derive the element type of a promise-returning function's resolved array on the whiteboard: `Awaited<ReturnType<F>>[number]`.

---

Study protocol: two cards per day, spoken aloud with the anchor file open; on day 2 re-answer yesterday's without the file. Track which "senior includes" items you naturally hit — those are your honest level.
