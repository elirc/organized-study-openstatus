# Tier 2: Mid-Level Missions

### Mission 9: State Has a Home and a Reason
**Tier:** Mid-Level
**Time Estimate:** 40 minutes
**Goal:** Separate local UI state from remote server state.
**The Concept:** A status report has a source of truth; state should too.
**Design Intent Before You Read the Code:** The sheet owns open/closed state; React Query owns server data; the page coordinates mutation side effects.
**Find It In The Code:** Open `apps/dashboard/src/components/forms/status-report/sheet.tsx:34-72`, `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:27-51`.

```tsx
const [open, setOpen] = useState(false); // local UI state belongs inside the sheet.
```

```tsx
const { data: statusReports, refetch } = useQuery(
  trpc.statusReport.list.queryOptions({ pageId: Number.parseInt(id) }),
); // remote state belongs to React Query cache.

queryClient.invalidateQueries({ queryKey: trpc.page.list.queryKey() });
// after mutation, invalidate related cached server state.
```

**The Aha Moment:** Put state where the owner can update it without surprising other owners.
**Socratic Checkpoint:** 5 questions.
1. Which state is local?
2. Which state is remote?
3. Which code refetches reports?
4. Which code invalidates page summaries?
5. What bug appears if the sheet owns the mutation?

How to self-grade: Strong answers cite `apps/dashboard/src/components/forms/status-report/sheet.tsx:34-72` and `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:27-51`.
**Connects To:** Mission 11 and Mission 14.

### Mission 10: The Custom Hook Ecosystem
**Tier:** Mid-Level
**Time Estimate:** 35 minutes
**Goal:** Read custom hooks as reusable environment sensors.
**The Concept:** Hooks are like monitors inside the UI: they observe conditions and report usable state.
**Design Intent Before You Read the Code:** Hooks should isolate browser APIs and expose simple values to components.
**Find It In The Code:** Open `packages/ui/src/hooks/use-mobile.ts:7-23`, `apps/dashboard/src/components/forms/status-report/form.tsx:51-57`, `apps/dashboard/src/components/forms/status-report/form.tsx:231-235`.

```ts
export function useIsMobile() {
  const [isMobile, setIsMobile] = React.useState<boolean | undefined>(undefined);

  React.useEffect(() => {
    const mql = window.matchMedia(`(max-width: ${MOBILE_BREAKPOINT - 1}px)`);
    const onChange = () => {
      setIsMobile(window.innerWidth < MOBILE_BREAKPOINT); // browser measurement isolated here.
    };
    mql.addEventListener("change", onChange);
    return () => mql.removeEventListener("change", onChange); // cleanup matters.
  }, []);

  return !!isMobile;
}
```

**The Aha Moment:** A custom hook turns browser complexity into a stable component input.
**Socratic Checkpoint:** 5 questions.
1. What browser API does this hook wrap?
2. Why does it initialize with `undefined`?
3. Where is it consumed?
4. What cleanup prevents leaks?
5. What would this hook look like if there were no custom hook layer?

How to self-grade: Strong answers cite `packages/ui/src/hooks/use-mobile.ts:7-23` and `apps/dashboard/src/components/forms/status-report/form.tsx:231-235`.
**Connects To:** Mission 16 and Mission 21.

### Mission 11: Side Effects Are Promises to the System
**Tier:** Mid-Level
**Time Estimate:** 45 minutes
**Goal:** Identify side effects and decide who should own them.
**The Concept:** A side effect is like notifying subscribers during an incident: once promised, it must be reliable.
**Design Intent Before You Read the Code:** Creating a report and notifying subscribers are different effects with different failure modes.
**Find It In The Code:** Open `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`, `packages/services/src/status-report/create.ts:82-95`.

```tsx
if (statusReport.notifySubscribers) {
  await sendStatusReportUpdateMutation.mutateAsync({ id: statusReport.id });
}
// UI performs notification after create success.
// If this fails, DB creation already happened.
```

```ts
await emitAudit(tx, ctx, {
  action: "status_report.create",
  entityType: "status_report",
  entityId: newReport.id,
  after: newReport,
});
// Audit is service-owned and part of the transaction boundary.
```

**The Aha Moment:** Side effects need an owner, retry story, and failure policy.
**Socratic Checkpoint:** 5 questions.
1. Which side effect is client-owned today?
2. Which side effect is service-owned?
3. What happens if email send fails after create?
4. Why might notification move server-side?
5. What test would prove the behavior?

