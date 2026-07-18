# Dashboard State Stories

Stories 6-10 teach you to work with state that already exists in the dashboard: query results, URL search params, table filters, and form sheet state. The goal is to build features without inventing new backend behavior too early.

## Story 6: Add Subscriber Counts Above The Subscribers Table

**Difficulty:** Medium-Easy  
**Estimated Time:** 2 hours  
**Skills You'll Practice:** derived arrays, dashboard metrics, table-adjacent UI  
**The Story:** As a dashboard user, I want to see active and unsubscribed subscriber counts so that I can quickly understand the health of a status page's audience.

**Acceptance Criteria:**
- [ ] The subscribers page shows active subscriber count.
- [ ] The subscribers page shows unsubscribed subscriber count.
- [ ] Counts are derived from the existing `subscribers` query.
- [ ] Existing default table filter for active subscribers remains unchanged.

**Read These Anchors First:**
- `apps/dashboard/src/app/(dashboard)/status-pages/[id]/subscribers/page.tsx:80-87` - page, subscribers, workspace, and components queries.
- `apps/dashboard/src/app/(dashboard)/status-pages/[id]/subscribers/page.tsx:183-189` - subscribers table and default active filter.
- `packages/api/src/router/pageSubscriber.ts:233-249` - protected subscriber list route.

**Files You'll Likely Touch:**
- `apps/dashboard/src/app/(dashboard)/status-pages/[id]/subscribers/page.tsx` - derive counts and render small summary UI.

**Detailed Implementation Plan:**
1. After `items` is computed, derive counts from `subscribers ?? []`.
2. Treat a subscriber as active when `unsubscribedAt` is falsy. The sample data has `unsubscribedAt: null` at `apps/dashboard/src/app/(dashboard)/status-pages/[id]/subscribers/page.tsx:50-54`.
3. Render the counts near the `SectionHeader` or directly above the table section.
4. Do not remove `defaultColumnFilters={[{ id: "status", value: ["active"] }]}` at `apps/dashboard/src/app/(dashboard)/status-pages/[id]/subscribers/page.tsx:184-189`.
5. If subscribers are still loading, render nothing or a neutral placeholder.

**Fake-Code Shape:**

```tsx
const subscriberRows = subscribers ?? [];
const activeCount = subscriberRows.filter((s) => !s.unsubscribedAt).length;
const unsubscribedCount = subscriberRows.length - activeCount;

<div className="grid gap-2 sm:grid-cols-2">
  <SummaryItem label="Active" value={activeCount} />
  <SummaryItem label="Unsubscribed" value={unsubscribedCount} />
</div>
```

**Tips:**
- Keep the counts read-only. This story should not change `pageSubscriber.list`.
- The table toolbar and columns still own filtering display.
- Use a simple inline UI if there is no existing metric-card pattern in this page.

**What Could Go Wrong:**
- Counting the example data in the billing overlay would show fake counts. Render counts only for real `subscribers`.
- Treating `acceptedAt` as the active signal can be wrong if a subscriber later unsubscribes.

**Suggested Checks:**
- Page with zero subscribers shows zero or hides the summary cleanly.
- Page with unsubscribed subscribers reports both counts correctly.
- Table still defaults to active subscribers.

**Review Questions:**
- Did the UI use the same data as the table?
- Does the count mean what the label says?
- Would this still work if a webhook subscriber has no email?

## Story 7: Add A Clear Monitor Filters Button

**Difficulty:** Medium-Easy  
**Estimated Time:** 2.5 hours  
**Skills You'll Practice:** controlled table state, URL state, reset actions  
**The Story:** As a dashboard user, I want one clear action to remove monitor list filters and sorting so that I can quickly return to the full monitor list.

**Acceptance Criteria:**
- [ ] A clear action appears only when a status filter or p95 sort is active.
- [ ] Clicking it clears `columnFilters`, `sorting`, and URL search params.
- [ ] Existing metric-card click behavior still works.
- [ ] Table default column visibility is unchanged.

**Read These Anchors First:**
- `apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx:47-49` - search params, sorting, and column filter state.
- `apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx:80-90` - initial URL-to-state hydration.
- `apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx:124-146` - metric-card filter/sort toggles.
- `apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx:164-194` - controlled `DataTable` props.

**Files You'll Likely Touch:**
- `apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx` - add derived active state and reset button.

**Detailed Implementation Plan:**
1. Derive `const hasActiveControls = columnFilters.length > 0 || sorting.length > 0`.
2. Render a small button near the metric cards or table header.
3. On click, call `setColumnFilters([])`, `setSorting([])`, and `setSearchParams({ status: null, sort: null })`.
4. Keep the existing metric card toggles unchanged.
5. Check that clearing from a URL-loaded state works after page refresh.

