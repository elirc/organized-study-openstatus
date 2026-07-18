# Journal

## First-Pass Mental Model

OpenStatus is a product monorepo: app surfaces live under `apps/`, reusable domain and platform code lives under `packages/`, and Turbo coordinates build, dev, test, seed, and migration tasks (`package.json:3-20`; `turbo.json:4-66`). The main product surfaces are Next apps for dashboard, public status pages, and marketing web, plus Go and Hono services for checking and workflows (`apps/dashboard/package.json:1-36`; `apps/status-page/package.json:1-28`; `apps/web/package.json:1-19`; `README.md:144-153`).

The core pattern is: React route or component calls tRPC through React Query, tRPC validates inputs and adapts request context, service functions enforce workspace ownership and write to Drizzle tables, then public status-page queries read those rows back into status banners and feeds (`apps/dashboard/src/lib/trpc/client.tsx:11-64`; `packages/api/src/trpc.ts:71-119`; `packages/api/src/service-adapter.ts:18-87`; `packages/services/src/status-report/create.ts:24-99`; `packages/api/src/router/statusPage.ts:65-400`).

## Important Discoveries

- The dashboard and status-page apps both expose `/api/trpc/edge` and `/api/trpc/lambda`, and the client chooses the endpoint based on router prefix (`apps/dashboard/src/lib/trpc/shared.ts:38-76`; `apps/status-page/src/app/api/trpc/edge/[trpc]/route.ts:11-24`; `apps/status-page/src/app/api/trpc/lambda/[trpc]/route.ts:12-25`).
- The API package splits routers into `edgeRouter` and `lambdaRouter`; status reports are in the edge router, while email and Stripe-like integrations go through lambda (`packages/api/src/edge.ts:25-49`; `packages/api/src/lambda.ts:7-14`).
- Auth is not one global object. The dashboard uses NextAuth with GitHub, Google, and dev/self-host Resend providers (`apps/dashboard/src/lib/auth/index.ts:15-24`; `apps/dashboard/src/lib/auth/providers.ts:5-27`). The public status page has its own viewer-oriented NextAuth flow using Resend and custom-domain email-domain validation (`apps/status-page/src/lib/auth/index.ts:15-65`; `apps/status-page/src/lib/auth/providers.ts:6-32`).
- Service functions are being carved out of tRPC routers. `statusReportRouter` explicitly says its tRPC schemas preserve a legacy wire contract while adapting to canonical service schemas (`packages/api/src/router/statusReport.ts:21-23`).
- The service layer has a real authorization abstraction for API keys and MCP actors, but dashboard `user` actors are currently a no-op for `requireScope`; member-role enforcement is called out as separate work (`packages/services/src/auth/require-scope.ts:7-24`; `packages/services/src/auth/require-scope.ts:25-48`).
- Status-report listing had an N+1 risk that was addressed by batched enrichment across updates, component associations, and page data (`packages/services/src/status-report/list.ts:66-161`).

## Why These Files Are Teaching Anchors

- `package.json` and `turbo.json` show the repo's operating system: scripts, task graph, and tools (`package.json:3-20`; `turbo.json:48-64`).
- `apps/dashboard/src/lib/trpc/client.tsx` and `apps/dashboard/src/lib/trpc/shared.ts` show how frontend state reaches backend procedures (`apps/dashboard/src/lib/trpc/client.tsx:39-64`; `apps/dashboard/src/lib/trpc/shared.ts:46-76`).
- `packages/api/src/trpc.ts` is the request-context and middleware gate (`packages/api/src/trpc.ts:59-94`; `packages/api/src/trpc.ts:148-264`).
- `packages/api/src/router/statusReport.ts` is a compact example of router-level validation, service adaptation, analytics metadata, and legacy return-shape preservation (`packages/api/src/router/statusReport.ts:24-81`; `packages/api/src/router/statusReport.ts:189-222`).
- `packages/services/src/status-report/create.ts` is the cleanest single example of workspace guard, transaction, relational insert, association update, and audit emission (`packages/services/src/status-report/create.ts:24-99`).
- `packages/db/src/schema/status_reports/status_reports.ts` anchors the domain model for incidents/status reports (`packages/db/src/schema/status_reports/status_reports.ts:8-89`).
- `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx` shows how database-backed status is turned into public user experience (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:56-414`).

## Where A Junior Might Get Confused

- Route folders such as `(dashboard)` and `[id]` are Next App Router conventions; they are not ordinary URL strings in code (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/layout.tsx:18-50`).
- `useTRPC()` does not call the server by itself. It returns typed procedure helpers that React Query then executes (`apps/dashboard/src/lib/trpc/client.tsx:11-12`; `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:25-37`).
- Some schemas live in DB package files, while service-specific schemas live in service files; the same domain has multiple boundary-specific contracts (`packages/db/src/schema/status_reports/validation.ts:10-55`; `packages/services/src/status-report/schemas.ts:12-92`; `packages/api/src/router/statusReport.ts:24-56`).

## Where A Mid-Level Engineer Should Slow Down

- Endpoint routing is convention-based: procedure prefixes listed in `lambdas` go to `/api/trpc/lambda`, and every other router goes to edge (`apps/dashboard/src/lib/trpc/shared.ts:38-76`).
- The dashboard mutation sends subscriber email from the UI after the create mutation succeeds, marked with a TODO to move to the server (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`). That is a reliability and ownership boundary worth questioning.
- The public status-page router loads a lot of relations in one `findFirst` and then performs substantial in-memory transformation (`packages/api/src/router/statusPage.ts:80-113`; `packages/api/src/router/statusPage.ts:136-400`).

## Where A Senior Engineer Should Be Skeptical

- `guardTRPCSource` explicitly says its `x-trpc-source` header is not a security boundary, so any security review must check `protectedProcedure` and service authorization instead (`apps/dashboard/src/lib/trpc/shared.ts:17-29`).
- Dashboard user actors bypass `requireScope`, leaving member-role enforcement as a known gap (`packages/services/src/auth/require-scope.ts:11-24`; `packages/services/src/auth/require-scope.ts:25-30`).
- The public status-page router has comments about SQL performance and moving Tinybird logic into the public path (`packages/api/src/router/statusPage.ts:49-55`).
- A sort comparator in status-page `statusReports` returns `-1` when `b.statusReportUpdates.length === 0`; a comparator usually needs opposite signs for symmetric empty cases, so this deserves a test (`packages/api/src/router/statusPage.ts:364-379`).

## How To Use Checkpoints

Checkpoint answers should include file paths and line ranges. A strong answer names the user-facing behavior, the boundary crossed, the contract used, and the failure mode. For example, a strong status-report answer mentions the dashboard form submit path (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:89-103`), the tRPC create procedure (`packages/api/src/router/statusReport.ts:58-81`), the service transaction (`packages/services/src/status-report/create.ts:32-98`), and the DB tables (`packages/db/src/schema/status_reports/status_reports.ts:15-63`).

