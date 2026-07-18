# User Stories

## Story 1: Clarify the Unresolved Report Warning
**Difficulty:** Easy
**Estimated Time:** 1 hour
**Skills You'll Practice:** React copy changes, conditional UI, reading component props
**The Story:** As a dashboard user, I want the unresolved-report warning to tell me what action to take so that I do not accidentally create duplicate incident reports.
**Acceptance Criteria:**
- [ ] The warning still appears only when at least one report is not `resolved`.
- [ ] The copy explicitly recommends adding a status report update instead of creating a new report.
- [ ] The create button and form behavior remain unchanged.
**Files You'll Likely Touch:** `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx` - warning is constructed and passed into `FormSheetStatusReport` (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:56-84`).
**High-Level Implementation Plan:**
1. Open `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx` and find `hasUnresolvedIssue`.
2. Edit only the JSX passed to `warning`.
3. Keep `hasUnresolvedIssue` logic unchanged.
4. Verify the create button still renders inside `FormSheetStatusReport`.
**Tips:**
- The unresolved check is `report.status !== "resolved"` (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:56-58`).
- The sheet simply displays whatever `warning` receives (`apps/dashboard/src/components/forms/status-report/sheet.tsx:46-51`).
- Do not change mutation code in this story (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:36-51`).
**What Could Go Wrong:**
- If you change the condition, the warning may appear for resolved reports.
- If you move the button, the form trigger may stop opening the sheet.
**Stretch Goal:** Add a link to the selected report's expanded updates row if the data table exposes a stable route or row action.
**Connects To:** Story 2, because you will keep improving status-report form guidance.

## Story 2: Improve the Empty Components Message
**Difficulty:** Easy
**Estimated Time:** 1 hour
**Skills You'll Practice:** Form UI, empty states, reading shared components
**The Story:** As a dashboard user creating a report, I want clearer guidance when a status page has no components so that I understand why I cannot select affected services.
**Acceptance Criteria:**
- [ ] The empty state still renders only when `items.length` is zero.
- [ ] The title explains that no page components exist for this status page.
- [ ] The form still submits with an empty `pageComponents` array when allowed by validation.
**Files You'll Likely Touch:** `apps/dashboard/src/components/forms/status-report/form.tsx` - empty state appears inside the `pageComponents` field (`apps/dashboard/src/components/forms/status-report/form.tsx:353-379`).
**High-Level Implementation Plan:**
1. Open `FormStatusReport`.
2. Find the `items.length ? <CheckboxTree /> : <EmptyStateContainer />` branch.
3. Update the title text only.
4. Confirm the `FormField` name remains `pageComponents`.
**Tips:**
- The field is named `pageComponents` in the form schema (`apps/dashboard/src/components/forms/status-report/form.tsx:62-69`).
- The empty branch currently uses `EmptyStateContainer` and `EmptyStateTitle` (`apps/dashboard/src/components/forms/status-report/form.tsx:371-374`).
- The tRPC router expects `pageComponents` from the dashboard (`packages/api/src/router/statusReport.ts:24-32`).
**What Could Go Wrong:**
- Removing the `FormField` wrapper can break validation messages.
- Renaming the field breaks the create payload.
**Stretch Goal:** Add a secondary description if the existing empty-state components support it.
**Connects To:** Story 3, because both stories practice safe UI-only changes.

## Story 3: Add Timezone Copy To the Date Field
**Difficulty:** Easy
**Estimated Time:** 1.5 hours
**Skills You'll Practice:** Form descriptions, browser APIs, date UX
**The Story:** As a dashboard user, I want the report date field to clearly state my timezone so that I trust the timestamp saved for incident communication.
**Acceptance Criteria:**
- [ ] The date field still displays the resolved browser timezone.
- [ ] The copy explains that the display is local but saved as UTC/Unix time.
- [ ] Date selection and time input behavior are unchanged.
**Files You'll Likely Touch:** `apps/dashboard/src/components/forms/status-report/form.tsx` - timezone is computed and rendered in date description (`apps/dashboard/src/components/forms/status-report/form.tsx:94-95`; `apps/dashboard/src/components/forms/status-report/form.tsx:300-310`).
**High-Level Implementation Plan:**
1. Locate `const timezone = Intl.DateTimeFormat().resolvedOptions().timeZone`.
2. Locate the `FormDescription` under the `date` field.
3. Update only display copy around the existing timezone and UTC code elements.
4. Do not change `field.onChange` logic for calendar or time input.
**Tips:**
- Date validation lives in the form schema (`apps/dashboard/src/components/forms/status-report/form.tsx:62-69`).
- The time input manually updates hours/minutes/seconds (`apps/dashboard/src/components/forms/status-report/form.tsx:269-290`).
- The service coerces dates again on the backend (`packages/services/src/status-report/schemas.ts:12-20`).
**What Could Go Wrong:**
- Editing date mutation logic can shift report times.
- Removing UTC wording can confuse incident timelines.
**Stretch Goal:** Add a tiny helper function for formatting timezone display if the copy becomes hard to read.
**Connects To:** Story 4, because the next story starts reading existing data to improve UI context.

