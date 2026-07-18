# Fake Code Contrasts

These examples are deliberately fake. Each contrast shows a common engineering idea in a tiny version, then points back to the real OpenStatus implementation.

## Contrast 1: UI Contract vs Service Contract

Bad fake code:

```ts
// Illustrative fake code: not from repo.
async function createReport(inputFromReactForm) {
  await db.insert(statusReport).values(inputFromReactForm);
}
```

Problem: the database is now coupled to the React form shape. If the form uses `pageComponents` but the DB needs join rows, the design collapses.

Better fake code:

```ts
// Illustrative fake code: not from repo.
const serviceInput = {
  title: form.title,
  pageComponentIds: form.pageComponents,
};
await createStatusReport({ ctx, input: serviceInput });
```

Real pattern: the tRPC router adapts `pageComponents` to `pageComponentIds` (`packages/api/src/router/statusReport.ts:64-74`), and the service schema owns canonical `pageComponentIds` (`packages/services/src/status-report/schemas.ts:12-20`).

## Contrast 2: Per-Row Queries vs Batched Enrichment

Bad fake code:

```ts
// Illustrative fake code: not from repo.
for (const report of reports) {
  report.updates = await db.select().from(updates).where(eq(updates.reportId, report.id));
}
```

Problem: 100 reports can become 101+ queries.

Better fake code:

```ts
// Illustrative fake code: not from repo.
const reportIds = reports.map((report) => report.id);
const allUpdates = await db.select().from(updates).where(inArray(updates.reportId, reportIds));
const updatesByReport = groupBy(allUpdates, "reportId");
```

Real pattern: `enrichReportsBatch` uses one query for all updates, one query for all component associations, and page queries keyed by distinct page ids (`packages/services/src/status-report/list.ts:66-161`).

## Contrast 3: Header Guard vs Authorization

Bad fake code:

```ts
// Illustrative fake code: not from repo.
if (request.headers["x-client"] === "dashboard") {
  allowWrite();
}
```

Problem: any caller can spoof a header.

Better fake code:

```ts
// Illustrative fake code: not from repo.
const session = await auth(request);
if (!session?.user?.id) throw unauthorized();
const workspace = await resolveWorkspace(session.user.id, request.cookies.workspaceSlug);
```

Real pattern: `guardTRPCSource` is documented as a spoofable convention filter (`apps/dashboard/src/lib/trpc/shared.ts:17-29`), while `protectedProcedure` checks session and resolves workspace (`packages/api/src/trpc.ts:148-200`).

## Contrast 4: Client-Owned Side Effect vs Server-Owned Side Effect

Current-like fake code:

```ts
// Illustrative fake code: resembles current pattern, not copied.
const created = await createReport(values);
if (values.notify) {
  await sendEmail(created.id);
}
```

Risk: if the browser closes after creation, notification may not happen.

Server-owned fake code:

```ts
// Illustrative fake code: possible future pattern.
await createReport({
  ...values,
  afterCommit: values.notify ? "queueNotification" : undefined,
});
```

Real situation: dashboard currently sends notification after create success and the code says `TODO: move to server` (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`). Service tests already use subscription spies that could support server-owned notification tests (`packages/services/src/status-report/__tests__/status-report.test.ts:41-56`).

## Contrast 5: Silent Blank UI vs Explicit State

Current real pattern:

```tsx
// apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:54-55
if (!statusReports || !page) return null;
```

Risk: loading, missing data, and error states can all look like empty space.

Better fake code:

```tsx
// Illustrative fake code: not from repo.
if (isLoading) return <StatusReportsSkeleton />;
if (error) return <ErrorState message="Could not load status reports." />;
if (!page) return <NotFoundState />;
```

Real next step: inspect current query return values in the page (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:27-32`) and introduce loading/error UI consistent with dashboard components.

## Contrast 6: Raw DB Rows vs Public Read Model

Bad fake code:

```ts
// Illustrative fake code: not from repo.
return await db.select().from(statusReport);
```

Problem: public UI needs affected component names, update arrays, open events, and page-level status, not just rows.

Better fake code:

```ts
// Illustrative fake code: not from repo.
return {
  statusReports: reportsWithUpdates,
  openEvents: computeOpenEvents(reportsWithUpdates),
  trackers: groupComponents(pageComponents),
};
```

Real pattern: `statusPage.get` loads relations, computes statuses/events/trackers, and parses a public output object (`packages/api/src/router/statusPage.ts:80-400`).

