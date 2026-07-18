# Full-Stack Growth Stories

Stories 21-25 are larger. They are still written for a junior developer, but they require more deliberate sequencing: database/schema, service, router, frontend, tests, and rollout thinking.

## Story 21: Add Internal Notes To Status Reports

**Difficulty:** Hard  
**Estimated Time:** 8-10 hours  
**Skills You'll Practice:** DB schema, service input, dashboard-only fields, public/private separation  
**The Story:** As a dashboard user, I want to add an internal note to a status report so that my team can track operator context without showing it on the public status page.

**Acceptance Criteria:**
- [ ] Status reports can store an optional internal note.
- [ ] Dashboard create/edit flows can set and view the internal note.
- [ ] Public status page never displays the internal note.
- [ ] Existing status reports without notes still work.
- [ ] Tests verify the field is saved but excluded from public rendering.

**Read These Anchors First:**
- `packages/db/src/schema/status_reports/status_reports.ts:15-39` - status report table.
- `packages/db/src/schema/status_reports/validation.ts:21-55` - insert/select schemas and exported types.
- `packages/services/src/status-report/schemas.ts:12-20` - service create input.
- `packages/services/src/status-report/create.ts:54-63` - status report insert.
- `packages/api/src/router/statusReport.ts:58-81` - create route adapts dashboard input to service input.
- `apps/dashboard/src/components/forms/status-report/form.tsx` - create/edit form surface.
- `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:329-360` - public feed mapping.

**Files You'll Likely Touch:**
- `packages/db/src/schema/status_reports/status_reports.ts`
- `packages/db/src/schema/status_reports/validation.ts`
- `packages/services/src/status-report/schemas.ts`
- `packages/services/src/status-report/create.ts`
- `packages/services/src/status-report/update.ts`
- `packages/api/src/router/statusReport.ts`
- `apps/dashboard/src/components/forms/status-report/form.tsx`
- `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx`
- status-report tests in `packages/services/src/status-report/__tests__/status-report.test.ts` and/or `packages/api/src/router/statusReport.test.ts`

**Detailed Implementation Plan:**
1. Add nullable `internalNote` text column to the status report DB schema.
2. Generate or write the migration using the repo's DB workflow. Inspect existing migrations before adding one.
3. Update insert/select validation schemas so TypeScript knows the field exists.
4. Add `internalNote: z.string().max(...).nullish()` to the service create/update schemas.
5. In `createStatusReport`, insert `internalNote: input.internalNote ?? null`.
6. In update logic, allow changing `internalNote` without changing public report updates.
7. In `statusReportRouter.create`, accept the field and pass it through.
8. Add a textarea in the dashboard form with label "Internal note".
9. Pass form value from page submit handler into the mutation.
10. Verify public page mapping does not use `internalNote`. If public schemas expose it accidentally, remove it from public output.
11. Add tests: service save, router create, and public exclusion if a public test exists.

**Fake-Code Shape:**

```ts
// DB shape only. Use the actual schema helpers and migration process.
internalNote: text("internal_note"),

// Service input shape.
export const CreateStatusReportInput = z.object({
  title: z.string(),
  message: z.string(),
  internalNote: z.string().max(2000).nullish(),
  // existing fields stay here
});

// Public mapping rule:
const publicReport = {
  title: report.title,
  status: report.status,
  // no internalNote here
};
```

**Tips:**
- The word "internal" should make you suspicious of public output. Check both dashboard and public status page data paths.
- Public feed mapping currently forwards report objects into `StatusFeed` after adding `affected` and `updates` at `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:340-346`.
- Keep internal note out of emails unless product explicitly asks otherwise.

**What Could Go Wrong:**
- Adding the DB column but not validation types causes TypeScript drift.
- Passing internal note to public schemas can leak operator context.
- Making the field required breaks existing reports.

**Suggested Checks:**
- Create a report with an internal note.
- Edit the note.
- View public status page and confirm the note is absent.
- Run status-report service/router tests.

**Review Questions:**
- Did the field remain dashboard-only?
- Did old rows still parse?
- Is there a test that would fail if the note leaked publicly?

## Story 22: Add Maintenance Impact Level

**Difficulty:** Hard  
**Estimated Time:** 8-10 hours  
**Skills You'll Practice:** enum modeling, full-stack forms, public display, defaulting  
**The Story:** As a dashboard user, I want to mark a maintenance window as low, medium, or high impact so that public visitors understand how disruptive it may be.

**Acceptance Criteria:**
- [ ] Maintenance windows store an impact level.
- [ ] Create and edit maintenance forms expose impact level.
- [ ] Dashboard maintenance table displays impact.
- [ ] Public maintenance feed/banner displays impact.
- [ ] Existing maintenance windows default safely.

