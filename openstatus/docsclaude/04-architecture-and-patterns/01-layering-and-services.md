# 01 — Layering and the Service Pattern

The hardest architectural skill in full-stack TypeScript is putting business logic in the right place. Get this wrong and your app turns into a knot of route handlers that all do too much, can't be reused, and can't be tested without spinning up an HTTP server.

OpenStatus has a clean answer. Steal it.

## The four layers

```
+---------------------------+
|  UI (apps/dashboard)       |  React components, hooks, forms
|  - validates user input    |
|  - calls tRPC procedures   |
|  - shows results           |
+---------------------------+
            v
+---------------------------+
|  Transport (packages/api)  |  tRPC routers
|  - HTTP boundary           |
|  - validates wire shape    |
|  - adapts ctx → service    |
|  - maps service errors     |
+---------------------------+
            v
+---------------------------+
|  Domain (packages/services)|  Business operations
|  - workspace scoping       |
|  - transactions            |
|  - audit emission          |
|  - side effects            |
+---------------------------+
            v
+---------------------------+
|  Persistence (packages/db) |  Drizzle schema + client
|  - tables, indexes, FKs    |
|  - select schemas (Zod)    |
+---------------------------+
```

Each layer talks only to the layer below. The UI never imports from `db` directly. The transport never embeds business rules. The service never assumes HTTP.

## Why this matters

If your business logic lives in route handlers, you can't reuse it for:
- A different transport (Hono, REST, GraphQL, MCP).
- A background worker.
- A test that doesn't want to spin up HTTP.
- A migration script.
- A CLI tool.

If it lives in a service module that takes a `ctx` and an `input`, it works *everywhere*. OpenStatus exploits this — the same `createStatusReport` can be called from tRPC, from a Connect/RPC adapter, from a background workflow, from a test.

## The service contract

