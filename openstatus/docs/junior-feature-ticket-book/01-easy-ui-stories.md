# Easy UI Stories

These first five stories are deliberately small. They train the habit of reading before editing, touching the fewest files possible, and checking that user-visible behavior changed without accidentally changing data flow.

## Story 1: Add A Helpful Components Page Empty State

**Difficulty:** Easy  
**Estimated Time:** 1 hour  
**Skills You'll Practice:** empty states, safe copy changes, reading query boundaries  
**The Story:** As a dashboard user, I want the status-page components screen to explain what page components do so that I understand why this page matters before I add monitors or static components.

**Acceptance Criteria:**
- [ ] When a status page has no components, the components page shows a useful empty-state message.
- [ ] The existing `FormComponentsUpdate` flow still renders for pages that can edit components.
- [ ] The page title still uses the loaded status page title.
- [ ] No API or service code changes are made.

**Read These Anchors First:**
- `apps/dashboard/src/app/(dashboard)/status-pages/[id]/components/page.tsx:15-33` - loads the page and renders `FormComponentsUpdate`.
- `packages/api/src/router/page.ts:148-159` - dashboard `page.get` route used by the page.
- `packages/api/src/router/pageComponent.ts:15-37` - component list query shape, useful if you decide to inspect component count.

**Files You'll Likely Touch:**
- `apps/dashboard/src/app/(dashboard)/status-pages/[id]/components/page.tsx` - add the helpful copy around the existing form.

**Detailed Implementation Plan:**
1. Open `components/page.tsx` and identify the existing render order: query page, return `null` while loading, render `SectionGroup`, render `SectionHeader`, render `FormComponentsUpdate`.
2. Do not change `trpc.page.get.queryOptions({ id: Number.parseInt(id) })`; that query is the page identity anchor.
3. Add a short description that explains components in domain language: components are the named services or groups shown on the public status page.
4. If you want to show the empty state only when there are no components, first confirm whether `statusPage` includes component data. If it does not, add a read-only `pageComponent.list` query using the existing router shape from `packages/api/src/router/pageComponent.ts:15-37`.
5. Keep `FormComponentsUpdate` in place. The empty state should guide, not replace, the editor.
6. Manually check a status page with components and one without components.

**Fake-Code Shape:**

```tsx
// Shape only. Match the repo's actual imports and component names.
const { data: components } = useQuery(
  trpc.pageComponent.list.queryOptions({ pageId: Number.parseInt(id) }),
);

const hasNoComponents = components?.length === 0;

return (
  <SectionGroup>
    <Section>
      <SectionHeader>
        <SectionTitle>{statusPage.title}</SectionTitle>
        <SectionDescription>
          Components are the public-facing services your users track.
        </SectionDescription>
      </SectionHeader>
      {hasNoComponents ? <EmptyStateContainer>{/* guidance */}</EmptyStateContainer> : null}
      <FormComponentsUpdate />
    </Section>
  </SectionGroup>
);
```

