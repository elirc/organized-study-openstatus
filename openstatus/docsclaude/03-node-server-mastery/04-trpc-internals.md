# 04 — tRPC Internals

tRPC feels like magic the first time. It isn't. It's a small, well-shaped library you can read in an afternoon. This file walks the OpenStatus setup end to end so the magic dissolves.

## The contract

You define procedures on the server. The client gets a typed object whose shape mirrors the server. Calling a client method makes an HTTP request, the server runs the matching procedure, the result comes back typed.

There's no separate API schema (no OpenAPI, no proto). The server *is* the schema, and TypeScript carries the types over.

## The three pieces

1. **`initTRPC.create()`** — builds the `t` object with router/procedure factories.
2. **Procedures** — input → middleware chain → resolver → output.
3. **Routers** — recursive maps of names to procedures or sub-routers.

[`packages/api/src/trpc.ts:104-119`](../../packages/api/src/trpc.ts#L104-L119):

```ts
export const t = initTRPC
  .context<Context>()
  .meta<Meta>()
  .create({
    transformer: superjson,
    errorFormatter({ shape, error }) {
      return {
        ...shape,
        data: {
          ...shape.data,
          zodError:
            error.cause instanceof ZodError ? treeifyError(error.cause) : null,
        },
      };
    },
  });

export const createTRPCRouter = t.router;
export const publicProcedure = t.procedure;
```

`.context<Context>()` declares what every resolver receives. `.meta<Meta>()` lets you attach arbitrary metadata to procedures (used for `track: Events.CreateReport`-style analytics). `transformer: superjson` enables `Date`, `Map`, `Set`, `BigInt`, `undefined` round-tripping.

## A procedure, decomposed

```ts
create: protectedProcedure                          // 1. base + middleware chain
  .meta({ track: Events.CreateReport })             // 2. metadata
  .input(createStatusReportTRPCInput)               // 3. Zod input schema
  .mutation(async ({ ctx, input }) => {             // 4. resolver
    try {
      const { initialUpdate } = await createStatusReport({
        ctx: toServiceCtx(ctx),
        input: { /* adapted */ },
      });
      return { ...initialUpdate, notifySubscribers: input.notifySubscribers };
    } catch (err) {
      toTRPCError(err);
    }
  }),
```

1. **`protectedProcedure`** is `t.procedure.use(enforceUserIsAuthed)`. Every call goes through the auth middleware first.
2. **`.meta({...})`** attaches read-only metadata. Middleware can inspect it (`opts.meta`).
3. **`.input(schema)`** validates the incoming JSON-decoded input. On failure, tRPC throws `BAD_REQUEST` with the `ZodError` attached.
4. **`.mutation(fn)`** (or `.query(fn)`, `.subscription(fn)`) sets the resolver. The resolver receives `{ ctx, input }` and returns the result (or throws).

## How types flow

The router type is captured at export:

```ts
// packages/api/src/root.ts (composition root)
import { edgeRouter } from "./edge";
import { lambdaRouter } from "./lambda";

export const appRouter = mergeRouters(edgeRouter, lambdaRouter);
export type AppRouter = typeof appRouter;
```

The client receives `AppRouter`:

```ts
import type { AppRouter } from "@openstatus/api";
const { useTRPC } = createTRPCContext<AppRouter>();
```

`createTRPCContext` is generic over the router. It builds a proxy where every property is typed based on the router's shape. When you call `trpc.statusReport.create.mutate(input)`, the input type is exactly `createStatusReportTRPCInput`'s inferred type, and the return type is the resolver's return type.

No code generation. Just `typeof` + generics + a proxy.

## The proxy trick

The client doesn't have a hand-written object with `statusReport.create.mutate`. It has a `Proxy` that intercepts every property access, builds up the path (`statusReport.create.mutate`), and when you call the leaf, sends a request with that path.

```ts
// Simplified mental model:
function createClient(url) {
  function build(path) {
    return new Proxy(() => {}, {
      get(_, key) { return build([...path, key]); },
      apply(_, __, args) {
        return fetch(url, { method: "POST", body: JSON.stringify({ path: path.join("."), input: args[0] }) }).then(r => r.json());
      },
    });
  }
  return build([]);
}
```

The real implementation is more sophisticated (batching, links, transformers), but that's the shape. Any property access works at runtime because of the proxy; TypeScript narrows it to valid procedures.

## Links — the request pipeline

The tRPC client is a chain of *links*. Each link can intercept, log, batch, retry, redirect.

OpenStatus's client setup ([`apps/dashboard/src/lib/trpc/client.tsx:42-55`](../../apps/dashboard/src/lib/trpc/client.tsx#L42-L55)):

```ts
createTRPCClient<AppRouter>({
  links: [
    loggerLink({ enabled: ... }),     // logs in dev or on error
    endingLink({                       // routes to edge or lambda
      headers: { "x-trpc-source": "client" },
    }),
  ],
})
```

`endingLink` is the custom link ([`apps/dashboard/src/lib/trpc/shared.ts:46-77`](../../apps/dashboard/src/lib/trpc/shared.ts#L46-L77)) that decides which HTTP endpoint to use based on the procedure's first path segment.

The order matters: requests flow top-down through links on the way out, bottom-up on the way back. `loggerLink` first → it sees the unmodified request and the final response.

## Batching

`httpBatchLink` (used inside `endingLink`) batches multiple procedure calls happening in the same tick into a single HTTP request. The server unpacks them, runs each, returns an array of results.

Benefit: if a page renders 5 components that each call `useQuery` on different procedures, you get *one* HTTP request, not five.

Cost: a batch is only as fast as its slowest member. A long-running query in the batch blocks the others.

## Transformers

`superjson` is a transformer. It handles encoding/decoding for types JSON can't represent. Both sides must use the same transformer (the client and server are both configured with `superjson`).

If you skip it: `Date` becomes a string, `undefined` becomes missing, `Map`/`Set` become `{}`. Don't skip it.

## Procedure builders are immutable

```ts
const protectedProcedure = t.procedure.use(enforceUserIsAuthed);
const writeProcedure = protectedProcedure.use(checkWriteScope);   // new procedure, not mutating
```

Each `.use(...)`, `.input(...)`, `.meta(...)` returns a new builder. You can compose freely:

```ts
const auditedProcedure = protectedProcedure
  .use(auditMiddleware)
  .meta({ trackProps: ["pageId"] });

export const myRoute = auditedProcedure.input(schema).mutation(...);
```

OpenStatus could centralize more middleware this way. Today the audit emission lives inside service code (which is the right call: services are where mutations actually happen, and they enforce the audit-row-in-transaction invariant).

## Errors

```ts
throw new TRPCError({ code: "NOT_FOUND", message: "Page not found" });
```

Valid codes are HTTP-status-like: `BAD_REQUEST`, `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`, `CONFLICT`, `PRECONDITION_FAILED`, `TOO_MANY_REQUESTS`, `INTERNAL_SERVER_ERROR`. The client gets a `TRPCClientError` with `.data.code` set; map to HTTP statuses for non-tRPC consumers.

OpenStatus's adapter does this mapping centrally ([`packages/api/src/service-adapter.ts:32-87`](../../packages/api/src/service-adapter.ts#L32-L87)). Routers don't `throw new TRPCError(...)` themselves; they catch and call `toTRPCError(err)`. That keeps error mapping in one place and prevents leakage of raw error messages.

## Server-side calling

You can call procedures from server code without HTTP:

```ts
const caller = appRouter.createCaller(ctx);
const result = await caller.statusReport.list({ pageId: 7 });
```

OpenStatus's tests use this pattern ([`packages/api/src/router/statusReport.test.ts:84-94`](../../packages/api/src/router/statusReport.test.ts#L84-L94)) — a real router invocation with a constructed context. Fast, integration-style.

## Subscriptions

tRPC supports WebSocket subscriptions. OpenStatus doesn't use them — uptime alerts go through email/Slack/Discord/etc. via the notification system. If real-time dashboards became a requirement, subscriptions would be a fit.

## Pitfall checklist

1. **Not using a transformer.** Dates become strings; your code mysteriously breaks.
2. **Inline `throw new TRPCError`** in routers, with no adapter. Error mapping scattered.
3. **Heavy work in a batch.** One slow procedure delays four fast ones.
4. **`.input()` after `.mutation()`** — wrong order. Builders are sequenced.
5. **Procedures that don't return.** Make it `void` explicitly so the client knows.
6. **Stateful middleware.** Don't store anything in module scope; use ctx.
7. **Server-side caller without proper ctx.** You'll bypass auth. Only do this when you've constructed ctx as if the request had authed.
8. **Forgetting `meta.track`** in analytics-sensitive routes. Silent metrics gap.
9. **Mixing edge and lambda procedures** in a single batch. The client splits by endpoint, not by router; the `endingLink` here decides per-call.
10. **Mutating ctx inside a middleware.** Use `opts.next({ ctx: { ...ctx, x: ... } })` — never write through.

## Drills

- Read [`packages/api/src/router/statusReport.ts`](../../packages/api/src/router/statusReport.ts) end to end. For each procedure, identify: input schema, resolver behavior, return shape, error path.
- Trace one procedure from button click to DB write. Note every middleware that runs, every transformer that touches data, every error boundary.
- Add a new public procedure that returns the current server time. Call it from the client. Confirm `Date` survives the trip thanks to superjson.
- Find the one place a `TRPCError` is thrown *outside* the adapter. Decide if it belongs there.
- Create a custom middleware that logs `ctx.user?.id` and the procedure path before running. Wire it onto one procedure. Confirm via console.
