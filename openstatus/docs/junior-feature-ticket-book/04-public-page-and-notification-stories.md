# Public Page And Notification Stories

Stories 16-20 practice flows where mistakes are visible outside the dashboard: public status pages, subscriber management, feed rendering, and notification test behavior.

## Story 16: Add A Public Feed Empty State

**Difficulty:** Medium  
**Estimated Time:** 3 hours  
**Skills You'll Practice:** public rendering, derived feed data, conditional UI  
**The Story:** As a public status-page visitor, I want to see a calm empty state when there are no recent incidents or maintenance windows so that I know the feed is intentionally empty.

**Acceptance Criteria:**
- [ ] Public page displays a feed empty state when both filtered reports and maintenances are empty.
- [ ] Empty state is hidden when embed mode hides the feed.
- [ ] Existing `StatusFeed` rendering is unchanged when there are feed items.
- [ ] No API changes are made.

**Read These Anchors First:**
- `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:117-129` - open event filtering.
- `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:329-360` - feed reports and maintenances are filtered inline.
- `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:328-361` - feed content is wrapped in embed-hide class.

**Files You'll Likely Touch:**
- `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx` - extract feed arrays and render empty state.

**Detailed Implementation Plan:**
1. Extract `feedReports` and `feedMaintenances` exactly as Story 5 describes.
2. Compute `const hasFeedItems = feedReports.length > 0 || feedMaintenances.length > 0`.
3. Inside the existing feed `StatusContent`, render either `StatusFeed` or a small empty message.
4. Keep the wrapper class `group-data-[hide-feed=true]/embed:hidden`.
5. Use domain-specific copy: "No recent incidents or maintenance windows."
6. Do not modify `page.lastEvents`; it is part of the status-page data contract.

**Fake-Code Shape:**

```tsx
<StatusContent className="group-data-[hide-feed=true]/embed:hidden">
  {hasFeedItems ? (
    <StatusFeed statusReports={feedReports} maintenances={feedMaintenances} />
  ) : (
    <div className="text-sm text-muted-foreground">
      No recent incidents or maintenance windows.
    </div>
  )}
</StatusContent>
```

**Tips:**
- Reports without updates are excluded intentionally at `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:331-339`.
- Maintenances are filtered against `page.lastEvents` at `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:347-353`.
- Keep public copy calm and factual.

**What Could Go Wrong:**
- Showing the empty state outside the embed-hide wrapper makes embedded pages show hidden sections.
- Counting raw reports can show feed items that are not actually rendered.

**Suggested Checks:**
- Public page with no feed items shows empty state.
- Public page with one incident shows `StatusFeed`.
- Embed URL that hides feed still hides the whole section.

**Review Questions:**
- Does the branch use the same arrays as `StatusFeed`?
- Did you preserve the public page query?
- Is the empty state useful without implying a problem?

## Story 17: Add Affected Component Count To Public Event Banner Tabs

**Difficulty:** Medium  
**Estimated Time:** 4 hours  
**Skills You'll Practice:** public event rendering, relation counts, safe display changes  
**The Story:** As a public status-page visitor, I want banner tabs to show how many components are affected so that I can quickly judge incident impact.

**Acceptance Criteria:**
- [ ] Report banner tabs can show affected component count.
- [ ] Maintenance banner tabs can show affected component count.
- [ ] Incident synthetic banner still renders without component count.
- [ ] Links and tab values remain unchanged.

**Read These Anchors First:**
- `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:143-164` - banner tab trigger loop.
- `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:165-211` - report banner content and affected components.
- `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:212-252` - maintenance banner content and affected components.
- `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:253-262` - incident fallback banner.

**Files You'll Likely Touch:**
- `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx` - derive count per event and render it in the tab trigger.

**Detailed Implementation Plan:**
1. Inside the `events.map` tab-trigger loop, write a helper function or inline block to find the matching report/maintenance.
2. For `e.type === "report"`, find the report in `page.statusReports` and count `statusReportsToPageComponents`.
3. For `e.type === "maintenance"`, find the maintenance in `page.maintenances` and count `maintenancesToPageComponents`.
4. For `incident`, return `null` count.
5. Render compact text after `e.name`, such as `(3 affected)`.
6. Do not change `value={`${e.type}-${e.id}`}` or keys.

**Fake-Code Shape:**

```tsx
function getAffectedCount(event: EventLike) {
  if (event.type === "report") {
    return page.statusReports.find((report) => report.id === event.id)
      ?.statusReportsToPageComponents.length;
  }
  if (event.type === "maintenance") {
    return page.maintenances.find((maintenance) => maintenance.id === event.id)
      ?.maintenancesToPageComponents.length;
  }
  return undefined;
}
```

