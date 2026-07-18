# Reading Map

## Mental Model

OpenStatus works like a status-room control panel connected to a public lobby display. The dashboard lets workspace members create monitors, pages, status reports, and notifications through typed tRPC procedures (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:23-128`; `packages/api/src/edge.ts:25-49`). Service modules enforce workspace boundaries and write Drizzle rows (`packages/services/src/status-report/create.ts:24-99`; `packages/db/src/db.ts:6-12`). The public status-page app reads those rows and converts them into banners, component cards, and feed items for users watching service health (`packages/api/src/router/statusPage.ts:65-400`; `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:136-414`).

## Top Files To Read In Order

1. `package.json` (`package.json:3-20`; `package.json:33-38`)  
   Why now: learn package manager, scripts, and workspace layout. Understand before reading: this is a monorepo. Explain after reading: how to run dev, tests, lint, and app-specific dev commands.

2. `turbo.json` (`turbo.json:4-66`)  
   Why now: learn task dependencies. Understand before reading: Turbo runs package scripts. Explain after reading: why `dx` depends on `seed`, which depends on `migrate`, which depends on `env`.

3. `README.md` (`README.md:58-143`; `README.md:144-153`)  
   Why now: compare official setup with code reality. Understand before reading: the app can run via Docker or manually. Explain after reading: required tools and the named stack.

4. `apps/dashboard/src/lib/trpc/client.tsx` (`apps/dashboard/src/lib/trpc/client.tsx:11-64`)  
   Why now: see how React calls the typed API. Understand before reading: React Query owns caching. Explain after reading: why the browser query client is reused.

5. `apps/dashboard/src/lib/trpc/shared.ts` (`apps/dashboard/src/lib/trpc/shared.ts:17-76`)  
   Why now: see how calls split between edge and lambda. Understand before reading: tRPC links route requests. Explain after reading: which router prefixes go to lambda.

6. `packages/api/src/trpc.ts` (`packages/api/src/trpc.ts:59-119`; `packages/api/src/trpc.ts:148-264`)  
   Why now: this is the API gate. Understand before reading: context carries session, request, db, user, and workspace. Explain after reading: how `protectedProcedure` resolves workspace.

7. `packages/api/src/edge.ts` (`packages/api/src/edge.ts:25-49`)  
   Why now: see the router table. Understand before reading: each property becomes a typed API namespace. Explain after reading: why `statusReport` is called as `trpc.statusReport.*`.

8. `packages/api/src/router/statusReport.ts` (`packages/api/src/router/statusReport.ts:24-222`)  
   Why now: a compact backend controller. Understand before reading: tRPC procedures validate input and call services. Explain after reading: why create returns the initial update row.

9. `packages/services/src/status-report/create.ts` (`packages/services/src/status-report/create.ts:24-99`)  
   Why now: business logic lives here. Understand before reading: service context has workspace and actor. Explain after reading: what is inside the transaction.

10. `packages/db/src/schema/status_reports/status_reports.ts` (`packages/db/src/schema/status_reports/status_reports.ts:8-89`)  
    Why now: the data model explains the feature. Understand before reading: Drizzle table declarations become typed rows. Explain after reading: report vs report update.

11. `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx` (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:23-128`)  
    Why now: see the feature from the product owner's view. Understand before reading: client components can use hooks. Explain after reading: how create, refetch, and invalidation work together.

12. `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx` (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:56-414`)  
    Why now: close the loop. Understand before reading: public pages read from `statusPage.get`. Explain after reading: how open events become banners and feed entries.

## Three Most Important Data Flows

1. Status report creation: dashboard form submits values (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:89-103`), tRPC validates and calls the service (`packages/api/src/router/statusReport.ts:58-81`), the service inserts report/update rows and associations (`packages/services/src/status-report/create.ts:32-80`), then dashboard refetches and invalidates page list cache (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:36-51`).

2. Public status rendering: public client queries `statusPage.get` (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:65-79`), router loads page, reports, maintenances, components, incidents (`packages/api/src/router/statusPage.ts:80-113`), computes `openEvents`, `lastEvents`, `trackers`, and page status (`packages/api/src/router/statusPage.ts:136-400`), then UI renders banners, component cards, and feed entries (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:143-360`).

3. Authenticated API request: dashboard tRPC client sends `x-trpc-source` (`apps/dashboard/src/lib/trpc/client.tsx:41-55`), route handlers guard the source and build context with `auth` (`apps/status-page/src/app/api/trpc/edge/[trpc]/route.ts:11-24`), tRPC middleware checks session and resolves active workspace (`packages/api/src/trpc.ts:148-200`), then routers receive a context that service adapters convert into `ServiceContext` (`packages/api/src/service-adapter.ts:18-24`).

## Pre-Reading Checklist

1. Can you name the package manager and workspace globs? Check `package.json:33-38`.
2. Can you name the three Next app surfaces? Check `apps/dashboard/package.json:1-6`, `apps/status-page/package.json:1-6`, and `apps/web/package.json:1-6`.
3. Can you explain why Turbo has `dx`, `seed`, `migrate`, and `env` tasks? Check `turbo.json:51-64`.
4. Can you distinguish dashboard auth from status-page viewer auth? Check `apps/dashboard/src/lib/auth/index.ts:15-24` and `apps/status-page/src/lib/auth/index.ts:15-19`.
5. Can you say where tRPC context gets `db`? Check `packages/api/src/trpc.ts:59-64`.
6. Can you say how a workspace is resolved? Check `packages/api/src/trpc.ts:171-200`.
7. Can you name the table that stores report updates? Check `packages/db/src/schema/status_reports/status_reports.ts:41-63`.
8. Can you name the join table between reports and components? Check `packages/db/src/schema/page_components/page_components.ts:124-157`.
9. Can you say which endpoint receives `statusReport.create`? Check `apps/dashboard/src/lib/trpc/shared.ts:38-76` and `packages/api/src/edge.ts:25-49`.
10. Can you find an existing test for cross-workspace rejection? Check `packages/api/src/router/statusReport.test.ts:77-99`.

## Red Flags Checklist

1. Client-side mutation triggers a second side effect that should be transactional or server-owned (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`).
2. A header guard is mistaken for auth, despite the code comment saying it is not a security boundary (`apps/dashboard/src/lib/trpc/shared.ts:17-29`).
3. A public router performs large relation loading and in-memory computation without pagination or caching (`packages/api/src/router/statusPage.ts:80-400`).
4. A comparator has suspicious asymmetric return values for empty update arrays (`packages/api/src/router/statusPage.ts:364-379`).
5. Service write functions omit `requireScope` before transactions; status-report create shows the expected placement (`packages/services/src/status-report/create.ts:24-32`).
6. New dashboard writes rely on user scopes even though user actors currently skip `requireScope` checks (`packages/services/src/auth/require-scope.ts:11-30`).
7. New tRPC routes return raw unknown errors instead of mapping through `toTRPCError`; the status-report router shows expected handling (`packages/api/src/router/statusReport.ts:62-80`; `packages/api/src/service-adapter.ts:32-87`).
8. New forms duplicate DB enum strings instead of importing schema enum values; `FormStatusReport` imports `statusReportStatus` (`apps/dashboard/src/components/forms/status-report/form.tsx:17-21`; `apps/dashboard/src/components/forms/status-report/form.tsx:62-69`).
9. New list pages fetch unbounded datasets without explaining why; status-report list uses `10_000` and comments on the legacy UI contract (`packages/api/src/router/statusReport.ts:189-218`).
10. Public status-page changes forget embed mode and hidden sections, which the client handles through `useEmbed` and `componentsVisible` (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:56-98`).