Every service verb in OpenStatus has the same shape ([`packages/services/src/status-report/create.ts:24-99`](../../packages/services/src/status-report/create.ts#L24-L99) is the template):

```ts
export async function createStatusReport(args: {
  ctx: ServiceContext;
  input: CreateStatusReportInput;
}): Promise<CreateStatusReportResult> {
  const { ctx } = args;
  requireScope(ctx, "write");                          // 1. authz first
  const input = CreateStatusReportInput.parse(args.input);  // 2. validate input

  return withTransaction(ctx, async (tx) => {           // 3. transactional
    // 4. ownership checks via DB
    const page_ = await tx.select({ id: page.id }).from(page)
      .where(and(eq(page.id, input.pageId), eq(page.workspaceId, ctx.workspace.id))).get();
    if (!page_) throw new NotFoundError("page", input.pageId);

    // 5. domain logic (insert, validate, associate)
    const newReport = await tx.insert(statusReport).values({...}).returning().get();
    // ...

    // 6. audit row in the same tx
    await emitAudit(tx, ctx, { action: "status_report.create", ... });

    return { statusReport: newReport, initialUpdate };
  });
}
```

Six steps, in this order:

1. **Scope check** — first thing. If the actor doesn't have permission, fail before opening a connection.
2. **Input parse** — Zod re-validates the input. Trust nothing.
3. **Transaction open** — one atomic unit for all writes.
4. **Ownership check** — verify the caller can act on the referenced resources.
5. **Domain logic** — the actual writes, validations, side effects.
6. **Audit emit** — inside the same transaction. Fail-closed.

If you find a service that breaks the order — e.g., transaction open before scope check, or audit emit after the transaction — flag it. Order matters.

## What belongs in a service vs a router

| Belongs in a service | Belongs in a router |
| --- | --- |
| DB writes and reads | HTTP error code mapping |
| Workspace scoping | Wire shape (legacy field names) |
| Authorization (scope check) | Analytics metadata (`meta.track`) |
| Transactions | Calling the service |
| Audit emission | Adapting input names between wire and domain |
| Cross-entity invariants | Validating wire input with Zod |
| Side effects (notifications, webhooks) | (almost nothing else) |

Routers should be 5–30 lines per procedure. If your router is doing real work, that work belongs in a service.

## The adapter pattern

Routers translate between two worlds: the HTTP/tRPC wire and the service domain. They use small adapters:

- `toServiceCtx(ctx)` ([`packages/api/src/service-adapter.ts:18-24`](../../packages/api/src/service-adapter.ts#L18-L24)) converts the tRPC context into a `ServiceContext`.
- `toTRPCError(err)` ([`packages/api/src/service-adapter.ts:32-87`](../../packages/api/src/service-adapter.ts#L32-L87)) converts service-layer errors back into `TRPCError`s.

Combined, they're the boundary. Routers do this pattern:

```ts
try {
  const result = await someService({ ctx: toServiceCtx(ctx), input: { ... } });
  return adaptResultToWireShape(result);
} catch (err) {
  toTRPCError(err);
}
```

Three lines of real logic; everything else is service.

## Wire shape ≠ domain shape

Notice in [`packages/api/src/router/statusReport.ts:21-32, 58-81`](../../packages/api/src/router/statusReport.ts#L21-L81):

```ts
// tRPC wire input — preserves legacy dashboard field name
const createStatusReportTRPCInput = z.object({
  pageComponents: z.array(z.number()),    // <-- legacy name
  // ...
});

// inside the resolver:
await createStatusReport({
  ctx: toServiceCtx(ctx),
  input: {
    pageComponentIds: input.pageComponents,  // <-- renamed for domain clarity
    // ...
  },
});
```

The dashboard sends `pageComponents`. The service uses `pageComponentIds` because that's what the field *is*. The router translates.

**Why bother?** Because you don't want to break the dashboard for a rename. And the service shouldn't carry legacy baggage from one consumer. The router is where the impedance mismatch lives — and that's the right place for it.

**Senior aside:** *"Backwards-compatible at the edge, clean in the middle" is the most reusable pattern in API design. The transport accepts what callers historically sent. The domain has the shape it wants to have. The adapter glues them. When the legacy name is finally retired, you change one file — the adapter — not every consumer.*

## Composition between services

A service can call another service. Because services accept `ctx` (which may carry a `tx`), composed calls inherit the transaction:

```ts
export async function resolveStatusReport(args) {
  return withTransaction(args.ctx, async (tx) => {
    const ctxWithTx = { ...args.ctx, db: tx };
    await updateStatusReport({ ctx: ctxWithTx, input: {...} });
    await addStatusReportUpdate({ ctx: ctxWithTx, input: {...} });
    await emitAudit(tx, args.ctx, { action: "status_report.resolve", ... });
  });
}
```

Both inner services run inside the outer transaction. If anything throws, everything rolls back atomically. The trick is `withTransaction` ([`packages/services/src/context.ts:40-47`](../../packages/services/src/context.ts#L40-L47)) — it reuses an existing tx instead of opening a nested one.

## What about side effects (notifications, webhooks, emails)?

Side effects are messy. They cross the transaction boundary (you can't roll back an email). Options:

1. **In-band**: the service triggers the notification synchronously after the DB commit. Risk: write succeeds, notification fails, user sees "success" with no email. OpenStatus does this for status reports today — and the dashboard triggers it from `onSuccess`, which is even messier.
2. **Outbox pattern**: write the notification intent as a row inside the transaction; a background worker drains the outbox. Failure-isolated, but adds latency.
3. **Eventbus**: emit an event, let subscribers handle. Decouples, but observability suffers.

OpenStatus has a TODO ([`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:38-44`](../../apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx#L38-L44)) to move the notification *to the server* — which is correct. The longer-term fix is some form of outbox so notification reliability isn't coupled to HTTP success.

A senior reviewing this would advocate for option 2: outbox.

## Anti-patterns in this layer

- **Fat router**: 200-line procedure that does workspace lookup, DB writes, audit, notifications, response shaping. Belongs in a service.
- **Skinny service**: service is just a passthrough to one Drizzle call. Either inline it in the router (it's not really business logic) or expand it (you forgot the ownership check and the audit row).
- **Service knows about HTTP**: imports `TRPCError`, reads `Request`, sets cookies. Boundary violation.
- **Router knows about DB schema**: imports a Drizzle table, runs a `select`. Skips the service layer.
- **Service that opens a transaction without `withTransaction`**: breaks nested-call composition.

## When the pattern is *wrong* (yes, sometimes)

- **Pure derivations.** If a "service" just computes a value from inputs (no DB, no side effects), it's a function, not a service. Don't dress it up.
- **Read-heavy endpoints with no business rule.** If the read is "give me a row by id, filtered by workspace," a thin Drizzle helper might be enough. But: most "thin" reads grow business rules over time (sort order, filtering, enrichment) — defaulting to a service module future-proofs you.
- **One-off scripts.** Migration scripts and backfills should just write SQL or Drizzle directly. Service layering is for product code.

## Pitfall checklist

1. **Business logic in the router.** Move it down.
2. **DB access in the UI.** Move it up.
3. **Service that knows about `req` or `cookies`.** Boundary violation.
4. **Service that throws `TRPCError`.** Throw a `ServiceError` subclass; let the adapter map.
5. **Mutation without a transaction** when multiple rows are written.
6. **Audit emission after the transaction.** Now writes can succeed without an audit row.
7. **Cross-layer import** (UI → DB, router → DB, service → tRPC). Each one is a smell.
8. **Service that doesn't take `ctx`.** It's not reusable across actors.
9. **`requireScope` missing on a write.** API keys can write without scope.
10. **Side effect inside the transaction** that can't be rolled back (email, webhook). Either move outside or use an outbox.

## Drills

- Open three different routers (`statusReport`, `monitor`, `page`). Compare their `create` procedures. Are they the same shape?
- Open three different services (`status-report/create`, `monitor/create`, `page/create`). Compare. Are they the same shape?
- Find one router doing something that should be in a service. (Hint: there shouldn't be any, but if there are, that's your target.)
- Find one place where a side effect crosses the transaction. Sketch an outbox refactor.
- Write a service verb for a new entity (e.g., `incident.archive`). Follow the 6-step template. Don't write the router yet. Show your verb to a friend; ask if it tells the story without seeing the router.
