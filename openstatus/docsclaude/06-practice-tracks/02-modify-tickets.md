# 02 — Modify Tickets

10 real change requests, each scoped to be doable in 1–4 hours. They progress from "single file, UI-only" to "full-stack with tests." For each ticket: read the anchors, write a plan, then implement.

## Ticket 1 (Easy) — Better empty state on the status reports page

**Acceptance:** when a status page has no reports, the page shows useful guidance ("Create your first status report to communicate incidents to subscribers"), not a blank table.

**Files:**
- [`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx`](../../apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx)
- [`apps/dashboard/src/components/content/empty-state.tsx`](../../apps/dashboard/src/components/content/empty-state.tsx) (or grep for `EmptyStateContainer`)

**Plan:** Replace the empty-array branch (which currently still renders an empty `DataTable`) with an `EmptyStateContainer` and a button that opens the create sheet.

**Check:** verify with `pnpm dev:dashboard`. Verify the existing flow (reports → table) still works.

## Ticket 2 (Easy) — Update count column

**Acceptance:** the status report data table shows "Updates: N" as a column.

**Files:**
- [`apps/dashboard/src/components/data-table/status-reports/columns.tsx`](../../apps/dashboard/src/components/data-table/status-reports/columns.tsx)
- (Read-only) [`packages/services/src/status-report/list.ts`](../../packages/services/src/status-report/list.ts) — verify `updates` is already in the row.

**Plan:** Add a column definition reading `row.original.updates.length`. Singular/plural copy.

**Check:** reports with 0, 1, 2+ updates all render correctly.

## Ticket 3 (Easy) — URL state for active tab

**Acceptance:** if a status page tab is selected, the URL reflects it; refresh preserves the selection.

**Files:**
- The tabs component used in the status pages layout (grep `Tabs` under `apps/dashboard`).

**Plan:** Use `useQueryStates` from `nuqs` instead of local `useState` for the active tab. Confirm reload preserves selection.

**Check:** click between tabs; reload; back/forward navigation works.

## Ticket 4 (Medium) — Filter status reports by status

**Acceptance:** the dashboard exposes a filter for one or more statuses; the list updates accordingly. URL-stateful.

