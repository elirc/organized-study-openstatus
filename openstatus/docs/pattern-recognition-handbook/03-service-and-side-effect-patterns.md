# Service And Side-Effect Patterns

## Pattern 1: Scope First, Transaction Second

Real code:

```ts
// packages/services/src/status-report/create.ts:24-32
export async function createStatusReport(args: {
  ctx: ServiceContext;
  input: CreateStatusReportInput;
}): Promise<CreateStatusReportResult> {
  const { ctx } = args;
  requireScope(ctx, "write");
  const input = CreateStatusReportInput.parse(args.input);

  return withTransaction(ctx, async (tx) => {
    // writes happen here
  });
}
```

Why: permission failure should not open a transaction. The `requireScope` comment explicitly says to place the check as the first line of a write verb before `withTransaction` (`packages/services/src/auth/require-scope.ts:21-23`).

Illustrative fake code:

```ts
// Illustrative fake code.
requirePermission(ctx, "write");
const input = schema.parse(raw);
return withTransaction(ctx, async (tx) => writeRows(tx, input));
```

Review question: does a write service parse/check before opening unnecessary DB work?

## Pattern 2: Ownership Lookup Before Write

Real code:

```ts
// packages/services/src/status-report/create.ts:32-40
const page_ = await tx
  .select({ id: page.id })
  .from(page)
  .where(and(eq(page.id, input.pageId), eq(page.workspaceId, ctx.workspace.id)))
  .get();
if (!page_) throw new NotFoundError("page", input.pageId);
```

Why: tenant isolation is a domain rule, not just a UI rule. Existing router tests exercise this exact class of bug (`packages/api/src/router/statusReport.test.ts:77-207`).

Illustrative fake code:

```ts
// Illustrative fake code.
const row = await tx.select().from(resource)
  .where(and(eq(resource.id, input.id), eq(resource.workspaceId, ctx.workspace.id)))
  .get();
if (!row) throw new NotFoundError("resource", input.id);
```

Review question: does a mutation check both id and workspace id before update/delete?

## Pattern 3: Transactional Bundle

Real bundle in status-report create:

- Insert parent report (`packages/services/src/status-report/create.ts:54-63`).
- Update affected component associations (`packages/services/src/status-report/create.ts:65-69`).
- Insert initial update (`packages/services/src/status-report/create.ts:71-80`).
- Emit audit rows (`packages/services/src/status-report/create.ts:82-95`).

Illustrative fake code:

```ts
// Illustrative fake code.
return withTransaction(ctx, async (tx) => {
  const parent = await insertParent(tx);
  await replaceAssociations(tx, parent.id, input.childIds);
  const firstEvent = await insertFirstTimelineEvent(tx, parent.id);
  await audit(tx, parent, firstEvent);
  return { parent, firstEvent };
});
```

Review question: if one write fails, should the other writes survive? If not, they belong in the same transaction.

## Pattern 4: Separate Notification Side Effect

Real code:

```ts
// packages/services/src/status-report/notify.ts:10-19
/**
 * Dispatch subscriber notifications for a specific status-report update.
 * Separate from the mutation services because the dashboard runs on the Edge
 * runtime and cannot fire-and-forget ...
 */
```

```ts
// packages/services/src/status-report/notify.ts:51-58
if (!ctx.workspace.limits["status-subscribers"]) {
  return;
}
await dispatchStatusReportUpdate(input.statusReportUpdateId);
```

Why: notification has different failure and runtime constraints from DB creation. The dashboard currently invokes notification as a second awaited call (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`).

Illustrative fake code:

```ts
// Illustrative fake code.
const created = await createIncident(input);
if (input.notify) {
  await notifySubscribers(created.initialUpdateId);
}
```

Review question: should this side effect be browser-owned, server-owned, queued, or transactionally audited?

## Pattern 5: Plan-Gated No-Op

Real code:

```ts
// packages/services/src/status-report/notify.ts:51-55
if (!ctx.workspace.limits["status-subscribers"]) {
  // Plan does not enable subscribers - silently skip
  return;
}
```

Why: some features are plan-gated. This code intentionally returns instead of throwing. That may be product-correct, but it should be reflected in UI expectations.

Review question: if a plan-gated side effect silently no-ops, does the UI tell the user why?

