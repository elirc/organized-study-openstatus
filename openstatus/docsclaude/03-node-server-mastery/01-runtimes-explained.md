# 01 — Runtimes Explained

"It works locally" → "it crashes in production" is almost always a runtime mismatch. Senior engineers know which runtime each piece of code runs in and what's available there.

## The three runtimes you'll touch

| | Node | Edge (V8 isolate) | Browser |
| --- | --- | --- | --- |
| `node:fs`, `node:net`, `node:tls`, `node:crypto` (full) | ✅ | ❌ / partial | ❌ |
| Streams, Buffers (Node-style) | ✅ | ❌ (Web Streams only) | Web Streams |
| `setTimeout` longer than ~30s | ✅ | ✅ within request budget | ✅ |
| Cold start | seconds | ms | n/a |
| Memory budget | large (per Lambda/container) | tiny (~128MB) | depends |
| Long-lived state (in-memory cache) | yes (per instance) | no — request-scoped | per tab |
| Bundle size matters | a bit | hugely (cold start) | hugely (download) |
| Process can crash without losing other requests | depends | one request at a time | tab-scoped |

The Edge runtime is essentially "the browser, but on a server, with limited Node compat." It can't load native modules, can't use most `node:*` APIs, can't keep state across requests, and gets killed if its bundle is too big.

## OpenStatus's edge/lambda split

[`packages/api/src/edge.ts`](../../packages/api/src/edge.ts) — edge-safe routers: workspace, monitor, page, statusReport, etc. These are pure DB + computation; nothing Node-specific.

[`packages/api/src/lambda.ts`](../../packages/api/src/lambda.ts) — Node-only routers: Stripe, email (Resend), API keys, integrations, blob. These ship Node SDKs that won't run in Edge.

