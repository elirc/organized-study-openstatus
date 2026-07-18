# Sequence Diagrams

## Sequence 1: Dashboard Creates A Status Report

```text
Dashboard Page
  -> FormSheetStatusReport
    -> FormStatusReport
      -> createStatusReportMutation
        -> tRPC endingLink
          -> /api/trpc/edge
            -> statusReportRouter.create
              -> toServiceCtx
                -> createStatusReport service
                  -> Drizzle transaction
                    -> insert status_report
                    -> insert status_report_to_page_component rows
                    -> insert status_report_update
                    -> emit audit rows
              <- initialUpdate + notifySubscribers
      <- onSuccess refetch + invalidate page list
```

Annotated real code excerpt:

```tsx
// apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:89-103
onSubmit={async (values) => {
  if ("date" in values) { // create mode includes date/message/notify fields.
    await createStatusReportMutation.mutateAsync({
      title: values.title,
      status: values.status,
      pageId: Number.parseInt(id), // route param becomes backend page id.
      pageComponents: values.pageComponents, // tRPC wire name.
      date: values.date,
      message: values.message,
      notifySubscribers: values.notifySubscribers,
    });
  }
}}
```

Annotated real code excerpt:

```ts
// packages/api/src/router/statusReport.ts:64-77
const { initialUpdate } = await createStatusReport({
  ctx: toServiceCtx(ctx), // converts tRPC ctx into ServiceContext.
  input: {
    title: input.title,
    status: input.status,
    pageId: input.pageId,
    pageComponentIds: input.pageComponents, // adapter renames field.
    date: input.date,
    message: input.message,
  },
});
return { ...initialUpdate, notifySubscribers: input.notifySubscribers };
// Dashboard currently expects the initial update row plus notify flag.
```

## Sequence 2: Protected Procedure Resolves Workspace

```text
tRPC route handler
  -> createTRPCContext
    -> session from auth()
    -> db added by createInnerTRPCContext
  -> protectedProcedure middleware
    -> check session.user.id
    -> read workspace-slug cookie
    -> resolveActiveWorkspace(userId, workspaceSlug)
    -> call resolver with ctx.user and ctx.workspace
```

Real code anchors:

- `createTRPCContext` reads session and request metadata, then calls `createInnerTRPCContext` (`packages/api/src/trpc.ts:71-94`).
- `createInnerTRPCContext` adds `db` to the context (`packages/api/src/trpc.ts:59-64`).
- `enforceUserIsAuthed` checks session, reads `workspace-slug`, resolves the workspace, and passes `user` and `workspace` into downstream context (`packages/api/src/trpc.ts:148-200`).

Illustrative fake code:

```ts
// Illustrative fake code: mental model for protectedProcedure.
async function protectedProcedureResolver(request) {
  const session = await auth(request);
  if (!session?.user?.id) throw new Error("UNAUTHORIZED");

  const workspaceSlug = request.cookies["workspace-slug"];
  const { user, workspace } = await resolveActiveWorkspace(session.user.id, workspaceSlug);

  return runActualProcedure({ user, workspace, db });
}
```

## Sequence 3: Public Status Page Renders Report Data

```text
Public Client
  -> trpc.statusPage.get({ slug: domain })
    -> publicProcedure
      -> db.query.page.findFirst with relations
      -> parse workspace/page/components
      -> compute component statuses
      -> compute pageEvents
      -> derive lastEvents and openEvents
      -> group trackers
      -> sort reports and maintenances
      -> parse public output schema
  <- page object
  -> render StatusBannerTabs or StatusBanner
  -> render ComponentCard list
  -> render StatusFeed
```

Real code anchors:

- Client query starts at `trpc.statusPage.get.queryOptions({ slug: domain })` (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:65-79`).
- Router loads page relations with status reports, updates, maintenances, page components, monitors, incidents, and groups (`packages/api/src/router/statusPage.ts:80-113`).
- Router computes component status and page status (`packages/api/src/router/statusPage.ts:136-229`).
- Router computes `lastEvents`, `openEvents`, grouped trackers, sorted reports, and parsed output (`packages/api/src/router/statusPage.ts:231-400`).
- Client renders banner content for report events (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:143-210`) and feed entries (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:329-360`).

## Sequence 4: Report List Query Avoids N+1

```text
listStatusReports
  -> count matching reports
  -> select matching report rows
  -> enrichReportsBatch(rows)
      -> one query for all updates
      -> one query for all component associations
      -> two parallel queries for pages and sibling components
      -> map rows into StatusReportWithRelations
```

Annotated real code excerpt:

```ts
// packages/services/src/status-report/list.ts:83-95
const allUpdates = await db
  .select()
  .from(statusReportUpdate)
  .where(inArray(statusReportUpdate.statusReportId, reportIds))
  .orderBy(desc(statusReportUpdate.date))
  .all();
const updatesByReport = new Map<number, StatusReportUpdate[]>();
// One query gets updates for every report in the current page.
// The map groups them in memory without re-querying per report.
```

Why it matters: dashboard tRPC currently asks for a high ceiling of 10,000 reports because the UI has no paging today (`packages/api/src/router/statusReport.ts:199-218`). A per-row relation query would become expensive quickly, so batched enrichment is the right shape (`packages/services/src/status-report/list.ts:66-161`).

## Sequence 5: Client-Owned Notification Side Effect

```text
createStatusReportMutation succeeds
  |
  | returned statusReport.notifySubscribers?
  | yes
  v
sendStatusReportUpdateMutation.mutateAsync({ id })
  |
  v
refetch report list
invalidate page list query
```

The dashboard currently performs this side effect in `onSuccess`, and the code has a TODO to move it server-side (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`). That means report creation can succeed even if notification later fails. The service already owns audit side effects inside the transaction (`packages/services/src/status-report/create.ts:82-95`), which gives a useful contrast when deciding future ownership.

