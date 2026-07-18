# API And Service Stories

Stories 11-15 make small backend changes. The main lesson is that dashboard routes should validate input at the tRPC boundary, translate context with `toServiceCtx(ctx)`, and delegate domain work to service functions when a service already owns that behavior.

## Story 11: Add Sort Order To The Subscribers Page

**Difficulty:** Medium  
**Estimated Time:** 4 hours  
**Skills You'll Practice:** router input reuse, dashboard query state, sorting semantics  
**The Story:** As a dashboard user, I want to sort subscribers newest-first or oldest-first so that I can audit recent subscriber changes.

**Acceptance Criteria:**
- [ ] Subscribers page exposes ascending and descending order options.
- [ ] Selected order is passed to `pageSubscriber.list`.
- [ ] Existing `pageId` filtering remains required.
- [ ] The active-subscriber table filter remains unchanged.

**Read These Anchors First:**
- `apps/dashboard/src/app/(dashboard)/status-pages/[id]/subscribers/page.tsx:80-87` - current subscribers query.
- `apps/dashboard/src/app/(dashboard)/status-pages/[id]/subscribers/page.tsx:183-189` - data table branch and active default filter.
- `packages/api/src/router/pageSubscriber.ts:233-249` - `list` accepts `order`.

**Files You'll Likely Touch:**
- `apps/dashboard/src/app/(dashboard)/status-pages/[id]/subscribers/page.tsx` - add order state and pass it to query.

**Detailed Implementation Plan:**
1. Add `const [order, setOrder] = useState<"asc" | "desc">("desc")`.
2. Change the subscribers query from `{ pageId }` to `{ pageId, order }`.
3. Add a select or two-button control near the table header.
4. Keep the default value `desc`, matching router default expectations.
5. Do not edit `pageSubscriber.list`; the router already accepts `order` at `packages/api/src/router/pageSubscriber.ts:235-238`.
6. Check that `defaultColumnFilters` still filters active subscribers after order changes.

**Fake-Code Shape:**

```tsx
const [order, setOrder] = useState<"asc" | "desc">("desc");

const { data: subscribers, refetch } = useQuery(
  trpc.pageSubscriber.list.queryOptions({ pageId, order }),
);

<Select value={order} onValueChange={(value) => setOrder(value as "asc" | "desc")}>
  {/* existing select primitives from the UI package */}
</Select>
```

**Tips:**
- This is a UI use of an existing API; resist changing the service unless the service ignores order after inspection.
- The query key changes automatically because the input object changes.
- Make the UI label user-facing: "Newest first" and "Oldest first" are clearer than `desc` and `asc`.

**What Could Go Wrong:**
- Forgetting `pageId` leaks the query outside its intended page.
- Calling `refetch` manually on every select change is usually unnecessary; React Query should refetch when the query input changes.

**Suggested Checks:**
- Toggle order and confirm table order changes.
- Confirm adding a subscriber still refetches the current order.
- Confirm active filter remains applied.

**Review Questions:**
- Did you use the existing router input?
- Is the UI language clearer than the enum?
- Are query input and local state kept in sync?

## Story 12: Add Monitor List Sort Order

**Difficulty:** Medium  
**Estimated Time:** 4 hours  
**Skills You'll Practice:** query inputs, URL state option, table sort interaction  
**The Story:** As a dashboard user, I want to load monitors newest-first or oldest-first so that workspace audits are easier.

**Acceptance Criteria:**
- [ ] Monitor list exposes ascending and descending creation order.
- [ ] Selected order is passed to `monitor.list`.
- [ ] Existing metric-card filters and p95 sort still work.
- [ ] Existing tag toolbar still works.

**Read These Anchors First:**
- `apps/dashboard/src/app/(dashboard)/monitors/(list)/page.tsx:11-16` - server prefetch for monitor list and tags.
- `apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx:43-49` - monitor query and local table state.
- `packages/api/src/router/monitor.ts:131-148` - `monitor.list` accepts optional `order`.
- `apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx:164-194` - table controlled state.

**Files You'll Likely Touch:**
- `apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx` - add order state and query input.
- `apps/dashboard/src/app/(dashboard)/monitors/(list)/page.tsx` - optionally update prefetch if you choose a non-default order from URL.
- `apps/dashboard/src/app/(dashboard)/monitors/(list)/search-params.ts` - only if encoding order in the URL.