**Read These Anchors First:**
- `packages/api/src/router/maintenance.ts:68-98` - create maintenance input and service adapter.
- `packages/api/src/router/maintenance.ts:100-129` - update maintenance input and service adapter.
- `packages/db/src/schema/maintenances/maintenance.ts` - maintenance table schema.
- `packages/db/src/schema/maintenances/validation.ts` - maintenance validation/types.
- `packages/services/src/maintenance/schemas.ts` - service input schemas.
- `apps/dashboard/src/components/forms/maintenance/form.tsx` - dashboard maintenance form.
- `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:212-252` - public maintenance banner content.
- `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:347-359` - public maintenance feed mapping.

**Files You'll Likely Touch:**
- Maintenance DB schema and validation files.
- Maintenance service schemas/create/update files.
- `packages/api/src/router/maintenance.ts`
- `apps/dashboard/src/components/forms/maintenance/form.tsx`
- `apps/dashboard/src/components/data-table/maintenances/columns.tsx`
- `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx`
- Maintenance tests.

**Detailed Implementation Plan:**
1. Define allowed impact values: `low`, `medium`, `high`.
2. Add an enum-like text column with default `medium` or `low`. Pick the default with product intent: `medium` is safer for visibility; `low` is less alarming.
3. Update maintenance validation schemas and inferred types.
4. Update service create/update inputs to include optional `impact`.
5. Update create and update service functions to persist `impact`.
6. Update tRPC create/update inputs at `packages/api/src/router/maintenance.ts:70-79` and `packages/api/src/router/maintenance.ts:102-110`.
7. Add a select to the maintenance form.
8. Display impact in dashboard columns.
9. Display impact in public maintenance banner/feed using a restrained badge.
10. Add tests for default impact and explicit impact.

**Fake-Code Shape:**

```ts
export const maintenanceImpact = ["low", "medium", "high"] as const;

const maintenanceImpactSchema = z.enum(maintenanceImpact);

impact: text("impact", { enum: maintenanceImpact }).notNull().default("medium");
```

```tsx
<Select
  value={field.value}
  onValueChange={field.onChange}
>
  <SelectItem value="low">Low impact</SelectItem>
  <SelectItem value="medium">Medium impact</SelectItem>
  <SelectItem value="high">High impact</SelectItem>
</Select>
```

**Tips:**
- Do not reuse status report status values; impact is a different domain concept.
- Public copy should avoid panic. "High impact" is enough; no need for dramatic language.
- Update both create and update, or users can set impact once and never change it.

**What Could Go Wrong:**
- Forgetting default values breaks existing rows.
- Adding impact only to dashboard but not public output makes the feature invisible to visitors.
- Naming the field `status` creates confusion with operational status.

**Suggested Checks:**
- Create maintenance without selecting impact and confirm default.
- Edit impact.
- Public banner/feed shows impact.
- Existing maintenance tests still pass.

**Review Questions:**
- Is impact modeled separately from status?
- Are defaults safe for existing data?
- Did public and dashboard surfaces agree?

## Story 23: Add Component Groups To Subscriber Scope Display

**Difficulty:** Hard  
**Estimated Time:** 7-9 hours  
**Skills You'll Practice:** relation-aware UI, form grouping, public/dashboard consistency  
**The Story:** As a dashboard user managing subscribers, I want subscriber scope to show component groups so that I can understand whether a subscriber is tied to a whole group of services.

**Acceptance Criteria:**
- [ ] Subscriber table or details view shows selected component names grouped by page component group when available.
- [ ] Ungrouped components still display.
- [ ] Dashboard add/edit subscriber form remains compatible with existing component ids.
- [ ] Public manage-subscription view still saves component ids, not group ids.

**Read These Anchors First:**
- `apps/dashboard/src/app/(dashboard)/status-pages/[id]/subscribers/page.tsx:85-101` - page components and groups become checkbox tree items.
- `apps/dashboard/src/app/(dashboard)/status-pages/[id]/subscribers/page.tsx:120-142` - create subscriber payload sends component ids.
- `packages/api/src/router/pageSubscriber.ts:254-291` - protected create subscription schema.
- `packages/api/src/router/pageSubscriber.ts:296-317` - protected update channel/scope schema.
- `packages/api/src/router/pageSubscriber.ts:180-207` - public update-scope still accepts component ids.
- `apps/dashboard/src/components/data-table/subscribers/columns.tsx` - subscriber table display.

**Files You'll Likely Touch:**
- `apps/dashboard/src/components/data-table/subscribers/columns.tsx`
- `apps/dashboard/src/components/data-table/subscribers/data-table-row-actions.tsx`
- Possibly `apps/dashboard/src/app/(dashboard)/status-pages/[id]/subscribers/page.tsx`
- Possibly public manage subscription form if it displays scope.