**Tips:**
- The full affected component names already render inside report content at `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:193-205`.
- Maintenance affected names render similarly at `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:233-246`.
- Do not show `(0 affected)` unless product wants that; "all services" may be clearer for zero associations.

**What Could Go Wrong:**
- Doing repeated `.find` calls is fine for small event arrays, but if it gets messy, build a map before render.
- Changing tab values breaks selected tab content.

**Suggested Checks:**
- Report event with two affected components shows count.
- Maintenance event with one affected component shows singular copy.
- Incident event still shows only incident name.

**Review Questions:**
- Did you keep derived display close to the data?
- Did you avoid changing event identity?
- Is zero-count copy domain-accurate?

## Story 18: Improve Manage Subscription Scope Copy

**Difficulty:** Medium  
**Estimated Time:** 3 hours  
**Skills You'll Practice:** public form UX, subscriber scope, safe copy changes  
**The Story:** As a subscribed user managing my status-page subscription, I want the page to explain whether I am subscribed to the whole page or selected components so that I understand what alerts I will receive.

**Acceptance Criteria:**
- [ ] Manage subscription page distinguishes whole-page subscription from component-scoped subscription.
- [ ] Existing update-scope mutation is unchanged.
- [ ] Existing unsubscribe behavior is unchanged.
- [ ] Copy is visible before saving changes.

**Read These Anchors First:**
- `packages/api/src/router/pageSubscriber.ts:141-175` - public `getByToken` route returns subscription details.
- `packages/api/src/router/pageSubscriber.ts:180-207` - public `updateScope` route.
- `packages/api/src/router/pageSubscriber.ts:212-228` - public unsubscribe route.
- `apps/status-page/src/components/forms/form-manage-subscription.tsx` - management form UI.
- `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/manage/[token]/page.tsx` - page shell.

**Files You'll Likely Touch:**
- `apps/status-page/src/components/forms/form-manage-subscription.tsx` - add explanatory copy based on selected component ids.

**Detailed Implementation Plan:**
1. Open the manage subscription form and identify where selected component ids are stored.
2. Derive whether the current selection means whole-page subscription. In this app, subscriber routes accept `componentIds` arrays, and absence/empty arrays often imply whole-page scope; verify in the form before changing copy.
3. Render copy near the checkbox tree or save button.
4. Keep the mutation payload for `updateScope` unchanged.
5. Confirm unsubscribe still calls the route from `packages/api/src/router/pageSubscriber.ts:212-228`.

**Fake-Code Shape:**

```tsx
const selectedIds = form.watch("componentIds");
const scopedToComponents = selectedIds.length > 0;

<p className="text-sm text-muted-foreground">
  {scopedToComponents
    ? `You will receive alerts for ${selectedIds.length} selected components.`
    : "You will receive alerts for the entire status page."}
</p>
```

**Tips:**
- Public subscriber routes are not protected by dashboard auth, so copy must be especially clear.
- `getByToken` returns `componentIds` at `packages/api/src/router/pageSubscriber.ts:161-171`.
- `updateScope` returns the updated component ids at `packages/api/src/router/pageSubscriber.ts:198-203`.

**What Could Go Wrong:**
- Misinterpreting empty array semantics can tell users the opposite of what they will receive.
- Moving the save button can break form submission.

**Suggested Checks:**
- Subscription with no selected components shows whole-page copy.
- Subscription with selected components shows count.
- Saving scope still works.

**Review Questions:**
- Did you verify scope semantics from code?
- Is the copy visible before the user saves?
- Did you leave mutations untouched?

## Story 19: Add Notification Test Result Details

**Difficulty:** Medium-Hard  
**Estimated Time:** 5 hours  
**Skills You'll Practice:** mutation feedback, provider validation, error messages  
**The Story:** As a dashboard user configuring a notifier, I want test-send feedback to include the provider name so that I know which integration succeeded or failed.

**Acceptance Criteria:**
- [ ] Test-send success message includes provider label.
- [ ] Test-send failure message includes provider label or context.
- [ ] Existing provider-specific validation remains in the router.
- [ ] No secrets are shown in the UI.