## Story 4: Show Affected Component Count Before Submit
**Difficulty:** Medium
**Estimated Time:** 2.5 hours
**Skills You'll Practice:** React Hook Form watch, derived UI, form state
**The Story:** As a dashboard user, I want to see how many page components I selected before submitting a report so that I can catch mistakes before notifying users.
**Acceptance Criteria:**
- [ ] The form displays a selected component count near the `Page Components` field.
- [ ] The count updates when checkboxes change.
- [ ] The count uses the existing `pageComponents` form field.
- [ ] No API contract changes are made.
**Files You'll Likely Touch:** `apps/dashboard/src/components/forms/status-report/form.tsx` - owns form state and `CheckboxTree` (`apps/dashboard/src/components/forms/status-report/form.tsx:96-109`; `apps/dashboard/src/components/forms/status-report/form.tsx:353-379`). `apps/dashboard/src/components/ui/checkbox-tree` - read only if needed to understand `onValueChange` (`apps/dashboard/src/components/forms/status-report/form.tsx:14-16`).
**High-Level Implementation Plan:**
1. In `FormStatusReport`, use `form.watch("pageComponents")` near the existing `watchMessage`.
2. Derive `selectedCount` from the watched array.
3. Render concise count text inside the `pageComponents` `FormDescription` or near the field label.
4. Keep `CheckboxTree` value and `onValueChange` unchanged.
**Tips:**
- `watchMessage` already demonstrates watching a field (`apps/dashboard/src/components/forms/status-report/form.tsx:107-108`).
- `CheckboxTree` receives `value={field.value ?? []}` (`apps/dashboard/src/components/forms/status-report/form.tsx:363-369`).
- Backend create uses `pageComponentIds` after router adaptation (`packages/api/src/router/statusReport.ts:64-74`).
**What Could Go Wrong:**
- Watching the wrong field returns undefined and displays `NaN`.
- Rendering count outside the form field may make it harder for users to associate with the checkbox tree.
**Stretch Goal:** Show "all components selected" when selected count equals total selectable components.
**Connects To:** Story 5, because both use existing data to improve dashboard context.

