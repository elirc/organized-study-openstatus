# Observability And Hypotheses

## Hypothesis-First Debugging

Do not start by changing code. Start by writing a hypothesis that can be falsified.

Example:

```text
Symptom: public page does not show the new report.
Hypothesis: statusPage.get loads the report, but lastEvents does not include it.
Evidence to inspect: packages/api/src/router/statusPage.ts:239-247 and client feed filter.
Falsify by: logging report ids from statusReports and event ids from lastEvents.
```

This hypothesis points to the router's `lastEvents` computation (`packages/api/src/router/statusPage.ts:239-247`) and the client feed filter (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:329-360`).

## Where To Add Temporary Logs

Use logs sparingly and remove them after the investigation.

### Dashboard submit boundary

```tsx
// Temporary investigation only.
console.log("[status-report submit]", {
  pageId: id,
  title: values.title,
  status: values.status,
  selectedComponents: values.pageComponents.length,
});
```

Relevant real boundary: dashboard submit builds mutation payload (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:89-103`).

### Router adapter boundary

```ts
// Temporary investigation only.
console.log("[statusReport.create input]", {
  pageId: input.pageId,
  pageComponents: input.pageComponents,
  notifySubscribers: input.notifySubscribers,
});
```

Relevant real boundary: tRPC create adapts `pageComponents` to `pageComponentIds` (`packages/api/src/router/statusReport.ts:58-81`).

### Service ownership boundary

```ts
// Temporary investigation only.
console.log("[createStatusReport ownership]", {
  workspaceId: ctx.workspace.id,
  pageId: input.pageId,
  pageComponentIds: input.pageComponentIds,
});
```

Relevant real boundary: service checks page workspace and validates component ids (`packages/services/src/status-report/create.ts:32-52`).

### Public event boundary

```ts
// Temporary investigation only.
console.log("[statusPage events]", {
  statusReports: _page.statusReports.map((r) => r.id),
  lastEvents: lastEvents.map((e) => `${e.type}:${e.id}`),
  openEvents: openEvents.map((e) => `${e.type}:${e.id}`),
});
```

Relevant real boundaries: status reports load in `statusPage.get` (`packages/api/src/router/statusPage.ts:80-113`), and `lastEvents`/`openEvents` are computed later (`packages/api/src/router/statusPage.ts:239-266`).

## Reading Existing Observability Clues

The tRPC server helper has a dashboard server fetch log that reports whether auth session cookies exist (`apps/dashboard/src/lib/trpc/server.tsx:38-53`). The tRPC shared `createOnError` logs handler label and error (`apps/dashboard/src/lib/trpc/shared.ts:7-15`). The protected procedure tracks analytics only after successful requests and only when metadata requests it (`packages/api/src/trpc.ts:206-238`).

Interpretation:

- Missing session cookie logs point toward auth/session setup.
- tRPC handler errors point toward router or service failures.
- Analytics metadata is not a debug substitute; it runs after success (`packages/api/src/trpc.ts:206-238`).

## Hypothesis Cards

### Card 1: Blank Dashboard

```text
Hypothesis: React Query has an error, but UI returns null before displaying it.
Inspect: apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:27-55
Next action: temporarily destructure error/isLoading from both queries.
```

### Card 2: Wrong Workspace

```text
Hypothesis: workspace-slug cookie points to a workspace that does not own this page.
Inspect: apps/dashboard/src/proxy.ts:45-65 and packages/api/src/trpc.ts:171-200
Next action: inspect cookie value and resolved workspace id.
```

### Card 3: Public Event Missing

```text
Hypothesis: report has no statusReportUpdates, so feed filters it out.
Inspect: packages/api/src/router/statusPage.ts:84-91 and apps/status-page/.../client.tsx:329-346
Next action: check update count for that report in statusPage.get result.
```

### Card 4: Notification Skipped

```text
Hypothesis: workspace plan lacks status-subscribers.
Inspect: packages/services/src/status-report/notify.ts:51-55
Next action: inspect workspace.limits["status-subscribers"].
```

