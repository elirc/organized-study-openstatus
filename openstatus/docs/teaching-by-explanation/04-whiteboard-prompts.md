# Whiteboard Prompts

Use these when preparing to explain the system in meetings, interviews, or planning sessions.

## Prompt 1: Draw The Status Report Create Flow

Whiteboard target:

```text
Dashboard page
  -> FormStatusReport
  -> trpc.statusReport.create
  -> protectedProcedure
  -> createStatusReport service
  -> status_report/status_report_update/join table
  -> React Query refetch/invalidate
  -> public statusPage.get
  -> StatusFeed and StatusBanner
```

Must cite while explaining:

- UI submit (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:89-103`)
- tRPC create (`packages/api/src/router/statusReport.ts:58-81`)
- service transaction (`packages/services/src/status-report/create.ts:32-98`)
- DB schema (`packages/db/src/schema/status_reports/status_reports.ts:15-63`)
- public feed mapping (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:329-360`)

Self-grade:

- Strong: includes both write path and public read path.
- Medium: includes UI/API/DB but misses public rendering.
- Weak: says "the frontend calls the backend" without naming contracts.

## Prompt 2: Explain Edge vs Lambda

Whiteboard target:

```text
endingLink
  lambda prefixes:
    stripeRouter
    emailRouter
    apiKeyRouter
    integrationRouter
    blob
  else:
    edge
```

Must cite:

- Prefix selection (`apps/dashboard/src/lib/trpc/shared.ts:38-76`)
- Edge router table (`packages/api/src/edge.ts:25-49`)
- Lambda router table (`packages/api/src/lambda.ts:1-14`)
- Lambda route comment about Stripe and Edge incompatibility (`apps/status-page/src/app/api/trpc/lambda/[trpc]/route.ts:9-21`)

Self-grade:

- Strong: explains runtime constraints and procedure-prefix routing.
- Medium: lists routers but cannot explain why split exists.
- Weak: thinks edge/lambda are different apps.

## Prompt 3: Explain Workspace Safety

Whiteboard target:

```text
session.user.id
  + workspace-slug cookie
  -> resolveActiveWorkspace
  -> ctx.user + ctx.workspace
  -> toServiceCtx
  -> service resource ownership checks
```

Must cite:

- Cookie setting in proxy (`apps/dashboard/src/proxy.ts:45-65`)
- Middleware resolution (`packages/api/src/trpc.ts:171-200`)
- Service context adapter (`packages/api/src/service-adapter.ts:18-24`)
- Page ownership check (`packages/services/src/status-report/create.ts:32-40`)

Self-grade:

- Strong: separates authentication, active workspace selection, and resource authorization.
- Medium: explains session and workspace but misses service ownership check.
- Weak: relies on route params or headers as security.

## Prompt 4: Explain The Data Model

Whiteboard target:

```text
page
  has many page_component
  has many status_report

status_report
  has many status_report_update
  joins many page_component through status_report_to_page_component
```

Must cite:

- Page table/relations (`packages/db/src/schema/pages/page.ts:13-96`)
- Page component table (`packages/db/src/schema/page_components/page_components.ts:20-63`)
- Status report/update tables (`packages/db/src/schema/status_reports/status_reports.ts:15-89`)
- Join table (`packages/db/src/schema/page_components/page_components.ts:124-157`)

Self-grade:

- Strong: distinguishes current report state from timeline updates.
- Medium: names tables but misses join table.
- Weak: thinks selected affected components are stored as an array column on the report.

## Prompt 5: Explain One Senior-Level Concern

Choose one:

1. Notification reliability: dashboard owns subscriber notification after create success (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`).
2. Public query performance: `statusPage.get` loads many relations and has a performance TODO (`packages/api/src/router/statusPage.ts:49-113`).
3. Authorization gap: `requireScope` skips dashboard user actors and notes member-role enforcement is separate (`packages/services/src/auth/require-scope.ts:11-30`).
4. Sorting risk: status-page report sort has suspicious empty-update comparator behavior (`packages/api/src/router/statusPage.ts:364-379`).

Whiteboard format:

```text
Concern:
Why it matters:
Current code evidence:
Possible failure:
Test I would write:
First refactor I would consider:
```

Example answer:

```text
Concern: notification reliability.
Why it matters: incident subscribers expect timely updates.
Current code evidence: notification happens in dashboard mutation onSuccess and has TODO move to server.
Possible failure: report is created, browser closes, subscriber notification never sends.
Test I would write: create report with notify true and assert server-side dispatch is called after successful create.
First refactor: move notify intent into a server-owned post-commit workflow.
```

Evidence for example: dashboard notification side effect (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`) and service test subscription spies (`packages/services/src/status-report/__tests__/status-report.test.ts:41-56`).