## Story 5: Add Last Update Count To Status Report Rows
**Difficulty:** Medium
**Estimated Time:** 3 hours
**Skills You'll Practice:** Data tables, derived row values, existing API data
**The Story:** As a dashboard user, I want each status report row to show how many updates it contains so that I can quickly identify reports with active communication history.
**Acceptance Criteria:**
- [ ] The status report table displays an update count for each row.
- [ ] The count is derived from existing `updates` data, not a new API call.
- [ ] Expanded row behavior for update details still works.
**Files You'll Likely Touch:** `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx` - passes `statusReports` to the table and uses expanded row data (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:112-124`). `apps/dashboard/src/components/data-table/status-reports/columns.tsx` - status report table column definitions. `packages/services/src/status-report/list.ts` - read only to confirm `updates` relation exists (`packages/services/src/status-report/list.ts:49-64`; `packages/services/src/status-report/list.ts:152-161`).
**High-Level Implementation Plan:**
1. Open the status-report columns file.
2. Add a column that reads `row.original.updates.length`.
3. Keep the existing expanded row component unchanged.
4. Verify `updates` is already included by `listStatusReports`.
**Tips:**
- The page already passes `row.original.updates` into `UpdatesDataTable` (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:118-123`).
- The service enriches every report with `updates` (`packages/services/src/status-report/list.ts:49-64`; `packages/services/src/status-report/list.ts:152-161`).
- Do not add another tRPC query.
**What Could Go Wrong:**
- If the column assumes `updates` can never be missing, check the inferred row type first.
- If the column changes row expansion behavior, update counts may work while nested updates break.
**Stretch Goal:** Use singular/plural copy: "1 update" vs "2 updates."
**Connects To:** Story 6, because the next story adds a filter over existing report data.

## Story 6: Add A Status Filter To Status Reports
**Difficulty:** Medium
**Estimated Time:** 4 hours
**Skills You'll Practice:** UI state, tRPC input changes, service list filters
**The Story:** As a dashboard user, I want to filter status reports by status so that I can focus on unresolved incidents.
**Acceptance Criteria:**
- [ ] The dashboard UI lets users select one or more report statuses.
- [ ] The selected statuses affect the `statusReport.list` query.
- [ ] The backend validates statuses using the existing status enum.
- [ ] Existing pageId filtering still works.
**Files You'll Likely Touch:** `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx` - owns query state (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:23-32`). `packages/api/src/router/statusReport.ts` - list input currently omits statuses at tRPC boundary (`packages/api/src/router/statusReport.ts:189-218`). `packages/services/src/status-report/schemas.ts` - service already supports `statuses` (`packages/services/src/status-report/schemas.ts:79-87`). `packages/services/src/status-report/list.ts` - applies statuses filter (`packages/services/src/status-report/list.ts:172-181`).
**High-Level Implementation Plan:**
1. Add local UI state or URL search params for selected statuses in the status-reports page.
2. Extend `statusReport.list` tRPC input to accept optional statuses using `CreateStatusReportInput.shape.status` or service schema equivalent.
3. Pass statuses through to `listStatusReports`.
4. Add UI controls using existing select/checkbox patterns.
5. Add or update a router test for filtering if practical.
**Tips:**
- The service already supports `statuses` arrays (`packages/services/src/status-report/schemas.ts:79-87`).
- The tRPC router currently passes `statuses: []` unconditionally (`packages/api/src/router/statusReport.ts:199-211`).
- Reuse the status enum from DB/service, not hardcoded strings (`packages/db/src/schema/status_reports/status_reports.ts:8-13`).
**What Could Go Wrong:**
- Passing `undefined` instead of an array may violate the service schema if not defaulted.
- Forgetting `pageId` causes reports from all pages in the workspace to show.
- UI state can reset on navigation if not encoded in URL.
**Stretch Goal:** Add a quick "Unresolved" preset that selects investigating, identified, and monitoring.
**Connects To:** Story 7, because both add dashboard state that changes server queries.

## Story 7: Add Period Filtering To The Dashboard Report List
**Difficulty:** Medium
**Estimated Time:** 4 hours
**Skills You'll Practice:** query params, existing service options, typed filters
**The Story:** As a dashboard user, I want to view reports from the last 1, 7, or 14 days so that I can review recent incidents without scanning the full history.
**Acceptance Criteria:**
- [ ] The UI exposes 1d, 7d, and 14d filter options.
- [ ] The selected period is passed to `statusReport.list`.
- [ ] The service's existing `period` behavior is used.
- [ ] Clearing the filter shows the default list again.
**Files You'll Likely Touch:** `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx` - report query call (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:30-32`). `packages/api/src/router/statusReport.ts` - list input already accepts `period` (`packages/api/src/router/statusReport.ts:189-218`). `packages/services/src/status-report/schemas.ts` - allowed periods (`packages/services/src/status-report/schemas.ts:7-10`). `packages/services/src/status-report/list.ts` - period maps to date threshold (`packages/services/src/status-report/list.ts:36-47`; `packages/services/src/status-report/list.ts:179-181`).
**High-Level Implementation Plan:**
1. Add period state to the status-reports page.
2. Render a small select or segmented control using existing UI components.
3. Pass `period` into `trpc.statusReport.list.queryOptions`.
4. Make clearing period pass `undefined`.
5. Verify the service behavior does not need changes.
**Tips:**
- Period values are `1d`, `7d`, and `14d` (`packages/services/src/status-report/schemas.ts:7-10`).
- tRPC already validates periods through `periods` in router list input (`packages/api/src/router/statusReport.ts:189-195`).
- The service translates period to `createdAt >= since` (`packages/services/src/status-report/list.ts:36-47`; `packages/services/src/status-report/list.ts:179-181`).
**What Could Go Wrong:**
- Filtering by `createdAt` may not match "last update date"; understand that behavior before changing copy.
- If you keep stale query keys, React Query may not refetch when period changes.
**Stretch Goal:** Encode period in the URL so the filter is shareable.
**Connects To:** Story 8, because the next story adds a persisted domain field.

