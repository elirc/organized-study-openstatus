# Bug Reproduction Scenarios

These scenarios are practice labs. They do not require modifying production code.

## Scenario 1: Cross-Workspace Page Injection

User-visible symptom: a workspace user attempts to create a report for a page id that belongs to another workspace and receives a not-found style error.

Real test anchor: router tests seed another workspace page and assert `statusReport.create` rejects it (`packages/api/src/router/statusReport.test.ts:18-60`; `packages/api/src/router/statusReport.test.ts:77-99`).

Reproduction outline:

```ts
// Illustrative fake code based on real test shape.
const ctx = createInnerTRPCContext({
  session: { user: { id: "1" } },
  workspace: { id: 1 },
});
const caller = edgeRouter.createCaller(ctx);

await caller.statusReport.create({
  title: "Cross workspace attempt",
  status: "investigating",
  pageId: otherWorkspacePageId,
  pageComponents: [],
  date: new Date(),
  message: "Should fail",
});
```

Expected guard: service checks page ownership by workspace id (`packages/services/src/status-report/create.ts:32-40`).

## Scenario 2: Report Created But Email Not Sent

User-visible symptom: dashboard shows a new incident report, but subscribers do not receive email.

Real code anchors:

- Client calls notification after create success (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`).
- Notification service silently skips when `status-subscribers` is disabled (`packages/services/src/status-report/notify.ts:51-55`).
- Service tests expose subscription dispatch spies (`packages/services/src/status-report/__tests__/status-report.test.ts:41-56`).

Reproduction outline:

```text
1. Use a workspace without status-subscribers enabled.
2. Create report with notifySubscribers checked.
3. Confirm create succeeds.
4. Confirm dispatchStatusReportUpdate is not called.
5. Decide whether this is expected plan behavior or UX confusion.
```

## Scenario 3: Public Feed Hides Report With No Updates

User-visible symptom: a report exists in the backend but does not appear in public feed.

Real code anchor: public feed filters out reports with zero updates (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:329-346`). Service create normally creates an initial update, so this symptom implies either legacy/orphan data or a write-path bug (`packages/services/src/status-report/create.ts:71-80`).

Reproduction outline:

```ts
// Illustrative fake setup only.
const report = { id: 1, statusReportUpdates: [] };
const visible = report.statusReportUpdates.length > 0;
// visible === false
```

Expected investigation: check whether `status_report_update` exists for the report (`packages/db/src/schema/status_reports/status_reports.ts:41-63`).

## Scenario 4: Status Page Shows Incident Banner For Only One Incident

User-visible symptom: multiple incidents are active, but only one incident-style banner is shown.

Real code anchor: the public client filters incident events so only the first incident is retained for banner display (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:117-129`).

Reproduction outline:

```ts
// Illustrative fake code matching the real logic.
let hasIncident = false;
const events = openEvents.filter((event) => {
  if (event.type !== "incident") return true;
  if (hasIncident) return false;
  hasIncident = true;
  return true;
});
```

Expected conclusion: this may be intended simplification, not a backend bug. The comment says a single incident is enough for the banner (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:117-118`).

## Scenario 5: Sorting Looks Wrong For Reports Without Updates

User-visible symptom: reports with no updates appear in unexpected positions.

Real code anchor: public `statusReports` sort handles empty update arrays with suspicious comparator symmetry (`packages/api/src/router/statusPage.ts:364-379`).

Reproduction outline:

```ts
// Illustrative fake code: pin the comparator behavior.
const reports = [
  { id: 1, updates: [{ date: new Date("2026-01-01") }] },
  { id: 2, updates: [] },
  { id: 3, updates: [{ date: new Date("2026-02-01") }] },
];
```

Expected next step: write a focused test around desired ordering before changing the comparator.

