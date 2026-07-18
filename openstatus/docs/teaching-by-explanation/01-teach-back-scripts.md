# Teach-Back Scripts

Use these scripts out loud. Then close the file and repeat them from memory with file references.

## Script 1: Thirty-Second System Overview

"OpenStatus is a pnpm/Turbo monorepo for status pages and uptime monitoring. The root scripts define dev, build, lint, test, and app-specific dev commands (`package.json:3-20`), and the workspace includes apps and packages (`package.json:33-38`; `pnpm-workspace.yaml:1-4`). The dashboard and public status-page are Next apps (`apps/dashboard/package.json:1-6`; `apps/status-page/package.json:1-6`). The API package owns tRPC routers and context (`packages/api/src/trpc.ts:59-119`; `packages/api/src/edge.ts:25-49`), services own business logic (`packages/services/package.json:8-81`), and DB owns Drizzle schema (`packages/db/src/db.ts:1-12`; `packages/db/src/schema/index.ts:1-27`)."

Practice prompt: say it again without package names, using product language only.

## Script 2: Status Report Create Flow

"When a dashboard user creates a status report, the page gets the status page id from the route and calls `trpc.statusReport.create` with title, status, page id, selected page components, date, message, and notify flag (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:23-32`; `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:89-103`). The tRPC router validates the dashboard wire shape, renames `pageComponents` to `pageComponentIds`, converts tRPC context into service context, and calls `createStatusReport` (`packages/api/src/router/statusReport.ts:24-32`; `packages/api/src/router/statusReport.ts:58-81`). The service checks write scope, confirms the page belongs to the current workspace, validates component ids, writes report/update/association rows, emits audit rows, and returns the new report plus initial update (`packages/services/src/status-report/create.ts:24-99`)."

Practice prompt: explain why this is not "just a form submit."

## Script 3: Public Status Page Rendering

"The public status-page client queries `statusPage.get` by domain slug (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:56-79`). The router loads page relations including reports, updates, maintenances, page components, monitors, incidents, and groups (`packages/api/src/router/statusPage.ts:80-113`). It computes component status, page status, open events, last events, grouped trackers, sorted reports, and public output (`packages/api/src/router/statusPage.ts:136-400`). The client turns that response into banner tabs, component cards, and feed rows (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:143-360`)."

Practice prompt: describe how an incident bulletin becomes a public banner.

## Script 4: Auth And Workspace Safety

"Dashboard calls are protected by `protectedProcedure`, not by the `x-trpc-source` header. The header guard is explicitly documented as spoofable (`apps/dashboard/src/lib/trpc/shared.ts:17-29`). Real protection starts when `enforceUserIsAuthed` checks session user id, reads the `workspace-slug` cookie, resolves the active workspace, and passes user/workspace into context (`packages/api/src/trpc.ts:148-200`). The service then still checks resource ownership, such as matching `page.id` and `page.workspaceId` before creating a report (`packages/services/src/status-report/create.ts:32-40`)."

Practice prompt: answer "Where is tenant isolation enforced?"

## Script 5: Reliability Risk Explanation

"The create-report path currently has one important reliability wrinkle: notification is triggered from the dashboard after the create mutation succeeds. The code calls `sendStatusReportUpdateMutation` in `onSuccess` when `notifySubscribers` is true, and the code comments say to move it server-side (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`). That means the database write can succeed while notification fails separately. By contrast, audit rows are service-owned inside the transaction (`packages/services/src/status-report/create.ts:82-95`)."

Practice prompt: explain the tradeoff without using the word "bug."

