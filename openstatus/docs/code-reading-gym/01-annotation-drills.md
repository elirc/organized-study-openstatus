# Annotation Drills

Each drill gives you a real code excerpt and asks you to annotate it. Write your annotations in a scratch file or comments, then compare with the guide.

## Drill 1: Annotate A React Query Mutation

Source: `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`

```tsx
const sendStatusReportUpdateMutation = useMutation(
  trpc.emailRouter.sendStatusReport.mutationOptions(),
);
const createStatusReportMutation = useMutation(
  trpc.statusReport.create.mutationOptions({
    onSuccess: async (statusReport) => {
      // TODO: move to server
      if (statusReport.notifySubscribers) {
        await sendStatusReportUpdateMutation.mutateAsync({
          id: statusReport.id,
        });
      }
      //
      refetch();
      queryClient.invalidateQueries({
        queryKey: trpc.page.list.queryKey(),
      });
    },
  }),
);
```

Your annotation tasks:

1. Mark the primary mutation.
2. Mark the secondary side effect.
3. Mark the cache refresh.
4. Mark the reliability risk.

Model annotation:

```tsx
const sendStatusReportUpdateMutation = useMutation(
  trpc.emailRouter.sendStatusReport.mutationOptions(),
); // Secondary mutation: sends subscriber notification after the report exists.

const createStatusReportMutation = useMutation(
  trpc.statusReport.create.mutationOptions({
    onSuccess: async (statusReport) => { // Runs only after create succeeds.
      // TODO: move to server
      if (statusReport.notifySubscribers) { // UI receives notify flag from router return.
        await sendStatusReportUpdateMutation.mutateAsync({
          id: statusReport.id, // The id is the initial update id returned by create.
        });
      }
      refetch(); // Refresh current report list query.
      queryClient.invalidateQueries({
        queryKey: trpc.page.list.queryKey(), // Refresh page list summaries elsewhere.
      });
    },
  }),
);
```

Why this matters: the TODO means the current design knowingly leaves notification reliability in the browser path (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:38-44`).

## Drill 2: Annotate The Router Adapter

Source: `packages/api/src/router/statusReport.ts:58-81`

```ts
create: protectedProcedure
  .meta({ track: Events.CreateReport })
  .input(createStatusReportTRPCInput)
  .mutation(async ({ ctx, input }) => {
    try {
      const { initialUpdate } = await createStatusReport({
        ctx: toServiceCtx(ctx),
        input: {
          title: input.title,
          status: input.status,
          pageId: input.pageId,
          pageComponentIds: input.pageComponents,
          date: input.date,
          message: input.message,
        },
      });
      return { ...initialUpdate, notifySubscribers: input.notifySubscribers };
    } catch (err) {
      toTRPCError(err);
    }
  }),
```

Your annotation tasks:

1. Mark transport-level validation.
2. Mark auth boundary.
3. Mark service boundary.
4. Mark legacy compatibility.
5. Mark error mapping.

Model annotation:

```ts
create: protectedProcedure // Authenticated user + workspace are required before resolver runs.
  .meta({ track: Events.CreateReport }) // Analytics metadata consumed after success in middleware.
  .input(createStatusReportTRPCInput) // Runtime validation for dashboard wire shape.
  .mutation(async ({ ctx, input }) => {
    try {
      const { initialUpdate } = await createStatusReport({
        ctx: toServiceCtx(ctx), // Converts tRPC context to service context.
        input: {
          pageComponentIds: input.pageComponents, // Adapts dashboard field to service field.
        },
      });
      return { ...initialUpdate, notifySubscribers: input.notifySubscribers };
      // Preserves dashboard return shape while service stays canonical.
    } catch (err) {
      toTRPCError(err); // Prevents raw service/DB errors from leaking.
    }
  }),
```

Related evidence: `protectedProcedure` is built from `enforceUserIsAuthed` (`packages/api/src/trpc.ts:148-264`), and `toTRPCError` maps service errors into tRPC errors (`packages/api/src/service-adapter.ts:32-87`).

## Drill 3: Annotate The Transaction

Source: `packages/services/src/status-report/create.ts:32-98`

```ts
return withTransaction(ctx, async (tx) => {
  const page_ = await tx
    .select({ id: page.id })
    .from(page)
    .where(
      and(eq(page.id, input.pageId), eq(page.workspaceId, ctx.workspace.id)),
    )
    .get();
  if (!page_) throw new NotFoundError("page", input.pageId);

  const validated = await validatePageComponentIds({
    tx,
    workspaceId: ctx.workspace.id,
    pageComponentIds: input.pageComponentIds,
  });

  const newReport = await tx.insert(statusReport).values({ ... }).returning().get();
  await updatePageComponentAssociations({ tx, statusReportId: newReport.id, componentIds: validated.componentIds });
  const initialUpdate = await tx.insert(statusReportUpdate).values({ ... }).returning().get();
  await emitAudit(tx, ctx, { action: "status_report.create", ... });
  return { statusReport: newReport, initialUpdate };
});
```

Your annotation tasks:

1. Mark tenant ownership check.
2. Mark component validation.
3. Mark core row insert.
4. Mark relationship write.
5. Mark timeline write.
6. Mark audit write.

Model explanation: this function treats report creation like one incident-room action. Either the bulletin, affected components, first update, and audit trail are all written together, or the transaction rolls back (`packages/services/src/context.ts:40-47`; `packages/services/src/status-report/create.ts:32-98`).

## Drill 4: Annotate Public Rendering

Source: `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:329-360`

```tsx
<StatusFeed
  statusReports={page.statusReports
    .filter(
      (report) =>
        report.statusReportUpdates.length > 0 &&
        page.lastEvents.some(
          (event) =>
            event.id === report.id && event.type === "report",
        ),
    )
    .map((report) => ({
      ...report,
      affected: report.statusReportsToPageComponents.map(
        (component) => component.pageComponent.name,
      ),
      updates: report.statusReportUpdates,
    }))}
/>
```

Your annotation tasks:

1. Mark visibility condition.
2. Mark event join.
3. Mark affected-service mapping.
4. Mark public component contract.

Model annotation:

```tsx
statusReports={page.statusReports
  .filter((report) =>
    report.statusReportUpdates.length > 0 && // Do not show empty reports in feed.
    page.lastEvents.some((event) =>
      event.id === report.id && event.type === "report" // Only recent/relevant computed events.
    ),
  )
  .map((report) => ({
    ...report,
    affected: report.statusReportsToPageComponents.map(
      (component) => component.pageComponent.name, // Convert join rows to display names.
    ),
    updates: report.statusReportUpdates, // Rename relation to StatusFeed prop shape.
  }))}
```

The router computes `lastEvents` before the client uses it (`packages/api/src/router/statusPage.ts:239-247`).

