# Boundary Patterns

## Pattern 1: Thin HTTP Route, Thick Router/Service

Real code:

```ts
// apps/status-page/src/app/api/trpc/edge/[trpc]/route.ts:11-24
const handler = (req: NextRequest) => {
  const blocked = guardTRPCSource(req);
  if (blocked) return blocked;

  return fetchRequestHandler({
    endpoint: "/api/trpc/edge",
    router: edgeRouter,
    req: req,
    createContext: () => createTRPCContext({ req, auth }),
    onError: createOnError("edge"),
  });
};
```

Why: HTTP route files should be small adapters. They should not contain business rules for status reports, workspaces, or notifications. Business behavior begins in tRPC routers and services (`packages/api/src/router/statusReport.ts:58-222`; `packages/services/src/status-report/create.ts:24-99`).

Illustrative fake code:

```ts
// Illustrative fake code.
export function route(req) {
  return frameworkAdapter({
    req,
    router,
    context: buildContext(req),
  });
}
```

Review question: if someone adds DB writes directly to a route handler, what service or router should own them instead?

## Pattern 2: Router As Compatibility Adapter

Real code:

```ts
// packages/api/src/router/statusReport.ts:21-23
// tRPC-side input schemas. These preserve the existing wire contract exactly;
// each procedure adapts between this shape and the canonical service schema.
```

```ts
// packages/api/src/router/statusReport.ts:64-74
input: {
  title: input.title,
  status: input.status,
  pageId: input.pageId,
  pageComponentIds: input.pageComponents,
  date: input.date,
  message: input.message,
}
```

Why: the dashboard can keep sending `pageComponents` while the service uses the more precise `pageComponentIds` name (`packages/services/src/status-report/schemas.ts:12-20`).

Illustrative fake code:

```ts
// Illustrative fake code.
const wireInput = { selectedThings: [1, 2] };
const domainInput = { thingIds: wireInput.selectedThings };
```

Review question: does this route leak transport-specific names into reusable service APIs?

## Pattern 3: Context Translation

Real code:

```ts
// packages/api/src/service-adapter.ts:18-24
export function toServiceCtx(ctx: AuthedContext): ServiceContext {
  return {
    workspace: ctx.workspace,
    actor: { type: "user", userId: ctx.user.id },
    requestId: ctx.req?.headers.get("x-request-id") ?? undefined,
  };
}
```

Why: tRPC context contains web/request/session details. Service context contains domain execution details: workspace, actor, request id, optional DB/Tinybird handles (`packages/services/src/context.ts:13-29`).

Illustrative fake code:

```ts
// Illustrative fake code.
function toDomainContext(httpContext) {
  return {
    tenant: httpContext.resolvedTenant,
    actor: { type: "user", id: httpContext.session.user.id },
  };
}
```

Review question: does the service depend on Next.js/tRPC objects, or only on `ServiceContext`?

## Pattern 4: Error Boundary Translation

Real code:

```ts
// packages/api/src/service-adapter.ts:41-87
if (err instanceof ServiceError) {
  switch (err.code) {
    case "NOT_FOUND":
      throw new TRPCError({ code: "NOT_FOUND", message: err.message });
    case "FORBIDDEN":
      throw new TRPCError({ code: "FORBIDDEN", message: err.message });
    // ...
  }
}
throw new TRPCError({
  code: "INTERNAL_SERVER_ERROR",
  message: "Internal server error",
  cause: err,
});
```

Why: callers need stable API error codes, while internals need to avoid leaking DB strings, stack traces, or third-party details (`packages/api/src/service-adapter.ts:79-87`).

Illustrative fake code:

```ts
// Illustrative fake code.
try {
  return await domainAction();
} catch (error) {
  throw mapDomainErrorToApiError(error);
}
```

Review question: does a new router mutation expose raw `err.message` to clients?

