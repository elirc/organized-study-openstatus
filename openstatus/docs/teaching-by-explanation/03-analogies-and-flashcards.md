# Analogies And Flashcards

## Domain-Specific Analogies

### Dashboard As Control Room

The dashboard is the control room where workspace users decide what to publish. The status-reports page creates incident bulletins and updates (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:60-128`).

### Public Status Page As Lobby Display

The public status page is the lobby display. It does not let visitors edit the incident; it translates backend state into banners, component cards, and event feed entries (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:136-414`).

### Service Layer As Incident Commander

The service layer is the incident commander: it checks authority, validates affected services, writes the official bulletin, records the first update, and emits audit rows (`packages/services/src/status-report/create.ts:24-99`).

### Drizzle Tables As The Incident Ledger

`status_report` stores the current bulletin; `status_report_update` stores the timeline; `status_report_to_page_component` stores affected services (`packages/db/src/schema/status_reports/status_reports.ts:15-63`; `packages/db/src/schema/page_components/page_components.ts:124-157`).

### tRPC Router As Dispatcher

The router receives the call, validates the message, translates names, and dispatches to services (`packages/api/src/router/statusReport.ts:24-81`).

## Flashcards

Q: Where is the root package manager declared?  
A: `packageManager` is `pnpm@10.26.0` (`package.json:33-33`).

Q: Where does the dashboard tRPC client get its router type?  
A: It imports `AppRouter` from `@openstatus/api` (`apps/dashboard/src/lib/trpc/client.tsx:9-12`), which is inferred from the merged app router (`packages/api/src/root.ts:1-6`).

Q: Which router owns status reports?  
A: `statusReportRouter` is mounted on `edgeRouter` as `statusReport` (`packages/api/src/edge.ts:18-32`).

Q: Which code adapts `pageComponents` to `pageComponentIds`?  
A: `statusReport.create` in the tRPC router (`packages/api/src/router/statusReport.ts:64-74`).

Q: Which function writes report and initial update rows?  
A: `createStatusReport` (`packages/services/src/status-report/create.ts:24-99`).

Q: Which DB table stores report timeline messages?  
A: `status_report_update` (`packages/db/src/schema/status_reports/status_reports.ts:41-63`).

Q: Where is workspace ownership checked during report create?  
A: The service selects a page by both `page.id` and `page.workspaceId` (`packages/services/src/status-report/create.ts:32-40`).

Q: Where does public status-page rendering get feed reports?  
A: It maps `page.statusReports` filtered by `page.lastEvents` into `StatusFeed` props (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:329-360`).

Q: What is the known notification reliability wrinkle?  
A: The dashboard sends subscriber notification in mutation `onSuccess`, with a TODO to move it server-side (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`).

Q: Which helper maps service errors to tRPC errors?  
A: `toTRPCError` (`packages/api/src/service-adapter.ts:32-87`).

## Fake Code Memory Hooks

Illustrative fake code:

```ts
// Router memory hook:
const apiInput = parseClientPayload(request.body);
const serviceInput = adaptToDomain(apiInput);
return serviceFunction({ ctx: toServiceCtx(ctx), input: serviceInput });
```

Real anchor: `statusReport.create` validates tRPC input, adapts field names, and calls `createStatusReport` (`packages/api/src/router/statusReport.ts:58-81`).

Illustrative fake code:

```ts
// Service memory hook:
requirePermission(ctx);
await withTransaction(ctx, async (tx) => {
  await proveTenantOwnership(tx, input.id, ctx.workspace.id);
  await writeDomainRows(tx, input);
  await writeAuditRows(tx, ctx);
});
```

Real anchor: `createStatusReport` checks scope, page ownership, component ids, writes rows, and emits audits (`packages/services/src/status-report/create.ts:24-99`).

