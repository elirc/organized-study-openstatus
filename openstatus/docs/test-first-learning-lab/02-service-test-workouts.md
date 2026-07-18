# Service Test Workouts

## Workout 1: Create Report Happy Path

Existing anchor: service test creates a report, initial update, association, and audit rows (`packages/services/src/status-report/__tests__/status-report.test.ts:116-163`).

Behavior to learn:

```text
Given a valid workspace page and page component
When createStatusReport runs
Then it creates:
  status_report row
  status_report_update row
  status_report_to_page_component row
  audit rows for report and update
```

Real test shape:

```ts
// packages/services/src/status-report/__tests__/status-report.test.ts:118-130
await withTestTransaction(async (tx) => {
  const ctx = { ...teamCtx, db: tx };
  const { statusReport: report, initialUpdate } = await createStatusReport({
    ctx,
    input: {
      title: `${TEST_PREFIX}-create-happy`,
      status: "investigating",
      message: "starting investigation",
      date: new Date(),
      pageId: testPageId,
      pageComponentIds: [testPageComponentId],
    },
  });
});
```

Practice extension: add an assertion that `initialUpdate.status` equals the input status. Check current test already asserts status report id and message (`packages/services/src/status-report/__tests__/status-report.test.ts:132-145`).

## Workout 2: Cross-Workspace Page Rejection

Existing anchor: service test expects `NotFoundError` when free workspace tries to create a report on team workspace page (`packages/services/src/status-report/__tests__/status-report.test.ts:165-180`).

Why this belongs in service tests: the service owns the workspace page lookup (`packages/services/src/status-report/create.ts:32-40`).

Illustrative fake test:

```ts
// Illustrative fake code.
await expect(
  createStatusReport({
    ctx: { ...wrongWorkspaceCtx, db: tx },
    input: { pageId: otherWorkspacePageId, pageComponentIds: [] },
  }),
).rejects.toBeInstanceOf(NotFoundError);
```

Practice extension: add a similar service test for page component ids from another workspace, mirroring router coverage (`packages/api/src/router/statusReport.test.ts:181-207`).

## Workout 3: Read-Only Actor Rejection

Existing anchor: service test creates an API-key context with `scopes: ["read"]` and expects `ForbiddenError` (`packages/services/src/status-report/__tests__/status-report.test.ts:183-207`).

Real helper: `makeApiKeyCtx` creates an `apiKey` actor with scopes (`packages/services/test/helpers.ts:132-151`). Real authorization function checks `apiKey` and `mcp` actors (`packages/services/src/auth/require-scope.ts:25-48`).

Illustrative fake test:

```ts
// Illustrative fake code.
const ctx = {
  ...makeApiKeyCtx(workspace, { keyId: "read-only", scopes: ["read"] }),
  db: tx,
};
await expect(writeService({ ctx, input })).rejects.toBeInstanceOf(ForbiddenError);
```

Practice extension: write the same test for `makeMcpCtx`, since `requireScope` treats `mcp` like `apiKey` (`packages/services/src/auth/require-scope.ts:25-30`; `packages/services/test/helpers.ts:153-172`).

## Workout 4: Notification Plan Gate

Existing service behavior: `notifyStatusReport` returns without dispatch if `ctx.workspace.limits["status-subscribers"]` is false (`packages/services/src/status-report/notify.ts:51-55`). Tests have subscription spies available on `globalThis` (`packages/services/src/status-report/__tests__/status-report.test.ts:41-56`).

Illustrative fake test:

```ts
// Illustrative fake code.
test("does not dispatch when plan lacks subscribers", async () => {
  await withTestTransaction(async (tx) => {
    const ctx = { ...freeCtx, db: tx };
    await notifyStatusReport({ ctx, input: { statusReportUpdateId } });
    expect(subscriptionSpies.dispatchStatusReportUpdate.mock.calls).toHaveLength(0);
  });
});
```

Practice question: should plan-gated notification be silent or user-visible? The current service comment says the silent skip matches existing email-router behavior (`packages/services/src/status-report/notify.ts:51-53`).

