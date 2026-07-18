# 02 — HTTP Handlers, Middleware, and Request Context

This file is about the layer between "a TCP socket accepted a request" and "your business logic runs." Get this layer wrong and you have correctness bugs at every endpoint.

## Next.js route handlers

A route handler lives at `app/api/.../route.ts` and exports HTTP-method functions:

```ts
export async function GET(req: Request)  { ... }
export async function POST(req: Request) { ... }
```

(Or `GET = handler; POST = handler` if you want one function for all methods.)

OpenStatus exports a single combined handler for tRPC because tRPC's `fetchRequestHandler` knows how to route by method. See [`apps/status-page/src/app/api/trpc/edge/[trpc]/route.ts`](../../apps/status-page/src/app/api/trpc/edge/[trpc]/route.ts) (similar for the dashboard and lambda variants):

```ts
const handler = (req: NextRequest) => {
  const blocked = guardTRPCSource(req);
  if (blocked) return blocked;

  return fetchRequestHandler({
    endpoint: "/api/trpc/edge",
    router: edgeRouter,
    req,
    createContext: () => createTRPCContext({ req, auth }),
    onError: createOnError("edge"),
  });
};

export { handler as GET, handler as POST };
```

That's the whole HTTP layer. Everything else is tRPC, services, DB.

## The request → response lifecycle

1. **Edge proxy / load balancer** terminates TLS.
2. **Next.js middleware** (if any) runs — can rewrite, redirect, or modify the request.
3. **Route handler** runs.
4. tRPC's `fetchRequestHandler` parses the path (`/api/trpc/edge/statusReport.create`), the body (batched JSON), invokes the matching procedure.
5. Procedure builds context (auth, db, workspace), runs the resolver, returns a result.
6. tRPC serializes (superjson), sends a 200 (or maps the error).
7. The browser's React Query observer wakes up.

## Middleware (Next.js sense)

