# Mid-Level Engineer Guide

## Architecture Diagram

```text
Dashboard route
  apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:23-128
    |
    | React Query + typed tRPC helpers
    v
Dashboard tRPC client
  apps/dashboard/src/lib/trpc/client.tsx:39-64
  apps/dashboard/src/lib/trpc/shared.ts:46-76
    |
    | /api/trpc/edge unless router prefix is in lambdas
    v
Next route handler
  apps/status-page/src/app/api/trpc/edge/[trpc]/route.ts:11-24
    |
    | createTRPCContext(auth, req)
    v
API package
  packages/api/src/trpc.ts:59-119
  packages/api/src/router/statusReport.ts:58-222
    |
    | toServiceCtx(ctx)
    v
Services package
  packages/services/src/status-report/create.ts:24-99
  packages/services/src/status-report/list.ts:164-226
    |
    | Drizzle/libSQL
    v
DB schema
  packages/db/src/schema/status_reports/status_reports.ts:15-89
  packages/db/src/schema/page_components/page_components.ts:124-157
    |
    | public read path
    v
Public status page
  packages/api/src/router/statusPage.ts:65-400
  apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:56-414
```

## Type System Deep Dive

This codebase uses TypeScript contracts at several boundaries:

- App-to-API: `AppRouter` is inferred from the merged router (`packages/api/src/root.ts:1-6`) and used to type client procedure helpers (`apps/dashboard/src/lib/trpc/client.tsx:9-12`).
- tRPC input: router schemas preserve old wire shapes like `pageComponents` (`packages/api/src/router/statusReport.ts:24-32`) and adapt them to service inputs like `pageComponentIds` (`packages/api/src/router/statusReport.ts:64-74`).
- Service input: canonical service schemas are stricter and reusable outside tRPC (`packages/services/src/status-report/schemas.ts:12-92`).
- DB rows: Drizzle table declarations feed `drizzle-zod` select schemas and inferred row types (`packages/db/src/schema/status_reports/status_reports.ts:15-63`; `packages/db/src/schema/status_reports/validation.ts:38-55`).
- Public output: `statusPage.get` declares an output schema, so its in-memory transformations must parse into `selectPublicPageSchemaWithRelation` (`packages/api/src/router/statusPage.ts:76-77`; `packages/api/src/router/statusPage.ts:385-400`).

Annotated contract bridge (`packages/api/src/router/statusReport.ts:24-32`; `packages/api/src/router/statusReport.ts:64-74`):

```ts
const createStatusReportTRPCInput = z.object({
  title: z.string(), // UI wire contract allows any string here.
  status: CreateStatusReportInput.shape.status, // reuses canonical enum from service schema.
  pageId: z.number(),
  pageComponents: z.array(z.number()), // legacy dashboard field name.
  date: z.coerce.date(),
  message: z.string(),
  notifySubscribers: z.boolean().nullish(), // UI-only behavior, not part of service create.
});

const { initialUpdate } = await createStatusReport({
  ctx: toServiceCtx(ctx),
  input: {
    pageComponentIds: input.pageComponents, // adapter translates names at boundary.
  },
});
```

## State Management Deep Dive

The important state in the status-report page is remote server state cached by React Query. The page gets the current page and reports through `useQuery` (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:27-32`), creates a mutation with an `onSuccess` callback (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:36-51`), then refreshes local report data and invalidates page-list data (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:46-49`).

Annotated state transition (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:27-51`):

```tsx
const { data: statusReports, refetch } = useQuery(
  trpc.statusReport.list.queryOptions({ pageId: Number.parseInt(id) }),
); // server state: cache key includes procedure + pageId.

const createStatusReportMutation = useMutation(
  trpc.statusReport.create.mutationOptions({
    onSuccess: async (statusReport) => {
      if (statusReport.notifySubscribers) {
        await sendStatusReportUpdateMutation.mutateAsync({ id: statusReport.id }); // secondary side effect.
      }
      refetch(); // refreshes this page's list.
      queryClient.invalidateQueries({ queryKey: trpc.page.list.queryKey() }); // refreshes other page summaries.
    },
  }),
);
```

## API Contract Map

- `trpc.statusReport.create`: protected mutation; input is title, status, pageId, pageComponents, date, message, optional notify flag; returns initial update merged with notify flag (`packages/api/src/router/statusReport.ts:24-32`; `packages/api/src/router/statusReport.ts:58-81`).
- `trpc.statusReport.createStatusReportUpdate`: protected mutation; appends update and returns update merged with notify flag (`packages/api/src/router/statusReport.ts:34-41`; `packages/api/src/router/statusReport.ts:83-104`).
- `trpc.statusReport.updateStatus`: protected mutation; updates title/status/component associations (`packages/api/src/router/statusReport.ts:51-56`; `packages/api/src/router/statusReport.ts:125-142`).
- `trpc.statusReport.list`: protected query; accepts optional period/order/pageId and returns page-backed reports for dashboard (`packages/api/src/router/statusReport.ts:189-222`).
- `trpc.statusPage.get`: public query; accepts slug plus optional visual overrides and returns a public page object with computed status, events, trackers, reports, and maintenances (`packages/api/src/router/statusPage.ts:65-400`).

## Component Interaction Map

```text
Status reports page
  apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:60-128
    uses FormSheetStatusReport
      apps/dashboard/src/components/forms/status-report/sheet.tsx:22-74
        uses FormStatusReport
          apps/dashboard/src/components/forms/status-report/form.tsx:79-417
            uses react-hook-form, zodResolver, statusReportStatus enum
              apps/dashboard/src/components/forms/status-report/form.tsx:19-21
              apps/dashboard/src/components/forms/status-report/form.tsx:62-77
            uses CheckboxTree for affected components
              apps/dashboard/src/components/forms/status-report/form.tsx:353-379
    uses DataTable and nested UpdatesDataTable
      apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:112-124
```