[`apps/dashboard/src/lib/trpc/shared.ts:38-77`](../../apps/dashboard/src/lib/trpc/shared.ts#L38-L77) — the client-side router that picks the endpoint based on the procedure's top-level name.

When you write a new procedure, ask: *does it import anything Node-specific?* If yes, register it in `lambdaRouter` and add its name to the `lambdas` array.

## How to tell what runtime a file is in

In Next.js App Router files (`route.ts`, `page.tsx`, `layout.tsx`):

```ts
export const runtime = "edge";    // or "nodejs"
```

Default is `nodejs`. The edge route handlers in OpenStatus declare `runtime = "edge"` or rely on the default for that surface.

In a `"use client"` file, you're in the browser. Period. No `node:*`, no `process.env` (except `NEXT_PUBLIC_*`, which Next.js inlines at build).

In a service file (no `"use client"`, no `"use server"`, not in an Edge route handler), you're in whatever runtime the *importer* uses. Be conservative: assume Edge, write portable code.

## The Edge runtime's "use this not that" cheatsheet

| Don't | Do |
| --- | --- |
| `import fs from "node:fs"` | Don't read files at runtime in Edge; bake them at build |
| `import { Buffer } from "node:buffer"` | `Uint8Array`, `TextEncoder`, `TextDecoder` |
| `crypto.createHash("sha256")` | `crypto.subtle.digest("SHA-256", data)` (returns a Promise) |
| `setTimeout(..., 60_000)` for cron-like work | A background worker (`apps/workflows`) or `after()` |
| Module-level mutable cache | request-scoped via `cache()` |
| Stripe Node SDK, Resend Node SDK | their HTTP API directly, or move the call to lambda |

## CommonJS vs ESM vs the bundler

Edge runs ES modules; the bundler converts what it must. Some Node CJS packages can't be converted cleanly (dynamic `require`, native bindings). When you `import` such a package into an Edge route, the build will succeed but throw at runtime with `Module not found: Can't resolve 'node:tls'` or similar.

The fix is rarely "find an ESM fork." It's usually "this package belongs in the Lambda runtime."

## Streams: Node vs Web

Web Streams (`ReadableStream`, `WritableStream`, `TransformStream`) work in Edge and modern Node. Node's old streams (`stream` module) don't work in Edge. When you process responses or large payloads:

```ts
// Web stream — works everywhere modern
for await (const chunk of response.body!) {
  // process Uint8Array chunks
}
```

In OpenStatus you rarely deal with streams directly — the surface area is small — but if you add file uploads or SSE, this matters.

## Process model

- **Node Lambda** — each request can run in any of N warm function instances. Module-level state persists between requests on the *same* instance, but isn't shared across instances and resets on cold start. Never rely on it for correctness.
- **Edge runtime** — request-scoped V8 isolates. No state between requests at all. Cheaper cold starts.
- **Browser tab** — long-lived process per tab. State persists across navigations *within* the same SPA, dies on full page load.

The implication: **never put correctness-critical state in module-level variables.** If you need shared state, use the DB. If you need caching, use a real cache (Vercel KV, Redis, the React Query client on the browser).

## Process env, `NEXT_PUBLIC_`, and secrets

- `process.env.FOO` in server code: reads the actual env var at runtime (Lambda) or build time (Edge — env vars are baked into the bundle).
- `process.env.NEXT_PUBLIC_FOO` in client code: Next.js statically replaces this string at build time. The value ends up in the JS bundle.
- Anything in client code is **visible to the user**. Never put secrets behind `NEXT_PUBLIC_*`.

OpenStatus uses `@t3-oss/env-nextjs` (look for `env.mjs` files like [`packages/db/env.mjs`](../../packages/db/env.mjs)) to validate env vars at boot. If a required var is missing, the app refuses to start — much better than discovering it via a runtime crash.

## Cold starts

Edge: ~10-50ms typical.
Lambda: 100ms-2s typical (cold) — heavily depends on bundle size and whether native deps need to initialize.

This is why OpenStatus puts Stripe and email on Lambda (it's already slow) but keeps the dashboard's hot path on Edge (sub-100ms response budget).

## Pitfall checklist

1. **Importing `node:fs` into an Edge route.** Build succeeds; runtime crashes.
2. **Module-level cache** in Lambda. Works locally; breaks under concurrent invocations across instances.
3. **`crypto.createHash` in Edge.** Not available. Use `crypto.subtle`.
4. **Long-running work in a request.** Both Edge and Lambda have execution budgets. Move to a worker.
5. **`NEXT_PUBLIC_DATABASE_URL`** because you wanted to "test something." Now your DB credentials are in the browser bundle.
6. **Treating Edge module state as persistent.** It's not.
7. **`require()` in an ESM file.** Use `import` or `await import()`.
8. **Forgetting `runtime = "edge"`** when a route assumes Edge constraints, then the route runs as Node and behaves differently in dev vs prod.
9. **Native dependencies** (anything with C++ bindings). Edge won't load them; Lambda might need extra config.
10. **Trying to share a DB connection across Edge invocations.** Each invocation gets a fresh isolate. Use HTTP-style DB clients (like Turso's libsql HTTP driver, which OpenStatus uses — [`packages/db/src/db.ts:1-12`](../../packages/db/src/db.ts#L1-L12)).

## Drills

- Open [`packages/db/src/db.ts`](../../packages/db/src/db.ts). Why does it use `drizzle-orm/libsql/http` instead of `drizzle-orm/libsql`? (Hint: HTTP-only client works in Edge; TCP doesn't.)
- Walk through what happens if you add a `stripe.checkout.sessions.create(...)` call inside [`packages/api/src/router/statusReport.ts`](../../packages/api/src/router/statusReport.ts). At which step does it fail?
- Find every `process.env.NEXT_PUBLIC_*` in [`apps/dashboard/src/`](../../apps/dashboard/src/). Confirm none are secrets.
- Read the `env.mjs` file for one package. Identify required vs optional vars and which Zod schema validates them.
- Predict: if you set `runtime = "nodejs"` on the edge tRPC handler, what changes in dev? In prod (Vercel)?