## Story 8: Add Severity To Status Reports
**Difficulty:** Hard
**Estimated Time:** 8 hours
**Skills You'll Practice:** schema design, migration thinking, form changes, API contracts, tests
**The Story:** As a dashboard user, I want to assign severity to a status report so that public viewers understand the impact level.
**Acceptance Criteria:**
- [ ] New reports can be assigned a severity such as minor, major, or critical.
- [ ] Severity is persisted with the status report.
- [ ] Dashboard list displays severity.
- [ ] Public status-page feed or banner displays severity.
- [ ] Existing reports have a safe default.
**Files You'll Likely Touch:** `packages/db/src/schema/status_reports/status_reports.ts` - add persisted column (`packages/db/src/schema/status_reports/status_reports.ts:15-39`). `packages/db/src/schema/status_reports/validation.ts` - update row schemas/types (`packages/db/src/schema/status_reports/validation.ts:38-55`). `packages/services/src/status-report/schemas.ts` - add service input field (`packages/services/src/status-report/schemas.ts:12-20`). `packages/services/src/status-report/create.ts` - insert severity (`packages/services/src/status-report/create.ts:54-63`). `packages/api/src/router/statusReport.ts` - accept/adapt tRPC input (`packages/api/src/router/statusReport.ts:24-81`). `apps/dashboard/src/components/forms/status-report/form.tsx` - add form control (`apps/dashboard/src/components/forms/status-report/form.tsx:62-77`; `apps/dashboard/src/components/forms/status-report/form.tsx:139-415`). `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx` - send form value (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:89-103`). `packages/api/src/router/statusPage.ts` and public client - render severity (`packages/api/src/router/statusPage.ts:80-113`; `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:329-360`).
**High-Level Implementation Plan:**
1. Add a severity enum/column in the DB schema with a default.
2. Update DB validation and inferred types.
3. Update service create/update schemas and insert/update logic.
4. Update tRPC schemas and preserve backward-compatible defaulting.
5. Add a severity select in `FormStatusReport`.
6. Pass severity from page mutation to router.
7. Display severity in dashboard table and public feed/banner.
8. Add tests for create default and explicit severity.
**Tips:**
- Follow the existing `statusReportStatus` enum pattern (`packages/db/src/schema/status_reports/status_reports.ts:8-20`).
- Keep service schema canonical and router schema adapter-friendly (`packages/api/src/router/statusReport.ts:21-23`).
- Public output parses through `selectPublicPageSchemaWithRelation`, so type/schema updates may need to flow there (`packages/api/src/router/statusPage.ts:385-400`).
**What Could Go Wrong:**
- DB default and service default can disagree.
- Public output schema can reject the new field if not updated.
- Existing tests may fail if fixtures do not include severity.
**Stretch Goal:** Add severity-based ordering for open events on the public banner.
**Connects To:** Story 9, because both require full-stack domain changes.

