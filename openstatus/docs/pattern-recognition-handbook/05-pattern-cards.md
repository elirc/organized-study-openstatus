# Pattern Cards

Use these as quick recognition drills.

## Card 1: Boundary Adapter

Looks like:

```ts
// Illustrative fake code.
apiProcedure.input(apiSchema).mutation(({ ctx, input }) => {
  return service({
    ctx: toServiceCtx(ctx),
    input: adapt(input),
  });
});
```

Real anchor: `statusReport.create` adapts tRPC input to service input (`packages/api/src/router/statusReport.ts:58-81`).

Ask: what contract is preserved for callers?

## Card 2: Canonical Service Schema

Looks like:

```ts
// Illustrative fake code.
export const CreateThingInput = z.object({
  name: z.string().min(1),
  relatedIds: z.array(z.number().int()).default([]),
});
```

Real anchor: `CreateStatusReportInput` (`packages/services/src/status-report/schemas.ts:12-20`).

Ask: can this service be called outside tRPC without understanding React form names?

## Card 3: Tenant Ownership Check

Looks like:

```ts
// Illustrative fake code.
where(and(eq(table.id, input.id), eq(table.workspaceId, ctx.workspace.id)))
```

Real anchor: page ownership lookup in `createStatusReport` (`packages/services/src/status-report/create.ts:32-40`).

Ask: does this query prove the resource belongs to the active workspace?

## Card 4: Transaction Bundle

Looks like:

```ts
// Illustrative fake code.
return withTransaction(ctx, async (tx) => {
  const row = await createRow(tx);
  await createAssociations(tx, row.id);
  await audit(tx, row);
  return row;
});
```

Real anchor: `createStatusReport` transaction (`packages/services/src/status-report/create.ts:32-98`).

Ask: if one write fails, should all writes roll back?

## Card 5: Batched Read Model

Looks like:

```ts
// Illustrative fake code.
const parentIds = parents.map((p) => p.id);
const children = await loadChildrenForParents(parentIds);
return parents.map((p) => ({ ...p, children: childrenByParent[p.id] ?? [] }));
```

Real anchor: `enrichReportsBatch` (`packages/services/src/status-report/list.ts:66-161`).

Ask: is this avoiding an N+1 query pattern?

## Card 6: Public Derived Model

Looks like:

```ts
// Illustrative fake code.
return publicSchema.parse({
  ...raw,
  status: computeStatus(raw.events),
  openEvents: filterOpen(raw.events),
});
```

Real anchor: `statusPage.get` returns computed public fields (`packages/api/src/router/statusPage.ts:231-400`).

Ask: should this be computed before or after React rendering?

## Card 7: Client Side Effect Warning

Looks like:

```ts
// Illustrative fake code.
onSuccess: async (created) => {
  await sendSecondaryEffect(created.id);
}
```

Real anchor: dashboard notification after create success (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`).

Ask: what happens if the secondary effect fails after the primary write succeeds?

## Card 8: Test Transaction

Looks like:

```ts
// Illustrative fake code.
await withTestTransaction(async (tx) => {
  await service({ ctx: { ...ctx, db: tx }, input });
});
```

Real anchor: `withTestTransaction` helper (`packages/services/test/helpers.ts:18-50`).

Ask: am I using `tx` for all reads/writes inside the test?