`apps/dashboard/middleware.ts` (or `apps/dashboard/src/proxy.ts` — OpenStatus's pattern). Runs on every matching path *before* the route handler. Typical uses:

- Set/refresh cookies (workspace slug, locale).
- Redirect unauthenticated users to login.
- Geo-fence.
- A/B test gating.

OpenStatus's middleware ([`apps/dashboard/src/proxy.ts`](../../apps/dashboard/src/proxy.ts)) is where the `workspace-slug` cookie is set on incoming server requests so that the tRPC context resolver can read it.

## tRPC middleware (different sense — per-procedure)

tRPC has its own middleware system that wraps procedures. [`packages/api/src/trpc.ts:148-241`](../../packages/api/src/trpc.ts#L148-L241):

```ts
const enforceUserIsAuthed = t.middleware(async (opts) => {
  const { ctx } = opts;
  if (!ctx.session?.user?.id) {
    throw new TRPCError({ code: "UNAUTHORIZED" });
  }
  // ... resolve workspace, inject user + workspace into ctx
  const result = await opts.next({ ctx: { ...ctx, user, workspace } });
  // ... post-execution analytics tracking via `after()`
  return result;
});

export const protectedProcedure = t.procedure.use(enforceUserIsAuthed);
```

Pattern: a middleware can run code before, mutate the context, await `next()`, and run code after. Errors thrown become tRPC errors with appropriate codes.

**Senior aside:** *Per-procedure middleware is how you DRY up cross-cutting concerns — auth, audit, rate limiting, tracing. If you find the same five lines at the top of ten routes, that's middleware screaming to be born.*

## Context — the request-scoped bundle

Every tRPC request gets a `Context` object. It carries:

- `session` — NextAuth session (user id + email, or null).
- `req` — the raw `NextRequest` (headers, cookies, IP).
- `db` — the Drizzle client (same one for every request; safe because it's stateless HTTP).
- `user`, `workspace` — null on entry; populated by `protectedProcedure`.
- `metadata` — user agent, location, etc.

See [`packages/api/src/trpc.ts:59-94`](../../packages/api/src/trpc.ts#L59-L94) for context construction.

The context is the *only* state your procedures should depend on. Don't reach for globals. Don't import a singleton built at module load. Everything request-scoped lives on `ctx`.

## Error handling at the boundary

When a procedure throws, tRPC catches it. If it's a `TRPCError`, the code is preserved. If it's anything else, tRPC defaults to `INTERNAL_SERVER_ERROR`.

OpenStatus's `errorFormatter` ([`packages/api/src/trpc.ts:109-119`](../../packages/api/src/trpc.ts#L109-L119)) adds a structured `zodError` to the response so the client can display field-level validation errors:

```ts
errorFormatter({ shape, error }) {
  return {
    ...shape,
    data: {
      ...shape.data,
      zodError:
        error.cause instanceof ZodError ? treeifyError(error.cause) : null,
    },
  };
}
```

That's how the form (`isTRPCClientError(err)`) can branch on validation failures vs other server errors.

## `onError` — server-side logging

Every route handler accepts `onError`. OpenStatus uses a shared logger ([`apps/dashboard/src/lib/trpc/shared.ts:10-15`](../../apps/dashboard/src/lib/trpc/shared.ts#L10-L15)):

```ts
export function createOnError(label: string) {
  return ({ error }) => {
    console.log(`Error in tRPC handler (${label})`);
    console.error(error);
  };
}
```

In production you'd wire this to Sentry/Datadog/logtape. The pattern matters more than the destination.

## Headers and cookies

```ts
req.headers.get("user-agent");
req.cookies.get("workspace-slug")?.value;
```

Cookies are part of the request. They're how OpenStatus tracks active workspace ([`packages/api/src/trpc.ts:176`](../../packages/api/src/trpc.ts#L176)). They're set in middleware or by NextAuth.

Important: **never trust a cookie's value blindly.** It came from the user. If `workspace-slug` says "secret-workspace," the server must still verify the user belongs to that workspace. OpenStatus does this in `resolveActiveWorkspace`.

## `x-trpc-source` — a convention header

[`apps/dashboard/src/lib/trpc/shared.ts:17-29`](../../apps/dashboard/src/lib/trpc/shared.ts#L17-L29):

```ts
export function guardTRPCSource(req: Request): Response | null {
  const source = req.headers.get("x-trpc-source");
  if (source !== "server" && source !== "client") {
    return new Response(null, { status: 401 });
  }
  return null;
}
```

The comment says it explicitly: **convention filter, not security**. The header is trivially spoofable. Its purpose is to keep bots and random crawlers from hitting the tRPC endpoint. Real auth is in `protectedProcedure`.

A senior reading this should:
- Recognize the comment is load-bearing.
- *Not* mistake `guardTRPCSource` for authentication.
- Push back hard if anyone proposes "we can rely on the header in production."

## CORS

The dashboard and the API live on the same origin in production, so CORS isn't relevant for the dashboard's tRPC calls. The status-page app does cross-origin (different subdomain), and the API route handler can set CORS headers if needed. OpenStatus avoids CORS by serving the API on the same origin as each app.

## Rate limiting

OpenStatus doesn't ship request-level rate limiting in the open source surface (it relies on Vercel's protections + workspace-level quotas at the service layer). A senior reviewing this would ask: "what stops 1M requests/sec from one IP?" The answer today is "Vercel." A real production deployment likely wants Upstash Ratelimit or similar at the middleware tier.

## Pitfall checklist

1. **Module-level state for per-request data.** Doesn't work in serverless; doesn't work for multi-tenant safety.
2. **Reading request body twice.** Streams are consumed once. Clone with `req.clone()` if you must.
3. **Throwing a generic `Error` from a procedure.** Becomes `INTERNAL_SERVER_ERROR` with a generic message — your debug info is lost. Use `TRPCError` with a code, or a custom error class the adapter maps.
4. **Trusting client headers.** `x-trpc-source`, `x-user-id`, anything from the browser. Verify on the server.
5. **Setting cookies in a tRPC procedure.** tRPC procedures aren't the right place — they have no direct response object. Set cookies in middleware or a route handler.
6. **Returning sensitive errors verbatim.** Stack traces, DB strings, third-party SDK internals. Map errors at the boundary.
7. **Long-running work in a request handler.** Move to a background worker or stream the response.
8. **Forgetting `await` on `next()`** inside a tRPC middleware. Order of operations breaks.
9. **Reading cookies in an Edge route from `next/headers` instead of `req.cookies`.** Both exist; pick the one Next.js expects in your runtime.
10. **Assuming `req.ip` exists.** It depends on the platform. Use `x-forwarded-for` parsing with care.

## Drills

- Open [`apps/status-page/src/app/api/trpc/edge/[trpc]/route.ts`](../../apps/status-page/src/app/api/trpc/edge/[trpc]/route.ts) and the dashboard equivalent. Identify what differs. Why are there separate route handlers per app?
- Walk through `protectedProcedure` ([`packages/api/src/trpc.ts:148-241`](../../packages/api/src/trpc.ts#L148-L241)) line by line. List every side effect.
- Read [`packages/api/src/trpc.ts:154-169`](../../packages/api/src/trpc.ts#L154-L169) — the "test escape hatch." Why does it exist? What kind of test would fail without it?
- Add a new tRPC middleware that adds a `requestStart` timestamp to ctx and logs total duration in a post-hook. Decide where it should mount in the chain.
- Find the only place in this repo that throws a `TRPCError` directly (not via the adapter). Is that the right boundary, or should it go through the service?