**Detailed Implementation Plan:**
1. Inspect subscriber row shape from `RouterOutputs["pageSubscriber"]["list"][number]`.
2. Identify whether each subscriber row already includes `components` with group metadata.
3. If group metadata is missing, inspect `listPageSubscribers` service before adding API output.
4. Prefer enriching list output in the service rather than making the table refetch group data per row.
5. Add a helper function that groups component names by group name, with "Ungrouped" fallback.
6. Render grouped scope in a table cell or row details panel.
7. Keep form payloads as `componentIds`; do not introduce `groupIds` unless product explicitly asks.
8. Add a test for grouped and ungrouped scope if service output changes.

**Fake-Code Shape:**

```ts
function groupComponentsForDisplay(components: SubscriberComponent[]) {
  return components.reduce<Record<string, string[]>>((acc, component) => {
    const groupName = component.group?.name ?? "Ungrouped";
    acc[groupName] ??= [];
    acc[groupName].push(component.name);
    return acc;
  }, {});
}
```

**Tips:**
- `toCheckboxTreeItems` already combines components and groups for form selection at `apps/dashboard/src/app/(dashboard)/status-pages/[id]/subscribers/page.tsx:98-101`.
- Subscriber create/update routes accept component ids, not group ids (`packages/api/src/router/pageSubscriber.ts:263-277` and `packages/api/src/router/pageSubscriber.ts:299-305`).
- Read output shape before designing UI. You may not need backend changes.

**What Could Go Wrong:**
- Introducing group ids can create a second source of truth.
- Fetching components per subscriber row causes N+1 query behavior.
- Ungrouped components can disappear if the grouping helper assumes every component has a group.

**Suggested Checks:**
- Subscriber scoped to grouped components displays group name.
- Subscriber scoped to ungrouped components displays fallback.
- Saving scope still sends component ids.

**Review Questions:**
- Did you keep persistence based on component ids?
- Did you avoid per-row network calls?
- Does display handle missing group data?

## Story 24: Add Status Page "Operational Summary" API Field

**Difficulty:** Expert  
**Estimated Time:** 10-14 hours  
**Skills You'll Practice:** read-model design, public API contract, performance, type safety  
**The Story:** As a public status-page visitor, I want a concise operational summary so that I can quickly understand total components, degraded components, active reports, and scheduled maintenance.

**Acceptance Criteria:**
- [ ] Public `statusPage.get` output includes an `operationalSummary` object.
- [ ] Summary includes total component count, degraded/error component count, active report count, and active/scheduled maintenance count.
- [ ] Public client renders the summary without recomputing all counts from scratch.
- [ ] Existing public page behavior remains unchanged if summary is absent during rollout.
- [ ] Tests cover summary calculation.

**Read These Anchors First:**
- `packages/api/src/router/statusPage.ts:64-113` - public `statusPage.get` route starts.
- `packages/api/src/router/statusPage.utils.ts:128-258` - event derivation helper.
- `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:56-136` - public page query and selected page data.
- `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:274-327` - component/group rendering.
- `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:329-360` - feed rendering.
- `packages/api/src/router/statusPage.utils.test.ts` - utility test home.

**Files You'll Likely Touch:**
- `packages/api/src/router/statusPage.ts`
- `packages/api/src/router/statusPage.utils.ts`
- `packages/api/src/router/statusPage.utils.test.ts`
- `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx`
- Shared DB/public schemas if the route output is parsed through one.

**Detailed Implementation Plan:**
1. Define the summary shape in TypeScript near the public status page route or utility file.
2. Implement a pure helper, for example `getOperationalSummary(page)`, that accepts the already-loaded public page data.
3. Count components from `page.pageComponents` or the normalized public page relation.
4. Count degraded/error components by component status values.
5. Count active reports from the same open events/report status logic used for public display.
6. Count maintenance windows using the same event or maintenance filtering semantics used by the public page.
7. Attach `operationalSummary` to the `statusPage.get` response.
8. In the public client, render summary cards near the title or before components.
9. Add tests for empty page, all operational, degraded components, active report, and maintenance.
10. Consider performance: use existing loaded data, not extra DB queries.

**Fake-Code Shape:**

```ts
export type OperationalSummary = {
  totalComponents: number;
  affectedComponents: number;
  activeReports: number;
  activeMaintenances: number;
};

export function getOperationalSummary(page: PublicStatusPage): OperationalSummary {
  return {
    totalComponents: page.pageComponents.length,
    affectedComponents: page.pageComponents.filter((c) =>
      ["degraded", "error"].includes(c.status),
    ).length,
    activeReports: page.statusReports.filter((report) => report.status !== "resolved").length,
    activeMaintenances: page.maintenances.filter(isActiveMaintenance).length,
  };
}
```