**Tips:**
- The components page is a client component, so React Query hooks are allowed (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/components/page.tsx:1-20`).
- Use existing content components if imported nearby, such as `SectionDescription` in `apps/dashboard/src/app/(dashboard)/status-pages/[id]/components/page.tsx:3-9`.
- If you add a second query, name the derived boolean clearly, such as `hasNoComponents`.

**What Could Go Wrong:**
- Showing the empty state before the query has loaded can make the page flash "empty" during loading.
- Replacing `FormComponentsUpdate` would remove the actual editing surface.
- Using `Number(id)` without validating can make `NaN` travel into a query; this page currently uses `Number.parseInt(id)` at `apps/dashboard/src/app/(dashboard)/status-pages/[id]/components/page.tsx:18-19`.

**Suggested Checks:**
- Open a status page components screen.
- Verify no console error appears.
- Verify the form still appears.

**Review Questions:**
- Did you change only display behavior?
- Does the empty state appear because data is truly empty, not because data is still loading?
- Could a user understand why components matter after reading the copy?

## Story 2: Make The Subscriber Empty State Actionable

**Difficulty:** Easy  
**Estimated Time:** 1 hour  
**Skills You'll Practice:** dashboard empty states, reading conditional rendering, preserving mutation triggers  
**The Story:** As a dashboard user, I want the subscribers empty state to tell me how to add the first subscriber so that I know what to do next.

**Acceptance Criteria:**
- [ ] The empty state still appears only when the workspace is not limited and `subscribers` is empty.
- [ ] The empty state tells users to use the "Add subscriber" action.
- [ ] The `FormSheetSubscriber` trigger still works.
- [ ] The billing overlay for limited workspaces is unchanged.

**Read These Anchors First:**
- `apps/dashboard/src/app/(dashboard)/status-pages/[id]/subscribers/page.tsx:73-101` - page id parsing, queries, mutation, and component tree data.
- `apps/dashboard/src/app/(dashboard)/status-pages/[id]/subscribers/page.tsx:111-149` - add-subscriber button or billing-locked button.
- `apps/dashboard/src/app/(dashboard)/status-pages/[id]/subscribers/page.tsx:183-197` - table versus empty state.
- `packages/api/src/router/pageSubscriber.ts:230-357` - protected subscriber list/create/update/delete routes.

**Files You'll Likely Touch:**
- `apps/dashboard/src/app/(dashboard)/status-pages/[id]/subscribers/page.tsx` - update empty state text and optionally add a small action.

**Detailed Implementation Plan:**
1. Find the final render branch: limited overlay, subscribers table, or empty state.
2. Update `EmptyStateTitle` and `EmptyStateDescription` in the empty branch.
3. If adding a button inside the empty state, reuse `setOpenAdd(true)` so it opens the existing `FormSheetSubscriber` instead of creating a new form.
4. Do not duplicate the `createAction` mutation. The `FormSheetSubscriber` trigger already owns submission at `apps/dashboard/src/app/(dashboard)/status-pages/[id]/subscribers/page.tsx:120-148`.
5. Check that limited workspaces still see `BillingOverlayContainer`.

**Fake-Code Shape:**

```tsx
// Shape only. Prefer the repo's existing Button import if already present.
<EmptyStateContainer>
  <EmptyStateTitle>No subscribers yet</EmptyStateTitle>
  <EmptyStateDescription>
    Add an email or webhook subscriber to notify users when this page changes.
  </EmptyStateDescription>
  <Button variant="outline" size="sm" onClick={() => setOpenAdd(true)}>
    Add subscriber
  </Button>
</EmptyStateContainer>
```

**Tips:**
- The `openAdd` state already exists at `apps/dashboard/src/app/(dashboard)/status-pages/[id]/subscribers/page.tsx:77-78`.
- The existing mutation refetches subscribers on success at `apps/dashboard/src/app/(dashboard)/status-pages/[id]/subscribers/page.tsx:89-93`.
- Keep the domain distinction clear: subscribers can be `email` or `webhook`, as the create mutation branch shows at `apps/dashboard/src/app/(dashboard)/status-pages/[id]/subscribers/page.tsx:124-142`.

**What Could Go Wrong:**
- Adding a second form trigger can desync `openAdd`.
- Accidentally rendering the CTA inside the billing-limited branch can suggest the user can add a subscriber when they cannot.

**Suggested Checks:**
- Empty subscriber page opens the existing add sheet.
- Add sheet can still submit.
- Limited workspace branch still shows upgrade overlay.

**Review Questions:**
- Did you reuse existing state?
- Did you keep the locked and unlocked flows separate?
- Would the empty state make sense to a non-engineer product owner?

## Story 3: Add Provider Reason Text To Disabled Notification Cards

**Difficulty:** Easy  
**Estimated Time:** 1.5 hours  
**Skills You'll Practice:** conditional copy, existing config maps, limits UX  
**The Story:** As a dashboard user, I want disabled notification provider cards to explain why they are disabled so that I know whether I need to upgrade or remove another notifier.

**Acceptance Criteria:**
- [ ] Disabled notification cards show a short reason.
- [ ] The reason distinguishes provider-specific plan limits from total notification-channel limit.
- [ ] Enabled cards keep their current description.
- [ ] PagerDuty's external install link remains intact.

**Read These Anchors First:**
- `apps/dashboard/src/app/(dashboard)/notifications/client.tsx:37-55` - notifications, workspace, monitors, create mutation, limit calculation.
- `apps/dashboard/src/app/(dashboard)/notifications/client.tsx:88-162` - provider card rendering and enabled calculation.
- `packages/api/src/router/notification.ts:53-123` - list/create/update/delete router shape.

**Files You'll Likely Touch:**
- `apps/dashboard/src/app/(dashboard)/notifications/client.tsx` - calculate disabled reason and render it.

**Detailed Implementation Plan:**
1. Find the `Object.keys(config).map` loop.
2. Add a `let disabledReason: string | null = null` near `let enabled = true`.
3. When a provider-specific workspace limit disables a card, set `disabledReason = "Upgrade to use this channel."`.
4. When `limitReached` disables a card, set `disabledReason = "Notification channel limit reached."`.
5. Render `disabledReason ?? "Send notifications to ..."` inside `ActionCardDescription`.
6. Apply the same copy path to the PagerDuty branch, because it returns early at `apps/dashboard/src/app/(dashboard)/notifications/client.tsx:105-129`.
7. Do not change the shape passed to `FormSheetNotifier` at `apps/dashboard/src/app/(dashboard)/notifications/client.tsx:131-160`.

**Fake-Code Shape:**

```tsx
let enabled = true;
let disabledReason: string | null = null;

if (key in workspace.limits && !workspace.limits[key as LimitKey]) {
  enabled = false;
  disabledReason = "Upgrade to use this notification channel.";
}

if (limitReached) {
  enabled = false;
  disabledReason = "Notification channel limit reached.";
}

<ActionCardDescription>
  {disabledReason ?? `Send notifications to ${config[key].label}`}
</ActionCardDescription>
```

**Tips:**
- Keep the `key as "opsgenie" | "sms" | "opsgenie" | "whatsapp"` cast contained if you touch it; it is currently used in `apps/dashboard/src/app/(dashboard)/notifications/client.tsx:94-99`.
- `limitReached` is based on total notifications at `apps/dashboard/src/app/(dashboard)/notifications/client.tsx:53-54`.
- PagerDuty has a special URL branch at `apps/dashboard/src/app/(dashboard)/notifications/client.tsx:105-127`.

**What Could Go Wrong:**
- If you compute reason after `limitReached`, provider-specific reasons may be overwritten. Decide which reason should win.
- If you remove `data-disabled`, disabled links may become clickable.

**Suggested Checks:**
- Simulate a limit-reached workspace if possible.
- Confirm enabled providers still open the form sheet.
- Confirm PagerDuty still navigates when enabled.

**Review Questions:**
- Did you avoid duplicating card markup too much?
- Does the reason correspond to the actual disabled condition?
- Did you keep creation behavior unchanged?

## Story 4: Add Monitor List Metric Helper Text

**Difficulty:** Easy  
**Estimated Time:** 1.5 hours  
**Skills You'll Practice:** derived metric reading, dashboard UX, no-data states  
**The Story:** As a dashboard user, I want the monitor metric cards to make it clearer that clicking them filters or sorts the monitor table so that I understand why the table changes.

**Acceptance Criteria:**
- [ ] Metric cards communicate that they are interactive.
- [ ] Existing click behavior for status filters and p95 sorting is unchanged.
- [ ] No changes are made to Tinybird query inputs.
- [ ] No table column behavior changes.

**Read These Anchors First:**
- `apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx:43-78` - monitor, tag, metric queries.
- `apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx:80-90` - URL search params hydrate filters/sorting.
- `apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx:108-162` - metric card rendering and click behavior.
- `apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx:164-194` - table receives filters, sorting, and default visibility.

**Files You'll Likely Touch:**
- `apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx` - add helper text or accessible title to metric cards.

**Detailed Implementation Plan:**
1. Read the `metrics.map` block from `apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx:108-162`.
2. Identify the two click modes: `p95` toggles sort, other metric keys toggle the `status` filter.
3. Add helper text inside each `MetricCardButton`, or add an accessible `title`/`aria-label` if the metric component is too compact.
4. If visible helper text is too noisy, add a short `title` such as `Click to filter monitors by ${metric.title}`.
5. Do not alter `setSearchParams`, `setColumnFilters`, or `setSorting`.
6. Check that p95 still sorts descending and status cards still filter.

**Fake-Code Shape:**

```tsx
const actionLabel =
  metric.key === "p95"
    ? "Click to sort by p95 latency"
    : `Click to filter by ${metric.title}`;

<MetricCardButton aria-label={actionLabel} title={actionLabel}>
  {/* existing header and value */}
</MetricCardButton>
```

**Tips:**
- The `metric.key === "p95"` branch is the only sorting branch (`apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx:129-136`).
- Status filter state is written with `setColumnFilters([{ id: "status", value: [metric.key] }])` at `apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx:138-140`.
- The table is already controlled by `columnFilters` and `sorting` at `apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx:185-188`.

**What Could Go Wrong:**
- Changing `metric.key` handling can break filters.
- Visible helper text may crowd the metric card if not styled carefully.

**Suggested Checks:**
- Click p95 twice: first click sorts, second clears.
- Click a status metric twice: first click filters, second clears.
- Keyboard focus still reaches the metric button.

**Review Questions:**
- Did you preserve behavior and only clarify it?
- Is the helper text true for both sorting and filtering?
- Would screen-reader users get the same interaction clue?

## Story 5: Add Feed Count Copy To The Public Status Page

**Difficulty:** Easy  
**Estimated Time:** 2 hours  
**Skills You'll Practice:** public page rendering, derived values, mapping existing data  
**The Story:** As a public status-page visitor, I want to know how many recent feed items are being shown so that I understand whether the page has recent activity.

**Acceptance Criteria:**
- [ ] Public status page shows a short count near the feed section.
- [ ] Count is derived from the same filtered reports and maintenances passed to `StatusFeed`.
- [ ] Embed mode still hides the feed when requested.
- [ ] No API changes are made.

**Read These Anchors First:**
- `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:56-136` - public page query and derived page selection.
- `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:329-360` - filtered reports and maintenances passed into `StatusFeed`.
- `packages/api/src/router/statusPage.ts:64-113` - public status-page `get` route begins the data contract.

**Files You'll Likely Touch:**
- `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx` - derive feed arrays before JSX and display count.

**Detailed Implementation Plan:**
1. In `Client`, find the `StatusFeed` props at `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:329-360`.
2. Extract the report filtering/mapping into a `const feedReports = ...` before `return`.
3. Extract the maintenance filtering/mapping into a `const feedMaintenances = ...` before `return`.
4. Compute `const feedItemCount = feedReports.length + feedMaintenances.length`.
5. Render a small count above `StatusFeed` inside the existing feed `StatusContent`.
6. Pass `feedReports` and `feedMaintenances` into `StatusFeed`.
7. Verify the CSS class `group-data-[hide-feed=true]/embed:hidden` still hides the whole feed section.

**Fake-Code Shape:**

```tsx
const feedReports = page.statusReports
  .filter((report) => report.statusReportUpdates.length > 0)
  .filter((report) =>
    page.lastEvents.some((event) => event.id === report.id && event.type === "report"),
  )
  .map((report) => ({
    ...report,
    affected: report.statusReportsToPageComponents.map((c) => c.pageComponent.name),
    updates: report.statusReportUpdates,
  }));

const feedItemCount = feedReports.length + feedMaintenances.length;
```

**Tips:**
- Do not change the filter semantics while extracting; move first, then render count.
- Reports with no updates are intentionally filtered out at `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:331-339`.
- The feed receives `affected` names, not raw relation rows, at `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:340-346`.

**What Could Go Wrong:**
- A careless extraction can compute feed arrays before `page` exists. Do it after `if (!pageInitial) return null` and after `const page = ...`.
- Counting unfiltered reports would disagree with visible feed items.

**Suggested Checks:**
- Public page with no last events shows count `0` or a polished empty phrase.
- Public page with reports and maintenances shows the exact visible total.
- Embed mode hiding still works.

**Review Questions:**
- Does the count match what `StatusFeed` receives?
- Did you avoid changing API data shape?
- Is the copy useful without feeling noisy?

