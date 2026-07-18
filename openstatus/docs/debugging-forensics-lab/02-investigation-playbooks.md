# Investigation Playbooks

## Playbook 1: Debug A Failed Create Mutation

Start from the user action and move inward.

1. Confirm the page submits create values only in create mode:

```tsx
// apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:89-103
if ("date" in values) {
  await createStatusReportMutation.mutateAsync({
    title: values.title,
    status: values.status,
    pageId: Number.parseInt(id),
    pageComponents: values.pageComponents,
    date: values.date,
    message: values.message,
    notifySubscribers: values.notifySubscribers,
  });
}
```

2. Confirm tRPC input accepts that shape:

```ts
// packages/api/src/router/statusReport.ts:24-32
const createStatusReportTRPCInput = z.object({
  title: z.string(),
  status: CreateStatusReportInput.shape.status,
  pageId: z.number(),
  pageComponents: z.array(z.number()),
  date: z.coerce.date(),
  message: z.string(),
  notifySubscribers: z.boolean().nullish(),
});
```

3. Confirm the router adapts to the service shape:

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

4. Confirm service ownership checks pass:

```ts
// packages/services/src/status-report/create.ts:32-40
const page_ = await tx
  .select({ id: page.id })
  .from(page)
  .where(and(eq(page.id, input.pageId), eq(page.workspaceId, ctx.workspace.id)))
  .get();
if (!page_) throw new NotFoundError("page", input.pageId);
```

5. Confirm writes happen inside the transaction: report insert, association update, initial update insert, and audit rows (`packages/services/src/status-report/create.ts:54-98`).

Investigation output template:

```text
User action:
Payload:
tRPC validation:
Auth/workspace:
Service ownership:
DB write:
Error mapping:
Likely fix:
Test to add:
```

## Playbook 2: Debug Wrong Public Status

The public status page computes status rather than simply reading one display field.

1. Confirm the public client calls `statusPage.get` with the expected slug (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:65-79`).
2. Confirm the router loads the expected relation graph (`packages/api/src/router/statusPage.ts:80-113`).
3. Confirm the component is treated as monitor or static. `isMonitorComponent` requires type `monitor`, non-null monitor relation, active monitor, and no deleted timestamp (`packages/api/src/router/statusPage.utils.ts:21-35`).
4. Confirm `getEvents` includes the relevant report, maintenance, or incident (`packages/api/src/router/statusPage.utils.ts:128-258`).
5. Confirm page status calculation checks error, degraded, info, then success (`packages/api/src/router/statusPage.ts:222-229`).

Illustrative fake code:

```ts
// Illustrative fake code: public status priority.
if (monitors.some((m) => m.status === "error")) return "error";
if (monitors.some((m) => m.status === "degraded")) return "degraded";
if (monitors.some((m) => m.status === "info")) return "info";
return "success";
```

Real priority logic lives in `packages/api/src/router/statusPage.ts:222-229`.

## Playbook 3: Debug Workspace/Auth Problems

Use this path when a request is unexpectedly `UNAUTHORIZED`, `FORBIDDEN`, or `NOT_FOUND`.

1. Route guard: `guardTRPCSource` can return 401 when the convention header is missing, but it is not real auth (`apps/dashboard/src/lib/trpc/shared.ts:17-29`).
2. Session guard: `protectedProcedure` throws `UNAUTHORIZED` without `ctx.session.user.id` (`packages/api/src/trpc.ts:148-152`).
3. Workspace resolution: middleware reads `workspace-slug` and calls `resolveActiveWorkspace` (`packages/api/src/trpc.ts:171-200`).
4. Service adapter: `toServiceCtx` passes `workspace` and actor user id to services (`packages/api/src/service-adapter.ts:18-24`).
5. Service resource check: domain functions still verify workspace ownership, such as status-report create page lookup (`packages/services/src/status-report/create.ts:32-40`).
6. Error translation: service errors become tRPC error codes (`packages/api/src/service-adapter.ts:41-87`).

Mini decision tree:

```text
401 before router? Check x-trpc-source.
UNAUTHORIZED in protectedProcedure? Check session.
Workspace Not Found? Check workspace-slug cookie and membership.
NOT_FOUND from service? Check resource id belongs to current workspace.
FORBIDDEN from service? Check actor scope or explicit ownership mismatch.
```

## Playbook 4: Debug Test Pollution

Tests in the service layer often need transaction rollback and seeded workspaces.

Real helper:

```ts
// packages/services/test/helpers.ts:37-50
export async function withTestTransaction<T>(
  fn: (tx: DrizzleTx) => Promise<T>,
): Promise<T> {
  let result!: T;
  try {
    await db.transaction(async (tx) => {
      result = await fn(tx);
      throw new RollbackSignal();
    });
  } catch (e) {
    if (!(e instanceof RollbackSignal)) throw e;
  }
  return result;
}
```

Rules from the helper comments:

- Use it as a top-level test wrapper only (`packages/services/test/helpers.ts:18-36`).
- Use `tx` inside the callback; committed `db` reads will not see in-transaction writes (`packages/services/test/helpers.ts:33-35`).
- Load seeded workspaces with `loadSeededWorkspace` (`packages/services/test/helpers.ts:102-119`) and known ids from `fixtures.ts` (`packages/services/test/fixtures.ts:1-5`).