## Full-Stack Feature Trace

1. The user opens the status-reports route. Its layout prefetches `trpc.statusReport.list` for the page id and hydrates the client tree (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/layout.tsx:16-35`).
2. The client page reads `page.get` and `statusReport.list` from React Query (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:23-32`).
3. The user submits `FormSheetStatusReport`; page-level `onSubmit` calls `createStatusReportMutation.mutateAsync` with title, status, pageId, pageComponents, date, message, and notify flag (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:75-103`).
4. The tRPC client sends this through `endingLink`, which routes `statusReport` to edge because it is not in the lambda prefix list (`apps/dashboard/src/lib/trpc/shared.ts:38-76`; `packages/api/src/edge.ts:25-49`).
5. The route handler checks `x-trpc-source`, builds context, and calls `edgeRouter` (`apps/status-page/src/app/api/trpc/edge/[trpc]/route.ts:11-24`).
6. `protectedProcedure` requires a session, resolves the active workspace from `workspace-slug`, and injects `user` and `workspace` into context (`packages/api/src/trpc.ts:148-200`).
7. `statusReport.create` validates input, adapts `pageComponents` to `pageComponentIds`, and calls `createStatusReport` (`packages/api/src/router/statusReport.ts:58-81`).
8. The service checks write scope, confirms the page belongs to the current workspace, validates component ids, inserts `status_report`, writes join rows, inserts the initial `status_report_update`, and emits audit rows (`packages/services/src/status-report/create.ts:24-99`).
9. The database stores report/update data in `status_report` and `status_report_update`; component associations live in `status_report_to_page_component` (`packages/db/src/schema/status_reports/status_reports.ts:15-63`; `packages/db/src/schema/page_components/page_components.ts:124-157`).
10. On success, the dashboard optionally sends subscriber notifications, refetches the report list, and invalidates `page.list` (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:36-51`).
11. Public users see the report because `statusPage.get` loads `statusReports` with updates and affected components, computes `openEvents`/`lastEvents`, and returns a parsed public object (`packages/api/src/router/statusPage.ts:80-113`; `packages/api/src/router/statusPage.ts:231-400`).
12. The public client maps reports into banner/feed props (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:165-210`; `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:329-360`).

## Diff Reading Exercise

Hypothetical change: "Move subscriber notification from dashboard `onSuccess` into `createStatusReport` service."

Review questions:

- Does the new service code preserve the current UI behavior where the notify flag is optional and defaults only in the form (`apps/dashboard/src/components/forms/status-report/form.tsx:96-105`; `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:93-101`)?
- Does the tRPC input still accept `notifySubscribers` without passing UI-only fields into canonical `CreateStatusReportInput` (`packages/api/src/router/statusReport.ts:24-32`; `packages/services/src/status-report/schemas.ts:12-20`)?
- Is the notification side effect inside the DB transaction or after commit? Existing service tests use transaction rollback and subscription spies, so side effects need careful placement (`packages/services/src/status-report/__tests__/status-report.test.ts:41-56`; `packages/services/src/status-report/create.ts:32-98`).
- Does a failure to notify fail the report creation or produce a retry path? Current UI `await`s the send mutation after create success, so report creation can succeed while notification failure is separately surfaced (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:38-44`).

## Non-Obvious Patterns

- Router adapters preserve legacy contracts while services become canonical. This is visible in `statusReportRouter` comments and field-name translation (`packages/api/src/router/statusReport.ts:21-23`; `packages/api/src/router/statusReport.ts:64-74`).
- Service functions accept `ctx.db`, allowing tests or nested calls to thread a transaction; `withTransaction` reuses a transaction when one already exists (`packages/services/src/context.ts:22-29`; `packages/services/src/context.ts:40-47`).
- Public status-page logic computes product concepts from raw rows: a report with no `to` becomes an open event, page status is derived from monitor/component/event status, and trackers group components for display (`packages/api/src/router/statusPage.ts:136-180`; `packages/api/src/router/statusPage.ts:222-360`).
- The app has two auth worlds: dashboard users and public page viewers (`apps/dashboard/src/lib/auth/index.ts:15-24`; `apps/status-page/src/lib/auth/index.ts:15-65`).

## Mid-Level Socratic Checkpoint

1. Why is `statusReport.create` in `edgeRouter` and not `lambdaRouter`?
2. Which values belong to the UI contract but not the service contract?
3. Where is workspace ownership enforced for create?
4. Why does `listStatusReports` enrich reports in batches?
5. What cache keys are refreshed after creating a report?
6. What is the risk of doing subscriber notification from client `onSuccess`?
7. Why does `guardTRPCSource` not count as authentication?
8. How would you explain the difference between `statusReport.list` and `statusPage.get`?

How to self-grade:

- Strong answers cite `packages/api/src/edge.ts:25-49`, `packages/api/src/lambda.ts:7-14`, `packages/api/src/router/statusReport.ts:24-32`, `packages/services/src/status-report/create.ts:32-46`, `packages/services/src/status-report/list.ts:66-161`, `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:36-51`, `apps/dashboard/src/lib/trpc/shared.ts:17-29`, and `packages/api/src/router/statusPage.ts:65-400`.
- Strong answers separate "contract", "transport", "business rule", "persistence", and "rendering".