How to self-grade: Strong answers cite `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51` and `packages/services/src/status-report/create.ts:82-95`.
**Connects To:** Mission 15 and Mission 19.

### Mission 12: The Full API Contract
**Tier:** Mid-Level
**Time Estimate:** 50 minutes
**Goal:** Map tRPC procedure contracts for status reports.
**The Concept:** API contracts are like status-page public promises: consumers build around them.
**Design Intent Before You Read the Code:** Router schemas preserve wire compatibility while services enforce canonical behavior.
**Find It In The Code:** Open `packages/api/src/router/statusReport.ts:24-222`.

```ts
const createStatusReportTRPCInput = z.object({
  pageComponents: z.array(z.number()), // dashboard-facing name
});

const { initialUpdate } = await createStatusReport({
  input: {
    pageComponentIds: input.pageComponents, // service-facing name
  },
});

return { ...initialUpdate, notifySubscribers: input.notifySubscribers };
// Return shape is intentionally legacy-friendly for dashboard consumers.
```

**The Aha Moment:** A router can be both backward-compatible and internally modern.
**Socratic Checkpoint:** 5 questions.
1. Which create input field is renamed for the service?
2. Which procedures are mutations?
3. Which procedure filters orphan reports?
4. What does `narrowPage` protect?
5. How are service errors translated?

How to self-grade: Strong answers cite `packages/api/src/router/statusReport.ts:24-222` and `packages/api/src/service-adapter.ts:32-87`.
**Connects To:** Mission 13 and Mission 14.

### Mission 13: The Middleware Chain
**Tier:** Mid-Level
**Time Estimate:** 45 minutes
**Goal:** Trace authentication and workspace resolution.
**The Concept:** Middleware is the checkpoint before the control room: no workspace, no dashboard operation.
**Design Intent Before You Read the Code:** API routes should not rely on UI state for workspace safety.
**Find It In The Code:** Open `packages/api/src/trpc.ts:148-264`, `apps/dashboard/src/proxy.ts:45-65`.

```ts
if (!ctx.session?.user?.id) {
  throw new TRPCError({ code: "UNAUTHORIZED" });
}
// protectedProcedure starts with session identity.

const workspaceSlug = ctx.req?.cookies.get("workspace-slug")?.value;
const resolved = await resolveActiveWorkspace({
  userId: Number(ctx.session.user.id),
  workspaceSlug,
});
// Workspace is resolved server-side from user + cookie.
```

**The Aha Moment:** The backend reconstructs identity and workspace; it does not trust the page id alone.
**Socratic Checkpoint:** 5 questions.
1. What happens without session user id?
2. Where does workspace slug come from?
3. Where is the cookie set if missing?
4. Why is test mode special?
5. What does analytics wait for?

How to self-grade: Strong answers cite `packages/api/src/trpc.ts:148-264` and `apps/dashboard/src/proxy.ts:45-65`.
**Connects To:** Mission 14 and Mission 22.

### Mission 14: End-to-End Feature Trace
**Tier:** Mid-Level
**Time Estimate:** 60 minutes
**Goal:** Trace status report creation from UI to public display.
**The Concept:** End-to-end tracing is incident response for code: follow the signal until it reaches users.
**Design Intent Before You Read the Code:** Each layer should own one kind of work: UI intent, API contract, service rules, persistence, public transformation.
**Find It In The Code:** Open `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:89-103`, `packages/api/src/router/statusReport.ts:58-81`, `packages/services/src/status-report/create.ts:32-98`, `packages/db/src/schema/status_reports/status_reports.ts:15-63`, `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:329-360`.

```tsx
await createStatusReportMutation.mutateAsync({
  title: values.title,
  status: values.status,
  pageId: Number.parseInt(id),
  pageComponents: values.pageComponents,
  date: values.date,
  message: values.message,
  notifySubscribers: values.notifySubscribers,
});
// UI intent becomes a typed mutation payload.
```

```ts
const newReport = await tx.insert(statusReport).values({ workspaceId, pageId, title, status });
const initialUpdate = await tx.insert(statusReportUpdate).values({ statusReportId, status, date, message });
// Service turns one UI action into two persisted domain rows.
```

**The Aha Moment:** A real feature is a chain of contracts, not one function.
**Socratic Checkpoint:** 5 questions.
1. What fields leave the form?
2. What field is renamed by the router?
3. What rows are inserted?
4. What join data is written?
5. How does the public feed find the report?