**Detailed Implementation Plan:**
1. Decide whether sort order is local state or URL state. For junior practice, start with local state.
2. Add `const [listOrder, setListOrder] = useState<"asc" | "desc">("desc")`.
3. Change `trpc.monitor.list.queryOptions()` to `trpc.monitor.list.queryOptions({ order: listOrder })`.
4. Render a compact select near the monitor section header.
5. Keep table sorting state separate. The API order controls loaded data order; table sort controls client sorting such as p95.
6. If you later move to URL state, update the server prefetch so hydration uses the same query key.

**Fake-Code Shape:**

```tsx
const [listOrder, setListOrder] = useState<"asc" | "desc">("desc");
const { data: monitors } = useQuery(
  trpc.monitor.list.queryOptions({ order: listOrder }),
);

// Keep this separate:
const [sorting, setSorting] = useState<SortingState>([]);
```

**Tips:**
- The router applies `order: input?.order ?? "desc"` at `packages/api/src/router/monitor.ts:137-142`.
- Monitor metric cards use table sorting/filtering after data is loaded (`apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx:124-146`).
- If prefetch uses no input but the client uses `{ order: "desc" }`, React Query may treat them as different keys. Consider updating prefetch to `{ order: "desc" }`.

**What Could Go Wrong:**
- Confusing API order with table sorting can make the UI feel contradictory.
- Hydration mismatch can cause an avoidable client refetch.

**Suggested Checks:**
- Switch order and confirm data reloads.
- Click p95 after changing order; p95 sort still toggles.
- Tag toolbar still filters.

**Review Questions:**
- Did you keep server ordering and table sorting conceptually separate?
- Did the query key match the prefetch key if needed?
- Would a teammate understand the state names?

## Story 13: Add A Safe "Send Test Webhook" Action For Dashboard Subscribers

**Difficulty:** Medium-Hard  
**Estimated Time:** 5 hours  
**Skills You'll Practice:** protected mutation usage, row actions, webhook safety  
**The Story:** As a dashboard user, I want to send a test payload to a webhook subscriber so that I can confirm the integration works before an incident.

**Acceptance Criteria:**
- [ ] Webhook subscriber rows expose a "Send test" action.
- [ ] Email subscriber rows do not show the action.
- [ ] Action calls `pageSubscriber.sendTestWebhook`.
- [ ] Success and failure feedback are visible to the user.
- [ ] No public subscriber route is used.

**Read These Anchors First:**
- `packages/api/src/router/pageSubscriber.ts:319-339` - protected `sendTestWebhook` mutation.
- `apps/dashboard/src/app/(dashboard)/status-pages/[id]/subscribers/page.tsx:183-189` - subscribers table setup.
- `apps/dashboard/src/components/data-table/subscribers/data-table-row-actions.tsx` - current row action location.
- `apps/dashboard/src/components/data-table/subscribers/columns.tsx` - row type and action column.

**Files You'll Likely Touch:**
- `apps/dashboard/src/components/data-table/subscribers/data-table-row-actions.tsx` - add conditional action.
- `apps/dashboard/src/components/data-table/subscribers/columns.tsx` - verify row action receives full row data.
- Possibly `apps/dashboard/src/app/(dashboard)/status-pages/[id]/subscribers/page.tsx` - pass page id if not already available in row data.

**Detailed Implementation Plan:**
1. Open the subscriber row actions file and identify how delete/edit actions are implemented.
2. Confirm the row data includes `id`, `pageId`, and `channelType`. The sample subscriber shape includes `channelType` and `pageId` at `apps/dashboard/src/app/(dashboard)/status-pages/[id]/subscribers/page.tsx:43-47`.
3. Add `const sendTestMutation = useMutation(trpc.pageSubscriber.sendTestWebhook.mutationOptions(...))`.
4. Render "Send test webhook" only when `row.original.channelType === "webhook"`.
5. Call `mutateAsync({ subscriberId: row.original.id, pageId: row.original.pageId })`.
6. Add toast feedback if row actions already use toast patterns. If not, add minimal button disabled/loading state.
7. Do not call `pageSubscriber.upsert`; that is public self-signup (`packages/api/src/router/pageSubscriber.ts:52-98`).