**Tips:**
- Keep the helper pure so it is easy to test.
- The public client already computes some display arrays; this story moves summary logic closer to the API contract.
- If summary semantics disagree with `openEvents`, users will see contradictory page status. Use the existing event logic as your guide.

**What Could Go Wrong:**
- Counting all reports instead of active reports inflates incident count.
- Adding extra DB queries can slow public page loads.
- Returning a summary shape that is not covered by parsing schemas can fail at runtime.

**Suggested Checks:**
- Utility tests for summary calculation.
- Public page still renders when no components exist.
- Summary numbers agree with visible components/feed.

**Review Questions:**
- Is the summary a read model, not a new source of truth?
- Did you reuse existing loaded data?
- Are summary semantics aligned with public events?

## Story 25: Add A Status Page Health Digest

**Difficulty:** Expert  
**Estimated Time:** 14-18 hours  
**Skills You'll Practice:** product design, cross-domain read model, caching strategy, public rendering  
**The Story:** As a status-page visitor, I want a short health digest that summarizes current status, recent incidents, and upcoming maintenance so that I can understand the page at a glance.

**Acceptance Criteria:**
- [ ] Public page displays a concise digest near the top.
- [ ] Digest includes current overall status.
- [ ] Digest includes count or mention of recent status reports.
- [ ] Digest includes count or mention of upcoming maintenance.
- [ ] Digest is computed consistently with existing public status data.
- [ ] Implementation avoids extra network requests from the public client.

**Read These Anchors First:**
- `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:136-272` - top public page banner/status rendering.
- `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:274-327` - components render.
- `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:329-360` - feed render.
- `packages/api/src/router/statusPage.ts:64-113` - public status page route.
- `packages/api/src/router/statusPage.utils.ts:46-94` - status data filling helper.
- `packages/api/src/router/statusPage.utils.ts:128-258` - public event derivation helper.

**Files You'll Likely Touch:**
- `packages/api/src/router/statusPage.utils.ts`
- `packages/api/src/router/statusPage.ts`
- `packages/api/src/router/statusPage.utils.test.ts`
- `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx`
- Possibly a small new presentational component under `apps/status-page/src/components/status-page/`

**Detailed Implementation Plan:**
1. Write the desired digest sentences before coding. Example: "All systems operational. No recent incidents. One maintenance window scheduled."
2. Decide whether the digest should be computed server-side in the API or client-side from existing response data. Prefer server-side/pure utility if it becomes a public contract.
3. Create a pure helper such as `buildHealthDigest({ status, lastEvents, statusReports, maintenances })`.
4. Use existing status values; do not invent a second status model.
5. Use the same recent-event logic as the feed so the digest does not contradict visible history.
6. Attach digest to public page output or compute it in the public client from already-loaded `page`.
7. Render digest below `StatusHeader` and above the banner, or inside the banner area if design fits.
8. Add tests for: no events, active incident, active maintenance, multiple event types.
9. If caching exists for public page data, verify digest participates in the same cache path instead of adding a separate request.

**Fake-Code Shape:**

```ts
type HealthDigestInput = {
  status: "operational" | "degraded" | "error" | "info";
  recentReportCount: number;
  upcomingMaintenanceCount: number;
};

export function buildHealthDigest(input: HealthDigestInput) {
  const statusLine =
    input.status === "operational"
      ? "All systems operational."
      : "Some systems need attention.";

  const reportLine =
    input.recentReportCount === 0
      ? "No recent incidents."
      : `${input.recentReportCount} recent incident updates.`;

  const maintenanceLine =
    input.upcomingMaintenanceCount === 0
      ? "No upcoming maintenance."
      : `${input.upcomingMaintenanceCount} maintenance window scheduled.`;

  return [statusLine, reportLine, maintenanceLine].join(" ");
}
```

**Tips:**
- This is a product-facing feature. Ask whether digest text should be translated before hardcoding English.
- Public page status is rendered with `<Status variant={page.status}>` at `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:136-139`.
- The banner already represents active events at `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:143-272`; digest should summarize, not duplicate every detail.

**What Could Go Wrong:**
- Digest says "all systems operational" while the banner shows an active incident.
- Adding a separate query increases public page load time.
- Hardcoded English may clash with locale-aware public routes.

**Suggested Checks:**
- Public page with no events has calm digest.
- Public page with active report has attention digest.
- Public page with maintenance has maintenance digest.
- No additional network request appears in the browser devtools.

**Review Questions:**
- Does digest agree with banner/feed/components?
- Is the implementation a summary, not a new truth source?
- Did you consider localization before merging?