How to self-grade: Strong answers cite all five line ranges in the mission and explain each boundary.
**Connects To:** Mission 18 and Mission 20.

### Mission 15: Read the Diff Like an Engineer
**Tier:** Mid-Level
**Time Estimate:** 45 minutes
**Goal:** Practice reviewing a realistic change.
**The Concept:** A diff is a status update for the codebase: it tells you what changed, but you must infer risk.
**Design Intent Before You Read the Code:** Moving notification from UI to server changes reliability, errors, and tests.
**Find It In The Code:** Open `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`, `packages/services/src/status-report/create.ts:24-99`, `packages/services/src/status-report/__tests__/status-report.test.ts:41-56`.

```tsx
// Existing behavior:
if (statusReport.notifySubscribers) {
  await sendStatusReportUpdateMutation.mutateAsync({ id: statusReport.id });
}
// Review question: if this moves server-side, who handles partial failure?
```

**The Aha Moment:** Review the behavior change, not just the code movement.
**Socratic Checkpoint:** 5 questions.
1. What behavior does the user see today?
2. What failure becomes possible after moving it?
3. Which tests need updating?
4. Which layer should own retry?
5. How would you preserve backward compatibility?

How to self-grade: Strong answers cite current UI behavior, service transaction, and test spy setup at the mission line ranges.
**Connects To:** Mission 19 and Mission 23.

### Mission 16: Composition Over Inheritance
**Tier:** Mid-Level
**Time Estimate:** 40 minutes
**Goal:** See how the UI builds complex screens from smaller pieces.
**The Concept:** A status page is composed of banner, components, feed, and updates; the code mirrors that.
**Design Intent Before You Read the Code:** Components should nest by responsibility, not inheritance.
**Find It In The Code:** Open `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:60-128`, `apps/dashboard/src/components/forms/status-report/sheet.tsx:36-72`, `packages/ui/src/components/blocks/status-blank.tsx:451-560`.

```tsx
<FormSheetStatusReport onSubmit={async (values) => { /* page-level mutation */ }}>
  <Button data-section="action" size="sm">
    <Plus />
    Create Status Report
  </Button>
</FormSheetStatusReport>
// Parent supplies trigger and save behavior; child supplies sheet structure.
```

**The Aha Moment:** Composition lets each component own one layer of behavior.
**Socratic Checkpoint:** 5 questions.
1. Which component owns the trigger button?
2. Which owns dirty-protection UI?
3. Which owns mutation?
4. Which shared component owns empty-state layout?
5. What duplication appears without composition?

How to self-grade: Strong answers cite `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:75-109`, `apps/dashboard/src/components/forms/status-report/sheet.tsx:36-72`, and `packages/ui/src/components/blocks/status-blank.tsx:451-560`.
**Connects To:** Mission 21 and Mission 23.

### Mission 17: TypeScript's Hidden Work
**Tier:** Mid-Level
**Time Estimate:** 45 minutes
**Goal:** Notice inferred types that quietly protect changes.
**The Concept:** Good types are like automated status checks: they catch mismatches before users do.
**Design Intent Before You Read the Code:** Router, service, DB, and UI types should flow from source definitions.
**Find It In The Code:** Open `packages/api/src/root.ts:1-6`, `apps/dashboard/src/lib/trpc/client.tsx:9-12`, `packages/db/src/schema/status_reports/validation.ts:49-55`, `packages/services/src/status-report/list.ts:49-64`.

```ts
export const appRouter = mergeRouters(edgeRouter, lambdaRouter);
export type AppRouter = typeof appRouter;
// The client imports this type, so procedure names and inputs stay synchronized.
```

```ts
export type StatusReportWithRelations = StatusReport & {
  updates: StatusReportUpdate[];
  pageComponents: PageComponent[];
  pageComponentIds: number[];
  page: (Page & { pageComponents: PageComponent[] }) | null;
};
// Service-level type describes enriched data, not just DB rows.
```

**The Aha Moment:** Types can encode both raw rows and richer domain objects.
**Socratic Checkpoint:** 5 questions.
1. Where is `AppRouter` inferred?
2. Where is it consumed?
3. Where are DB row types inferred?
4. Why is `StatusReportWithRelations` not just a DB row?
5. What compiler error would protect a renamed procedure?

How to self-grade: Strong answers cite all mission ranges and explain inferred vs explicit types.
**Connects To:** Mission 18 and Mission 23.