**Files:**
- [`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx`](../../apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx)
- [`packages/api/src/router/statusReport.ts`](../../packages/api/src/router/statusReport.ts) — list procedure already accepts `period`; add `statuses`.
- (Read-only) [`packages/services/src/status-report/list.ts:172-181`](../../packages/services/src/status-report/list.ts#L172-L181) — service already handles `statuses`.
- (Read-only) [`packages/services/src/status-report/schemas.ts`](../../packages/services/src/status-report/schemas.ts) — service schema already includes `statuses`.

**Plan:** Extend the router input to accept `statuses?: StatusReportStatus[]`. Pass through to the service. Add a select/checkbox UI. Encode in URL via `nuqs`.

**Tests to add:** router-level test that statuses filter narrows the result.

**Check:** filter to "resolved" — list shows only resolved. Clear filter — all shown.

## Ticket 5 (Medium) — Sortable monitor list

**Acceptance:** clicking the "Last response" column header sorts ascending; clicking again toggles to descending; clicking a third time clears.

**Files:**
- [`apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx`](../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx) — already uses `sorting`/`setSorting` from `@tanstack/react-table`.
- The monitor columns file.

**Plan:** Add `enableSorting: true` to the column. Provide an accessor for the sortable value.

**Check:** sort works; URL state survives refresh.

## Ticket 6 (Medium) — Severity field on status reports

**Acceptance:** users can assign a severity (low/medium/high/critical) to a report; severity is persisted; the dashboard list shows it; the public status page feed displays it.

**Files (in order):**
- [`packages/db/src/schema/status_reports/status_reports.ts`](../../packages/db/src/schema/status_reports/status_reports.ts) — add `severity` column with default.
- [`packages/db/src/schema/status_reports/validation.ts`](../../packages/db/src/schema/status_reports/validation.ts) — regenerate / extend.
- [`packages/services/src/status-report/schemas.ts`](../../packages/services/src/status-report/schemas.ts) — add `severity`.
- [`packages/services/src/status-report/create.ts`](../../packages/services/src/status-report/create.ts) — insert.
- [`packages/api/src/router/statusReport.ts`](../../packages/api/src/router/statusReport.ts) — accept in wire input.
- [`apps/dashboard/src/components/forms/status-report/form.tsx`](../../apps/dashboard/src/components/forms/status-report/form.tsx) — add a select.
- [`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx`](../../apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx) — pass through.
- Public side: update [`packages/api/src/router/statusPage.ts`](../../packages/api/src/router/statusPage.ts) output and [`apps/status-page/.../client.tsx`](../../apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx).
- Migration: `pnpm --filter @openstatus/db generate`, review SQL, `pnpm --filter @openstatus/db migrate`.

**Plan:** Schema → service → router → form → page. Tests for the service (severity persisted) and router (rejects invalid severity).

**Check:** Create a report with each severity. Verify in DB, in dashboard list, in public page.

## Ticket 7 (Medium) — Notification on report status change

**Acceptance:** when a status report's status changes to `resolved`, subscribers are notified — currently they're only notified on creation.

**Files:**
- [`packages/services/src/status-report/resolve.ts`](../../packages/services/src/status-report/resolve.ts) (if exists; else update verb)
- [`packages/services/src/status-report/notify.ts`](../../packages/services/src/status-report/notify.ts)

**Plan:** Call `notifyStatusReport` from inside the resolve verb after the update. Decide: in-band or outbox?

**Check:** create a report → resolve it → email received in dev (or mocked).

## Ticket 8 (Hard) — Pagination for the dashboard report list

**Acceptance:** the report list paginates 20/page. URL stateful. Pagination controls present.

**Files:**
- [`packages/api/src/router/statusReport.ts:189-218`](../../packages/api/src/router/statusReport.ts#L189-L218) — list procedure currently uses `limit: 10_000` sentinel.
- [`packages/services/src/status-report/list.ts:184-202`](../../packages/services/src/status-report/list.ts#L184-L202) — service already supports `limit`/`offset` and returns `totalSize`.
- [`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx`](../../apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx).
- A pagination component (grep for `DataTablePagination`).

**Plan:** Pass `limit` + `offset` from UI to router to service. Render pagination using `totalSize`. URL state via `nuqs`. Remove the 10,000 sentinel — guard against missing inputs by defaulting in the router.

**Check:** list of 30 reports paginates; navigation preserves filters; deep-link works.

## Ticket 9 (Hard) — Outbox for subscriber notifications

**Acceptance:** notifications are written to an outbox table inside the create transaction; a background worker drains the outbox.

**Files (new):**
- New DB table `notification_outbox` with `id`, `kind`, `payload`, `status`, `attempts`, `last_error`, `created_at`, `dispatched_at`.
- New service: `packages/services/src/notification-outbox/{enqueue,drain}.ts`.
- Modify [`packages/services/src/status-report/create.ts`](../../packages/services/src/status-report/create.ts) and `add-update.ts` to enqueue inside the tx.
- Remove the dashboard `onSuccess` email mutation.
- New cron or workflow handler that drains pending outbox rows.

**Plan:** Design first — write a 1-page RFC. Then build incrementally: table + enqueue + draining job + remove client-side path.

**Check:** create a report → outbox row exists → worker dispatches → row marked dispatched. Failures retry.

## Ticket 10 (Expert) — Workspace member roles

**Acceptance:** workspace members have roles (`owner`, `admin`, `member`). The `requireScope` helper enforces role-based authorization for `user` actors — currently a no-op.

**Files:**
- [`packages/services/src/auth/require-scope.ts`](../../packages/services/src/auth/require-scope.ts) — implement the user branch.
- Schema for member roles.
- Tests across the entire codebase: every write verb gains a "rejects when actor is member-without-write-role" test.
- UI: role badges, role-change UI, "you don't have permission" toasts.

**Plan:** This is a project, not a ticket. Scope it: which verbs require admin vs member? Where's the role stored? Migration. Test impact (every service test needs an authorized ctx).

**Check:** a "member" can read, create reports, but cannot delete monitors. An "admin" can. Verify all writes have role tests.

## General workflow per ticket

1. Read every linked anchor before writing code.
2. Write a plan in your scratch file: what files, what order, what tests.
3. Implement.
4. Run `pnpm test` for affected packages.
5. Run `pnpm lint` (or `biome check`).
6. Use the feature in the browser.
7. Write a 3-bullet PR description: what changed, why it's safe, what you tested.

## Self-prompt for AI help

```
I'm working on Ticket N from docs/06-practice-tracks/02-modify-tickets.md.
The behavior I need is: [criterion].
The files I opened are: [paths].
The step I'm stuck on is: [step].
Don't give me the solution. Ask me 3-5 questions that help me discover the next move.
```
