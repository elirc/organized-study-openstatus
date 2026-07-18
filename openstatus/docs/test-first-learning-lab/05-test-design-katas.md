# Test Design Katas

## Kata 1: Add Status Filter To Report List

Feature premise: dashboard should filter status reports by status.

Existing facts:

- Service schema already supports `statuses` (`packages/services/src/status-report/schemas.ts:79-87`).
- Service list applies `inArray(statusReport.status, input.statuses)` when statuses are provided (`packages/services/src/status-report/list.ts:172-175`).
- tRPC list currently does not expose statuses and passes `statuses: []` (`packages/api/src/router/statusReport.ts:189-211`).

Write the test first:

```ts
// Illustrative fake code.
test("statusReport.list filters by statuses", async () => {
  const caller = edgeRouter.createCaller(ctx);
  const reports = await caller.statusReport.list({
    pageId,
    statuses: ["resolved"],
  });
  expect(reports.every((report) => report.status === "resolved")).toBe(true);
});
```

Expected first failure: tRPC input rejects `statuses`.

## Kata 2: Move Notification Server-Side

Feature premise: report notification should not depend on browser `onSuccess`.

Existing facts:

- Dashboard currently sends notification after create success (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`).
- `notifyStatusReport` dispatches a specific update and checks workspace ownership/plan (`packages/services/src/status-report/notify.ts:20-58`).
- Service tests expose subscription spies (`packages/services/src/status-report/__tests__/status-report.test.ts:41-56`).

Write the test first:

```ts
// Illustrative fake code.
test("create with notify dispatches after creating initial update", async () => {
  await withTestTransaction(async (tx) => {
    const ctx = { ...teamCtx, db: tx };
    const result = await createStatusReportAndNotify({
      ctx,
      input: { ...validInput, notifySubscribers: true },
    });
    expect(result.initialUpdate.id).toBeGreaterThan(0);
    expect(subscriptionSpies.dispatchStatusReportUpdate.mock.calls[0][0]).toBe(
      result.initialUpdate.id,
    );
  });
});
```

Design question: should this be a new service, a post-commit job, or a router orchestration? The answer changes the test layer.

## Kata 3: Add Loading/Error State To Dashboard Reports

Feature premise: dashboard should not return blank UI while reports/page query state is unknown.

Existing facts:

- Page currently returns null if reports or page is absent (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:54-55`).
- Queries are created in the client page (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:27-32`).
- Layout prefetches report list (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/layout.tsx:16-35`).

Test approach:

```tsx
// Illustrative fake code.
it("shows loading state while reports load", () => {
  mockUseQueryOnce({ isLoading: true, data: undefined });
  render(<Page />);
  expect(screen.getByText(/loading/i)).toBeInTheDocument();
});
```

Design question: does this codebase already have a preferred dashboard skeleton or empty-state component? Search `Skeleton`, `EmptyState`, and data-table loading patterns before inventing one.

## Kata 4: Public Status For Static Components Ignores Incidents

Existing fact: `getEvents` skips incidents when `componentType === "static"` (`packages/api/src/router/statusPage.utils.ts:185-203`).

Write the test first:

```ts
// Illustrative fake code.
test("static component does not inherit monitor incidents", () => {
  const events = getEvents({
    maintenances: [],
    incidents: [{ monitorId: 123, createdAt: new Date(), resolvedAt: null }],
    reports: [],
    pageComponentId: 99,
    componentType: "static",
  });
  expect(events).toEqual([]);
});
```

Expected learning: public component type changes event eligibility. Static components are manually reported, not monitor-backed (`packages/db/src/schema/page_components/page_components.ts:20-63`; `packages/api/src/router/statusPage.utils.ts:185-203`).

## Kata 5: Audit Row For Report Update

Existing facts:

- `createStatusReport` emits audit rows for report create and update create (`packages/services/src/status-report/create.ts:82-95`).
- `expectAuditRow` helper checks action/entity rows (`packages/services/test/helpers.ts:240-264`).

Write the test first:

```ts
// Illustrative fake code.
await expectAuditRow({
  workspaceId: ctx.workspace.id,
  action: "status_report_update.create",
  entityType: "status_report_update",
  entityId: initialUpdate.id,
  db: tx,
});
```

Expected learning: audit is not a UI feature; it is a service-owned side effect tied to domain writes.