**Fake-Code Shape:**

```tsx
const sendTest = useMutation(
  trpc.pageSubscriber.sendTestWebhook.mutationOptions({
    onSuccess: () => toast.success("Test webhook sent"),
    onError: (error) => toast.error(error.message),
  }),
);

if (subscriber.channelType === "webhook") {
  return (
    <DropdownMenuItem
      disabled={sendTest.isPending}
      onClick={() =>
        sendTest.mutate({
          subscriberId: subscriber.id,
          pageId: subscriber.pageId,
        })
      }
    >
      Send test webhook
    </DropdownMenuItem>
  );
}
```

**Tips:**
- The router requires both `subscriberId` and `pageId` at `packages/api/src/router/pageSubscriber.ts:323-327`.
- Keep webhook-only visibility in the UI, but rely on the backend for real enforcement.
- If row actions are not client components, you may need to add `"use client"`.

**What Could Go Wrong:**
- Showing the action for email subscribers creates a guaranteed backend error.
- Forgetting `pageId` fails validation.
- Calling a public route bypasses dashboard ownership checks.

**Suggested Checks:**
- Webhook row shows action.
- Email row does not.
- Invalid webhook returns a readable error.
- Successful send gives feedback.

**Review Questions:**
- Did you call the protected procedure?
- Is the action hidden for unsupported rows?
- Can the user tell whether the test worked?

## Story 14: Add Notification Provider Filtering To The API

**Difficulty:** Medium-Hard  
**Estimated Time:** 6 hours  
**Skills You'll Practice:** router input, service schema extension, dashboard query input, tests  
**The Story:** As a dashboard user, I want to filter notifications by provider so that I can quickly review all Slack, Discord, or webhook notifiers.

**Acceptance Criteria:**
- [ ] `notification.list` accepts optional provider filter.
- [ ] Dashboard can select a provider and pass it to the query.
- [ ] Existing unfiltered notification list still works.
- [ ] Router or service tests cover filtered and unfiltered behavior.

**Read These Anchors First:**
- `packages/api/src/router/notification.ts:53-70` - current `notification.list` has no input and returns all items.
- `packages/api/src/router/notification.ts:99-123` - provider enum is already used by create.
- `apps/dashboard/src/app/(dashboard)/notifications/client.tsx:37-49` - dashboard notification query and create mutation.
- `apps/dashboard/src/app/(dashboard)/notifications/client.tsx:64-71` - table render.
- `packages/services/src/notification/list.ts` - service list implementation.
- `packages/services/src/notification/schemas.ts` - service list input schema if present.

**Files You'll Likely Touch:**
- `packages/services/src/notification/schemas.ts` - add provider filter if the service owns list input.
- `packages/services/src/notification/list.ts` - apply provider filter.
- `packages/api/src/router/notification.ts` - accept optional provider and pass it to service.
- `apps/dashboard/src/app/(dashboard)/notifications/client.tsx` - add provider select and query input.
- `packages/services/src/notification/__tests__/notification.test.ts` or `packages/api/src/router/notification.test.ts` - add coverage.

**Detailed Implementation Plan:**
1. Read the service list function before changing the router. If it already has filter support, use it.
2. Add `provider?: z.enum(notificationProvider)` to the canonical service list input if needed.
3. In `list.ts`, add a conditional `where` clause for provider while preserving workspace scoping.
4. Change `notificationRouter.list` from no input to optional input.
5. Pass provider into `listNotifications`.
6. In the dashboard, add local provider state and call `trpc.notification.list.queryOptions(provider ? { provider } : undefined)`.
7. Add a small select above the table.
8. Write a test that creates two notification providers and verifies filtering returns only one.

**Fake-Code Shape:**

```ts
// Router shape. Use actual repo imports and schemas.
list: protectedProcedure
  .input(z.object({ provider: z.enum(notificationProvider).optional() }).optional())
  .query(async ({ ctx, input }) => {
    const { items } = await listNotifications({
      ctx: toServiceCtx(ctx),
      input: {
        limit: 10_000,
        offset: 0,
        order: "desc",
        provider: input?.provider,
      },
    });
    return items;
  });
```