**Fake-Code Shape:**

```tsx
const hasActiveControls = columnFilters.length > 0 || sorting.length > 0;

{hasActiveControls ? (
  <Button
    variant="ghost"
    size="sm"
    onClick={() => {
      setSearchParams({ status: null, sort: null });
      setColumnFilters([]);
      setSorting([]);
    }}
  >
    Clear filters
  </Button>
) : null}
```

**Tips:**
- The state lives in three places: `searchParams`, `columnFilters`, and `sorting`.
- Clearing only local state will leave the URL stale.
- Clearing only URL state will not update the controlled table until React state changes.

**What Could Go Wrong:**
- If `setSearchParams` uses the wrong key names, the URL keeps old state. Check `apps/dashboard/src/app/(dashboard)/monitors/(list)/search-params.ts`.
- If you render the button inside every metric card, the UI gets noisy.

**Suggested Checks:**
- Click a status metric, then clear.
- Click p95, then clear.
- Load the page with `?status=...` or `?sort=...`, then clear.

**Review Questions:**
- Did you clear all sources of truth?
- Is the button hidden when there is nothing to clear?
- Does it behave the same after refresh?

## Story 8: Add Notification Provider Search

**Difficulty:** Medium-Easy  
**Estimated Time:** 3 hours  
**Skills You'll Practice:** local state, filtering config maps, preserving form state  
**The Story:** As a dashboard user, I want to search notification providers by name so that I can find Slack, Discord, PagerDuty, or webhook quickly.

**Acceptance Criteria:**
- [ ] A search input filters provider cards by provider label.
- [ ] Existing notification table is not filtered by this search.
- [ ] Provider cards still respect plan limits and total channel limits.
- [ ] `defaultOpen` behavior from URL `channel` still works.

**Read These Anchors First:**
- `apps/dashboard/src/app/(dashboard)/notifications/client.tsx:25-35` - notification config and base URL.
- `apps/dashboard/src/app/(dashboard)/notifications/client.tsx:88-162` - provider card loop.
- `apps/dashboard/src/app/(dashboard)/notifications/client.tsx:131-160` - `FormSheetNotifier` props and submit payload.

**Files You'll Likely Touch:**
- `apps/dashboard/src/app/(dashboard)/notifications/client.tsx` - add search state and filter the config keys before mapping.

**Detailed Implementation Plan:**
1. Import `useState` if this file does not already import it.
2. Add `const [providerSearch, setProviderSearch] = useState("")`.
3. Build `const providerKeys = Object.keys(config).filter(...)`.
4. Filter against `config[key].label.toLowerCase()`.
5. Replace `Object.keys(config).map` with `providerKeys.map`.
6. Add a small empty state for "No providers match" if the filter returns no keys.
7. Do not filter existing `notifications`; this search only helps pick a provider to create.

**Fake-Code Shape:**

```tsx
const providerKeys = Object.keys(config).filter((provider) => {
  const key = provider as keyof typeof config;
  return config[key].label
    .toLowerCase()
    .includes(providerSearch.trim().toLowerCase());
});

<Input
  value={providerSearch}
  onChange={(event) => setProviderSearch(event.target.value)}
  placeholder="Search providers"
/>

{providerKeys.map((notifier) => {
  // existing card body
})}
```

**Tips:**
- Keep the `keyof typeof config` cast close to where it is used.
- `searchParams.channel` currently opens a provider sheet by default at `apps/dashboard/src/app/(dashboard)/notifications/client.tsx:136`.
- If the URL asks for a provider that does not match the search, decide whether to force it visible or leave search empty by default.

**What Could Go Wrong:**
- Filtering the table data instead of provider cards changes a different feature.
- Recomputing `Object.keys(config)` in multiple places can make empty-state logic disagree with rendering.

**Suggested Checks:**
- Search "slack"; Slack remains.
- Search a nonsense string; provider creation area shows a clean empty message.
- Open a `?channel=slack` URL; Slack sheet behavior still works.

**Review Questions:**
- Is the search local and reversible?
- Did provider plan limits still apply after filtering?
- Does search affect only creation, not existing notifications?

## Story 9: Show Component Count On The Status Page Components Screen

**Difficulty:** Medium-Easy  
**Estimated Time:** 3 hours  
**Skills You'll Practice:** adding read-only query, derived display, query loading safety  
**The Story:** As a dashboard user, I want to see how many components belong to a status page so that I know whether the public page has enough service coverage.

**Acceptance Criteria:**
- [ ] Components page shows total component count.
- [ ] Count is loaded from `pageComponent.list`.
- [ ] Count handles loading and zero states cleanly.
- [ ] Existing component update form remains unchanged.