**Read These Anchors First:**
- `packages/api/src/router/notification.ts:142-321` - `sendTest` provider-specific validation and dispatch.
- `apps/dashboard/src/components/forms/notifications/form-slack.tsx` - one provider form with test-send behavior.
- `apps/dashboard/src/components/forms/notifications/form-webhook.tsx` - generic webhook form.
- `apps/dashboard/src/components/forms/notifications/form.tsx` - shared notification form composition.
- `apps/dashboard/src/app/(dashboard)/notifications/client.tsx:131-160` - `FormSheetNotifier` provider and submit payload.

**Files You'll Likely Touch:**
- Provider-specific notification form files that expose "send test".
- Possibly shared notification form components if there is a shared test-send helper.

**Detailed Implementation Plan:**
1. Search notification form files for `sendTest`.
2. Identify whether success/error toast logic is duplicated or shared.
3. If shared, update the shared helper once. If duplicated, update one provider first and then repeat carefully.
4. Use the provider label from `config` or the provider prop; do not hardcode secrets or webhook URLs.
5. Keep the router's provider-specific validation untouched.
6. Confirm errors from `TRPCError` still show useful messages.

**Fake-Code Shape:**

```tsx
const label = config[provider].label;
const sendTestMutation = useMutation(
  trpc.notification.sendTest.mutationOptions({
    onSuccess: () => toast.success(`${label} test notification sent`),
    onError: (error) => toast.error(`${label} test failed: ${error.message}`),
  }),
);
```

**Tips:**
- Router validation branches by provider at `packages/api/src/router/notification.ts:150-321`.
- The Slack branch validates `slackDataSchema` before sending at `packages/api/src/router/notification.ts:226-236`.
- The webhook branch sends `endpoint` and `headers` at `packages/api/src/router/notification.ts:238-251`; never echo those values in toast copy.

**What Could Go Wrong:**
- Showing raw config data can expose tokens or webhook URLs.
- Updating only one provider may create inconsistent UX.
- Catching errors too broadly can hide useful schema messages.

**Suggested Checks:**
- Send test for one valid provider.
- Trigger validation error for one provider.
- Confirm no secret values appear in toast or console.

**Review Questions:**
- Does feedback identify the provider?
- Are secrets hidden?
- Did router validation remain server-side?

## Story 20: Add Public Page Password Help Text

**Difficulty:** Medium  
**Estimated Time:** 3 hours  
**Skills You'll Practice:** auth-adjacent UX, public form copy, preserving security behavior  
**The Story:** As a visitor to a restricted status page, I want password help text that tells me what to do if I do not have access so that I do not repeatedly retry.

**Acceptance Criteria:**
- [ ] Restricted login page shows short help text.
- [ ] Password verification behavior is unchanged.
- [ ] Auth email domain or IP restriction behavior is unchanged.
- [ ] Help text does not reveal whether a guessed password is close.

**Read These Anchors First:**
- `packages/api/src/router/statusPage.ts:1214-1247` - public password verification route.
- `apps/status-page/src/app/(status-page)/[domain]/[locale]/(auth)/login/page.tsx` - restricted page login route.
- `apps/status-page/src/components/forms/form-password.tsx` - password form UI.
- `packages/api/src/router/page.ts:268-299` - dashboard-side password protection configuration.

**Files You'll Likely Touch:**
- `apps/status-page/src/components/forms/form-password.tsx` - add help text.
- Possibly `apps/status-page/src/app/(status-page)/[domain]/[locale]/(auth)/login/page.tsx` - if the page shell owns descriptive copy.

**Detailed Implementation Plan:**
1. Open the password form and find current error/help copy.
2. Add neutral help text like "Contact the page owner if you need access."
3. Do not change the mutation input or verification route.
4. Do not add copy that reveals policy internals, allowed domains, or IP rules.
5. Confirm the form still submits and displays errors as before.

**Fake-Code Shape:**

```tsx
<FormDescription>
  Enter the password shared by the status page owner. If you do not have one,
  contact the organization that sent you this page.
</FormDescription>
```

**Tips:**
- Password protection is configured from dashboard through `updatePasswordProtection` at `packages/api/src/router/page.ts:268-299`.
- Public verification is separate and lives under `statusPage.verifyPassword` at `packages/api/src/router/statusPage.ts:1214-1247`.
- Keep help text generic to avoid leaking access-control details.

**What Could Go Wrong:**
- Mentioning exact allowed domains or IP ranges can leak policy details.
- Changing the route call can break restricted pages.

**Suggested Checks:**
- Restricted page still accepts a valid password.
- Invalid password still shows failure.
- Help text appears before repeated retries.

**Review Questions:**
- Is the copy useful but not revealing?
- Did you preserve security behavior?
- Did you change only the public form surface?

