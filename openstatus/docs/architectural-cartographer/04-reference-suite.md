# Reference Suite

## Doc 1: Junior Onboarding Guide

Start with the root scripts. `pnpm install`, `turso dev --db-file openstatus-dev.db`, `pnpm dx`, and one of the app dev scripts are the manual setup path described by the repo (`README.md:93-143`; `package.json:15-18`). The DB package has `env`, `migrate`, `seed`, and `dev` scripts (`packages/db/package.json:6-14`), and Turbo wires `dx -> seed -> migrate -> env` (`turbo.json:51-64`).

Read in this order: root scripts (`package.json:3-20`), Turbo task graph (`turbo.json:4-66`), dashboard tRPC client (`apps/dashboard/src/lib/trpc/client.tsx:39-64`), tRPC context (`packages/api/src/trpc.ts:59-119`), router table (`packages/api/src/edge.ts:25-49`), status-report router (`packages/api/src/router/statusReport.ts:58-222`), service create (`packages/services/src/status-report/create.ts:24-99`), DB schema (`packages/db/src/schema/status_reports/status_reports.ts:15-89`), public status page (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:56-414`).

Day-one target: explain how a status report moves from a dashboard button to a public feed. Strong explanation cites the form submit (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:89-103`), API create procedure (`packages/api/src/router/statusReport.ts:58-81`), service transaction (`packages/services/src/status-report/create.ts:32-98`), and public feed mapping (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:329-360`).

## Doc 2: Mid-Level Architecture Guide

The architecture is a typed path from UI to service:

```text
React route -> React Query/tRPC -> Next route handler -> tRPC router -> service -> Drizzle/libSQL -> public query -> public React UI
```

The dashboard creates typed clients from `AppRouter` (`apps/dashboard/src/lib/trpc/client.tsx:9-12`). `endingLink` sends selected prefixes to lambda and everything else to edge (`apps/dashboard/src/lib/trpc/shared.ts:38-76`). Next route handlers call `fetchRequestHandler` with `edgeRouter` or `lambdaRouter` (`apps/status-page/src/app/api/trpc/edge/[trpc]/route.ts:11-24`; `apps/status-page/src/app/api/trpc/lambda/[trpc]/route.ts:12-25`). `createTRPCContext` provides session, request metadata, and DB (`packages/api/src/trpc.ts:59-94`). Protected procedures resolve the active workspace (`packages/api/src/trpc.ts:148-200`). Services accept `ServiceContext` and can run inside caller-provided transactions (`packages/services/src/context.ts:22-47`).

Main contract rule: do not confuse UI shape, tRPC shape, service shape, and DB shape. Status-report create uses `pageComponents` at tRPC and `pageComponentIds` in the service (`packages/api/src/router/statusReport.ts:24-32`; `packages/api/src/router/statusReport.ts:64-74`; `packages/services/src/status-report/schemas.ts:12-20`).

## Doc 3: Senior Ownership Guide

Own these risks first:

- Authorization: dashboard user actors skip `requireScope`; member-role enforcement is not implemented there (`packages/services/src/auth/require-scope.ts:11-30`).
- Side effects: report creation and notification are split across service and client `onSuccess` (`packages/services/src/status-report/create.ts:24-99`; `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`).
- Public performance: `statusPage.get` loads broad relation graphs and performs in-memory transformations (`packages/api/src/router/statusPage.ts:80-400`).
- UI loading/error states: status-report page returns null while data is absent (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:54-55`).
- Runtime split: new routers need deliberate edge vs lambda placement (`packages/api/src/edge.ts:25-49`; `packages/api/src/lambda.ts:7-14`; `apps/dashboard/src/lib/trpc/shared.ts:38-76`).

## Doc 4: Code Review Guide

Review in this order:

1. Boundary: Is the route public or protected? Compare with `protectedProcedure` (`packages/api/src/trpc.ts:255-264`) and `publicProcedure` (`packages/api/src/trpc.ts:135-142`).
2. Input contract: Is Zod validating the runtime shape? Use status-report create as reference (`packages/api/src/router/statusReport.ts:24-32`).
3. Context: Does the service receive `toServiceCtx(ctx)`? (`packages/api/src/service-adapter.ts:18-24`).
4. Ownership: Does the service prove rows belong to `ctx.workspace.id` before writing? (`packages/services/src/status-report/create.ts:32-46`).
5. Transaction: Are related writes grouped? (`packages/services/src/status-report/create.ts:32-98`).
6. Associations: Are join tables updated through domain helpers? (`packages/services/src/status-report/create.ts:65-69`; `packages/db/src/schema/page_components/page_components.ts:124-157`).
7. Cache: Does UI update the right query keys? (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:46-49`).
8. Tests: Is there a service or router test for cross-workspace access? (`packages/api/src/router/statusReport.test.ts:77-207`; `packages/services/src/status-report/__tests__/status-report.test.ts:116-208`).

## Doc 5: Debugging Guide

For dashboard create failures:

1. Confirm form values pass the `"date" in values` branch (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:89-103`).
2. Confirm the mutation is `trpc.statusReport.create` (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:36-51`).
3. Confirm the request is going to edge, not lambda (`apps/dashboard/src/lib/trpc/shared.ts:38-76`).
4. Confirm `protectedProcedure` has a session and workspace slug (`packages/api/src/trpc.ts:148-200`).
5. Confirm service page lookup matches workspace id (`packages/services/src/status-report/create.ts:32-40`).
6. Confirm component ids belong to the same page/workspace (`packages/services/src/status-report/create.ts:42-52`).
7. Confirm DB rows exist in `status_report`, `status_report_update`, and join table (`packages/db/src/schema/status_reports/status_reports.ts:15-63`; `packages/db/src/schema/page_components/page_components.ts:124-157`).

For public display failures:

1. Confirm `statusPage.get` can find the page by lowercased slug or custom domain (`packages/api/src/router/statusPage.ts:80-82`).
2. Confirm updates are loaded and ordered (`packages/api/src/router/statusPage.ts:84-91`).
3. Confirm events are computed from reports/maintenances/incidents (`packages/api/src/router/statusPage.ts:231-266`).
4. Confirm the client maps reports to feed props (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:329-360`).

## Doc 6: Change Playbook

To add or modify a status-report feature:

1. Update DB schema only if persistence changes; status reports live in `packages/db/src/schema/status_reports/status_reports.ts:15-89`.
2. Add or update DB/Zod row validation if row shape changes (`packages/db/src/schema/status_reports/validation.ts:38-55`).
3. Update canonical service schema (`packages/services/src/status-report/schemas.ts:12-92`).
4. Implement business behavior in service files and enforce workspace scope early (`packages/services/src/status-report/create.ts:24-99`; `packages/services/src/status-report/add-update.ts:17-68`).
5. Adapt tRPC router input/output without breaking existing dashboard shape (`packages/api/src/router/statusReport.ts:21-81`).
6. Update dashboard form or page behavior (`apps/dashboard/src/components/forms/status-report/form.tsx:79-417`; `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:23-128`).
7. Update public status-page output mapping if user-visible status changes (`packages/api/src/router/statusPage.ts:65-400`; `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:136-414`).
8. Add service and/or router tests for workspace isolation and expected row changes (`packages/services/src/status-report/__tests__/status-report.test.ts:116-208`; `packages/api/src/router/statusReport.test.ts:77-207`).

## Doc 7: Interview Walkthrough

Talk track:

"OpenStatus is a pnpm/Turbo monorepo for status pages and uptime monitoring. The root scripts define dev, build, lint, test, and app-specific dev commands (`package.json:3-20`), and the README names Next.js, Hono, Go, Turso, Drizzle, Tinybird, Tailwind, and shadcn/ui as the stack (`README.md:144-153`)."

"The dashboard uses typed tRPC clients generated from `AppRouter` (`packages/api/src/root.ts:1-6`; `apps/dashboard/src/lib/trpc/client.tsx:9-12`). Calls go to edge or lambda depending on procedure prefix (`apps/dashboard/src/lib/trpc/shared.ts:38-76`). Protected procedures resolve session and workspace before executing (`packages/api/src/trpc.ts:148-200`)."

"For a concrete feature, creating a status report starts in the dashboard status-reports page (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:89-103`), hits `statusReport.create` (`packages/api/src/router/statusReport.ts:58-81`), runs a service transaction that creates report/update rows and associations (`packages/services/src/status-report/create.ts:32-98`), and later appears on the public status page through `statusPage.get` and the public client feed (`packages/api/src/router/statusPage.ts:65-400`; `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:329-360`)."