**Read These Anchors First:**
- `apps/dashboard/src/app/(dashboard)/status-pages/[id]/components/page.tsx:15-33` - current page query and form render.
- `packages/api/src/router/pageComponent.ts:15-37` - component list input.
- `packages/api/src/router/page.ts:148-159` - status page identity query.

**Files You'll Likely Touch:**
- `apps/dashboard/src/app/(dashboard)/status-pages/[id]/components/page.tsx` - add `pageComponent.list` query and count display.

**Detailed Implementation Plan:**
1. In the components page, derive `pageId` once from `id`.
2. Keep the existing `page.get` query.
3. Add `const { data: components } = useQuery(trpc.pageComponent.list.queryOptions({ pageId }))`.
4. Compute `const componentCount = components?.length`.
5. Render copy like `3 components configured` only when the query has resolved.
6. Keep `FormComponentsUpdate` as the main editor.
7. If you need pluralization, handle `1 component` separately.

**Fake-Code Shape:**

```tsx
const pageId = Number.parseInt(id);
const { data: components } = useQuery(
  trpc.pageComponent.list.queryOptions({ pageId }),
);

const componentCountLabel =
  components === undefined
    ? "Loading components"
    : components.length === 1
      ? "1 component configured"
      : `${components.length} components configured`;
```

**Tips:**
- This story is intentionally read-only: no router or service changes.
- Use the same `pageId` for both queries so the count matches the page being edited.
- Avoid showing `0` during loading unless you explicitly know the query finished.

**What Could Go Wrong:**
- Calling `pageComponent.list` with `NaN` can cause confusing router errors.
- Adding the count inside `FormComponentsUpdate` could make a small task larger than needed.

**Suggested Checks:**
- Page with no components says zero.
- Page with one component uses singular copy.
- Page with multiple components uses plural copy.

**Review Questions:**
- Did you fetch the least data needed?
- Does the count correspond to the current page id?
- Did you leave editing behavior alone?

## Story 10: Add Maintenance Period Quick Filters To The Status Page Maintenance View

**Difficulty:** Medium  
**Estimated Time:** 4 hours  
**Skills You'll Practice:** existing router inputs, dashboard query state, period filters  
**The Story:** As a dashboard user, I want to filter maintenance windows by recent period so that I can quickly review upcoming or recently changed maintenance work.

**Acceptance Criteria:**
- [ ] Maintenance page exposes 1d, 7d, and 14d quick filters.
- [ ] Selected period is passed to `maintenance.list`.
- [ ] Page id filtering remains intact.
- [ ] Clearing the filter returns to the existing default list.

**Read These Anchors First:**
- `packages/api/src/router/maintenance.ts:38-66` - `maintenance.list` already accepts `period`, `pageId`, and `order`.
- `packages/api/src/router/maintenance.ts:68-98` - create route, useful for understanding maintenance fields.
- `apps/dashboard/src/app/(dashboard)/status-pages/[id]/maintenances/page.tsx` - current dashboard maintenance page.
- `apps/dashboard/src/components/forms/maintenance/form.tsx` - maintenance form fields if you need domain context.

**Files You'll Likely Touch:**
- `apps/dashboard/src/app/(dashboard)/status-pages/[id]/maintenances/page.tsx` - add local period state and pass it to query.

**Detailed Implementation Plan:**
1. Open the maintenance page and find the `trpc.maintenance.list.queryOptions` call.
2. Add state for `period`, with type `"1d" | "7d" | "14d" | undefined`.
3. Render a segmented control, select, or small button group using existing UI components.
4. Pass `{ pageId, period }` into `maintenance.list`.
5. Add a clear option that sets `period` to `undefined`.
6. Do not modify `packages/api/src/router/maintenance.ts`; it already accepts the input.
7. Verify the page still only lists maintenance windows for the selected status page.

**Fake-Code Shape:**

```tsx
type Period = "1d" | "7d" | "14d";
const [period, setPeriod] = useState<Period | undefined>();

const { data: maintenances } = useQuery(
  trpc.maintenance.list.queryOptions({
    pageId,
    period,
  }),
);
```

**Tips:**
- The router maps empty input to defaults, including `order: "desc"` and a large limit (`packages/api/src/router/maintenance.ts:50-60`).
- Keep the period labels honest: the backend period behavior may be based on creation or date filtering inside the service. Read the service before promising "upcoming."
- If you use URL search params, follow the monitors list pattern in `apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx:47-90`.

**What Could Go Wrong:**
- Accidentally dropping `pageId` will show maintenance windows from other pages.
- Passing `null` instead of `undefined` may not match the router schema.

**Suggested Checks:**
- Select each period and watch the query refetch.
- Clear the period and confirm the original list returns.
- Confirm create/edit maintenance still works.

**Review Questions:**
- Did you use the existing router contract instead of adding a new one?
- Is the UI label accurate to the backend semantics?
- Does page scoping remain intact?

