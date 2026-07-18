# Symptom Index

## Symptom 1: The Dashboard Status Reports Page Is Blank

Likely first files:

- `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:23-55`
- `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/layout.tsx:16-35`
- `apps/dashboard/src/lib/trpc/server.tsx:99-117`

Why: the page returns `null` if either `statusReports` or `page` is absent (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:54-55`). The layout prefetches `statusReport.list` before hydration (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/layout.tsx:16-35`), and the parent `[id]` layout calls `fetchQueryOrNotFound` for missing pages (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/layout.tsx:28-31`; `apps/dashboard/src/lib/trpc/server.tsx:99-117`).

Debug shape:

```tsx
// Real anchor: apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:27-55
const { data: page } = useQuery(trpc.page.get.queryOptions({ id: Number.parseInt(id) }));
const { data: statusReports } = useQuery(trpc.statusReport.list.queryOptions({ pageId: Number.parseInt(id) }));

if (!statusReports || !page) return null;
// Symptom: loading, error, and missing data are visually indistinguishable here.
```

Illustrative fake code:

```tsx
// Illustrative fake code: better debugging UI, not current repo code.
if (pageQuery.isLoading || reportsQuery.isLoading) return <LoadingState />;
if (pageQuery.error || reportsQuery.error) return <ErrorState />;
if (!page) return <MissingPageState />;
```

## Symptom 2: Creating A Status Report Works, But Subscribers Are Not Notified

Likely first files:

- `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`
- `packages/services/src/status-report/notify.ts:10-58`
- `packages/services/src/status-report/__tests__/status-report.test.ts:41-56`

Why: notification is a second client-side mutation after create succeeds (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`). The service notification function enforces workspace ownership and skips if the workspace plan lacks `status-subscribers` (`packages/services/src/status-report/notify.ts:20-58`).

Investigation prompts:

1. Did `statusReport.create` return `notifySubscribers: true`? The router merges that flag into the initial update return (`packages/api/src/router/statusReport.ts:75-77`).
2. Did the client call `sendStatusReportUpdateMutation`? The call is awaited in `onSuccess` (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:38-44`).
3. Does the workspace plan allow subscribers? `notifyStatusReport` returns without dispatch if `status-subscribers` is disabled (`packages/services/src/status-report/notify.ts:51-55`).

## Symptom 3: Public Status Page Does Not Show A Newly Created Report

Likely first files:

- `packages/api/src/router/statusPage.ts:80-113`
- `packages/api/src/router/statusPage.ts:239-266`
- `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:329-360`

Why: the public feed shows reports only when they have updates and match `page.lastEvents` (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:329-346`). The router computes `lastEvents` from page events (`packages/api/src/router/statusPage.ts:239-247`).

Debug checklist:

```text
Does statusPage.get load statusReports with statusReportUpdates? packages/api/src/router/statusPage.ts:84-91
Does getEvents produce a report event? packages/api/src/router/statusPage.utils.ts:205-258
Does lastEvents include it? packages/api/src/router/statusPage.ts:239-247
Does StatusFeed map it? apps/status-page/.../client.tsx:329-360
```

## Symptom 4: A Cross-Workspace Report Mutation Is Rejected

Likely first files:

- `packages/services/src/status-report/create.ts:32-52`
- `packages/api/src/router/statusReport.test.ts:77-207`
- `packages/api/src/service-adapter.ts:41-48`

Why: this is expected behavior. The service looks up the page by both `page.id` and `page.workspaceId` (`packages/services/src/status-report/create.ts:32-40`) and validates page component ids in the same workspace/page context (`packages/services/src/status-report/create.ts:42-52`). Router tests intentionally assert cross-workspace rejection (`packages/api/src/router/statusReport.test.ts:77-207`).

Debug warning: do not "fix" this by loosening filters. The safer public behavior is to make resources in other workspaces appear missing or forbidden depending on boundary (`packages/api/src/router/statusReport.test.ts:200-207`; `packages/api/src/service-adapter.ts:41-48`).

## Symptom 5: Public Page Feels Slow

Likely first files:

- `packages/api/src/router/statusPage.ts:49-55`
- `packages/api/src/router/statusPage.ts:80-400`
- `packages/services/src/status-report/list.ts:66-161`

Why: the public router has a TODO about SQL performance and loads a broad relation graph before performing in-memory transformations (`packages/api/src/router/statusPage.ts:49-113`; `packages/api/src/router/statusPage.ts:136-400`). Compare that to the status-report service list, which uses batched enrichment to avoid N+1 behavior (`packages/services/src/status-report/list.ts:66-161`).

Illustrative fake code:

```ts
// Illustrative fake code: slow shape.
const page = await loadEverythingForPage(slug);
const events = computeEverything(page);

// Illustrative fake code: faster shape when sections are hidden.
const shell = await loadPageShell(slug);
const events = showFeed ? await loadEvents(shell.id) : [];
const trackers = showComponents ? await loadTrackers(shell.id) : [];
```

