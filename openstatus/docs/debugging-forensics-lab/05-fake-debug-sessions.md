# Fake Debug Sessions

These sessions model the internal monologue of debugging. The code snippets marked "fake" are teaching sketches.

## Session 1: "The Create Button Did Nothing"

Observation: user clicks "Create Status Report", sheet closes, no new row appears.

Step 1: inspect submit path.

```tsx
// Real anchor: apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:89-103
if ("date" in values) {
  await createStatusReportMutation.mutateAsync({ ... });
}
```

Hypothesis: `values` may be update-mode shape without `date`, so create branch is skipped.

Check: `FormStatusReport` chooses schema based on `defaultValues` (`apps/dashboard/src/components/forms/status-report/form.tsx:96-105`). `FormSheetStatusReport` passes optional `defaultValues` through (`apps/dashboard/src/components/forms/status-report/sheet.tsx:22-63`).

Fake debugging note:

```ts
// Illustrative fake code.
console.log("is create mode?", "date" in values, values);
```

Likely fix direction: ensure the create sheet is not accidentally passed `defaultValues`.

## Session 2: "The API Returns NOT_FOUND, But The Page Exists"

Observation: create returns NOT_FOUND.

Step 1: inspect service ownership lookup.

```ts
// Real anchor: packages/services/src/status-report/create.ts:32-40
.where(and(eq(page.id, input.pageId), eq(page.workspaceId, ctx.workspace.id)))
```

Hypothesis: the page exists but not in the active workspace.

Check:

- Active workspace comes from `workspace-slug` cookie (`packages/api/src/trpc.ts:171-200`).
- Dashboard proxy sets `workspace-slug` if missing (`apps/dashboard/src/proxy.ts:45-65`).
- Cross-workspace rejection is tested (`packages/api/src/router/statusReport.test.ts:77-99`).

Fake query:

```sql
-- Illustrative fake SQL.
select id, workspace_id, slug from page where id = :pageId;
```

Conclusion: NOT_FOUND may be correct tenant isolation behavior.

## Session 3: "The Public Status Page Still Looks Green"

Observation: an unresolved report exists, but public page status remains success.

Step 1: inspect event generation.

```ts
// Real anchor: packages/api/src/router/statusPage.utils.ts:205-258
reports
  .filter((report) => {
    if (pageComponentId) {
      return report.statusReportsToPageComponents.some(
        (r) => r.pageComponent?.id === pageComponentId,
      );
    }
    return true;
  })
```

Hypothesis: the report is not associated with the component being displayed.

Check: create service calls `updatePageComponentAssociations` after report insert (`packages/services/src/status-report/create.ts:65-69`). Join rows live in `status_report_to_page_component` (`packages/db/src/schema/page_components/page_components.ts:124-157`).

Fake query:

```sql
-- Illustrative fake SQL.
select * from status_report_to_page_component where status_report_id = :reportId;
```

Conclusion: public status can stay green if the report is not connected to the relevant page component.

## Session 4: "Tests Pass Alone But Fail Together"

Observation: service tests pass individually but fail in suite.

Step 1: inspect test isolation helpers.

```ts
// Real anchor: packages/services/test/helpers.ts:37-50
await db.transaction(async (tx) => {
  result = await fn(tx);
  throw new RollbackSignal();
});
```

Hypothesis: a test wrote committed rows instead of using `tx`.

Check helper comments: inside callback, replace `db.select`, `db.insert`, and `db.delete` with calls on `tx`; committed `db` reads will not see transaction writes (`packages/services/test/helpers.ts:29-36`).

Fake bad test:

```ts
// Illustrative fake code: incorrect inside withTestTransaction.
await withTestTransaction(async (tx) => {
  await db.insert(page).values(...); // wrong: committed db, not tx
});
```

Fake better test:

```ts
// Illustrative fake code.
await withTestTransaction(async (tx) => {
  await tx.insert(page).values(...); // correct: rolls back
});
```