## Story 9: Add Draft Status Reports
**Difficulty:** Hard
**Estimated Time:** 10 hours
**Skills You'll Practice:** workflow design, DB schema, service rules, public/private visibility
**The Story:** As a dashboard user, I want to save a draft status report before publishing so that I can prepare incident communication without exposing it publicly.
**Acceptance Criteria:**
- [ ] Dashboard users can create a draft report.
- [ ] Draft reports are visible in the dashboard list.
- [ ] Draft reports are not shown on the public status page.
- [ ] Publishing a draft makes it public and creates or preserves the initial update.
- [ ] Tests prove drafts do not leak publicly.
**Files You'll Likely Touch:** `packages/db/src/schema/status_reports/status_reports.ts` - add draft/published state (`packages/db/src/schema/status_reports/status_reports.ts:15-39`). `packages/services/src/status-report/create.ts` - create behavior (`packages/services/src/status-report/create.ts:24-99`). `packages/services/src/status-report/update.ts` - publish/update behavior. `packages/api/src/router/statusReport.ts` - tRPC create/update contract (`packages/api/src/router/statusReport.ts:58-142`). `apps/dashboard/src/components/forms/status-report/form.tsx` - draft UI (`apps/dashboard/src/components/forms/status-report/form.tsx:381-413`). `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx` - list and create behavior (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:23-128`). `packages/api/src/router/statusPage.ts` - filter drafts from public data (`packages/api/src/router/statusPage.ts:80-113`; `packages/api/src/router/statusPage.ts:364-400`). `packages/api/src/router/statusReport.test.ts` and/or statusPage tests - visibility tests (`packages/api/src/router/statusReport.test.ts:77-207`).
**High-Level Implementation Plan:**
1. Decide whether draft is a boolean column or a new lifecycle enum.
2. Add schema/default so existing reports remain public.
3. Update service create to accept draft state.
4. Add publish/update service behavior.
5. Update tRPC create/update inputs.
6. Add dashboard UI for "Save draft" vs "Publish".
7. Filter drafts from `statusPage.get`.
8. Add tests proving dashboard sees drafts and public page does not.
**Tips:**
- Public page gets reports through relation loading inside `statusPage.get` (`packages/api/src/router/statusPage.ts:80-113`).
- Dashboard `statusReport.list` should probably include drafts for owners (`packages/api/src/router/statusReport.ts:189-222`).
- Be careful with notification: drafts should not trigger subscriber email (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`).
**What Could Go Wrong:**
- Drafts can leak through public `lastEvents` if only UI rendering filters them.
- Notifications can send for drafts if the create path is not guarded.
- Publishing can create duplicate initial updates if workflow is unclear.
**Stretch Goal:** Add a "scheduled publish time" after draft support works.
**Connects To:** Story 10, because draft/publish reliability raises architectural questions.

## Story 10: Move Report Notifications To A Reliable Server-Owned Workflow
**Difficulty:** Expert
**Estimated Time:** 16+ hours
**Skills You'll Practice:** architecture design, side effects, retries, service boundaries, tests, migration planning
**The Story:** As a workspace owner, I want subscriber notifications for status reports to be handled reliably on the server so that incident communication does not depend on a browser staying open after creation.
**Acceptance Criteria:**
- [ ] Creating a report with notification enabled queues or performs notification server-side.
- [ ] The dashboard no longer owns the email-send side effect after create success.
- [ ] Report creation behavior remains backward-compatible for callers.
- [ ] Notification failures are observable and do not silently disappear.
- [ ] Tests cover create-with-notify, create-without-notify, and notification failure behavior.
**Files You'll Likely Touch:** `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx` - remove or change client-side notification side effect (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`). `packages/api/src/router/statusReport.ts` - accept notify flag and pass to appropriate server path (`packages/api/src/router/statusReport.ts:24-81`). `packages/services/src/status-report/create.ts` - consider service ownership or event emission (`packages/services/src/status-report/create.ts:24-99`). `packages/services/src/status-report/notify.ts` - existing notification behavior. `packages/services/src/status-report/__tests__/status-report.test.ts` - subscription spies already exist (`packages/services/src/status-report/__tests__/status-report.test.ts:41-56`). `packages/subscriptions/src/dispatcher.ts` - dispatches subscriber channels (`packages/subscriptions/src/dispatcher.ts:15-63`; `packages/subscriptions/src/dispatcher.ts:130-130`). `packages/api/src/router/email/index.ts` - current email router path used by dashboard (`packages/api/src/lambda.ts:1-14`).
**High-Level Implementation Plan:**
1. Document current behavior and failure policy from dashboard `onSuccess`.
2. Decide whether notification belongs in service create, a post-commit job, or a dedicated workflow.
3. Keep the tRPC input backward-compatible with `notifySubscribers`.
4. Move side effect off the browser path.
5. Ensure DB writes and notification dispatch have a clear transaction/commit boundary.
6. Add service tests using existing subscription spies.
7. Remove dashboard dependency on `emailRouter.sendStatusReport` for create.
8. Add observability for notification failures.
**Tips:**
- Current code has a TODO saying "move to server" (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:38-40`).
- Service tests already mock subscription dispatch (`packages/services/src/status-report/__tests__/status-report.test.ts:41-56`).
- `lambdaRouter` owns email-related router namespaces today (`packages/api/src/lambda.ts:1-14`), while `statusReport` is edge (`packages/api/src/edge.ts:25-49`).
- Be deliberate about Edge vs Node runtime if email SDKs or queues require Node APIs (`apps/status-page/src/app/api/trpc/lambda/[trpc]/route.ts:9-21`).
**What Could Go Wrong:**
- Sending inside the transaction can notify subscribers for rolled-back reports.
- Sending after commit without a durable queue can still lose notifications.
- Moving to lambda may require client routing changes or router relocation.
**Stretch Goal:** Add a notification delivery audit trail to the dashboard.
**Connects To:** This is the capstone story: it forces you to reason about architecture, reliability, and product trust.