**Tips:**
- Provider enum comes from DB schema imports in `packages/api/src/router/notification.ts:4-17`.
- Keep the old unfiltered call valid so existing clients do not break.
- Tests should prove workspace scoping still applies, not only provider filtering.

**What Could Go Wrong:**
- Adding provider filter only in the dashboard filters already-loaded data but does not satisfy the API story.
- Forgetting optional input breaks existing `trpc.notification.list.queryOptions()` calls.
- Service `where` logic can accidentally drop workspace scoping if rewritten carelessly.

**Suggested Checks:**
- Unfiltered dashboard list still loads.
- Filtering by one provider returns only that provider.
- TypeScript catches invalid provider strings.

**Review Questions:**
- Is validation at the boundary?
- Is filtering owned by the service rather than only the UI?
- Did tests cover old and new behavior?

## Story 15: Add A Router Test For Monitor Bulk Visibility Updates

**Difficulty:** Medium  
**Estimated Time:** 5 hours  
**Skills You'll Practice:** testing existing behavior, tRPC router tests, regression protection  
**The Story:** As a maintainer, I want a test around bulk monitor visibility updates so that future changes do not accidentally expose or hide the wrong monitors.

**Acceptance Criteria:**
- [ ] Test calls the existing `monitor.updateMonitors` route.
- [ ] Test updates at least two monitors.
- [ ] Test verifies `public` changed for selected monitors.
- [ ] Test verifies unselected monitor is unchanged.
- [ ] Production code is not modified unless a bug is found.

**Read These Anchors First:**
- `packages/api/src/router/monitor.ts:108-129` - `updateMonitors` route.
- `packages/api/src/router/monitor.ts:131-148` - list route used to verify results.
- `packages/api/src/router/monitor.test.ts` - existing monitor router test patterns.
- `packages/services/src/monitor/__tests__/monitor.test.ts` - service-level monitor test patterns.

**Files You'll Likely Touch:**
- `packages/api/src/router/monitor.test.ts` - add a regression test.

**Detailed Implementation Plan:**
1. Open the existing monitor router test file and find how it creates caller/context.
2. Create or load three monitors in the test workspace.
3. Call `caller.monitor.updateMonitors({ ids: [id1, id2], public: true })`.
4. Fetch monitors through `caller.monitor.list()` or direct DB query, matching existing test style.
5. Assert selected monitors are public.
6. Assert third monitor is unchanged.
7. Add a second assertion for `active` only if test setup is simple.

**Fake-Code Shape:**

```ts
test("bulk updates public visibility only for selected monitors", async () => {
  const caller = createCaller(ctx);
  const first = await caller.monitor.new(makeMonitorInput({ name: "first" }));
  const second = await caller.monitor.new(makeMonitorInput({ name: "second" }));
  const third = await caller.monitor.new(makeMonitorInput({ name: "third" }));

  await caller.monitor.updateMonitors({
    ids: [first.id, second.id],
    public: true,
  });

  const monitors = await caller.monitor.list();
  expect(monitors.find((m) => m.id === first.id)?.public).toBe(true);
  expect(monitors.find((m) => m.id === second.id)?.public).toBe(true);
  expect(monitors.find((m) => m.id === third.id)?.public).toBe(false);
});
```

**Tips:**
- `updateMonitors` accepts optional `public` and `active` fields (`packages/api/src/router/monitor.ts:108-115`).
- The route delegates to `bulkUpdateMonitors` with the selected ids (`packages/api/src/router/monitor.ts:116-125`).
- Reuse existing test helpers rather than inventing a new database setup.

**What Could Go Wrong:**
- Creating active monitors may trigger endpoint checks if you use the route incorrectly. Inspect existing tests for `skipCheck`.
- List order can vary; assert by id, not array position.
- If existing fixtures already have monitors, make your test names unique.

**Suggested Checks:**
- Run the monitor router test file.
- Confirm the test fails if you intentionally remove `ids` from the service call, then undo that experiment.

**Review Questions:**
- Does the test protect real behavior?
- Is it stable regardless of list order?
- Did you avoid changing production code for a test-only story?

