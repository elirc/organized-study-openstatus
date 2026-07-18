# System Maps

## Map 1: Monorepo Terrain

```text
openstatus/
  apps/
    dashboard      -> authenticated product control room
    status-page    -> public status displays and viewer auth
    web            -> marketing/docs/playground surface
    workflows      -> background/cron-style Hono app
    checker        -> Go checker service
  packages/
    api            -> tRPC context, routers, adapters
    services       -> domain business operations
    db             -> Drizzle client, schema, seed/migrate
    ui             -> shared components and hooks
    tinybird       -> analytics/time-series queries
    subscriptions  -> subscriber dispatch behavior
```

Evidence: the root workspace includes `apps/*`, `packages/**/*`, and `packages/config/*` (`package.json:33-38`; `pnpm-workspace.yaml:1-4`). The README names Next.js, Hono, Go, Turso, Drizzle, Tinybird, Tailwind, and shadcn/ui as the stack (`README.md:144-153`). The services package exports domain modules such as `./status-report`, `./page`, `./monitor`, and `./notification` (`packages/services/package.json:8-81`).

## Map 2: Product Surface Ownership

```text
Dashboard app
  owns workspace user workflows
  reads/writes through protected tRPC procedures
  example: status report create/list page

Status-page app
  owns public visitor experience
  reads public page data through public tRPC procedures
  example: banner, components, feed, subscription controls

API package
  owns transport contract
  decides protected vs public, edge vs lambda, input validation

Services package
  owns business rules
  checks workspace ownership, transactions, audit, side effects

DB package
  owns persistence shape
  defines tables, relations, row schemas
```

Code anchors:

- Dashboard status reports page uses `useTRPC`, `useQuery`, and `useMutation` (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:23-51`).
- Public status page uses `trpc.statusPage.get` and renders status content from the response (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:56-136`; `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:143-360`).
- API context and procedures are initialized in `packages/api/src/trpc.ts` (`packages/api/src/trpc.ts:59-119`; `packages/api/src/trpc.ts:132-264`).
- Service context and transactions are defined in `packages/services/src/context.ts` (`packages/services/src/context.ts:22-47`).
- DB connection uses Drizzle with schema from `packages/db/src/schema` (`packages/db/src/db.ts:1-12`; `packages/db/src/schema/index.ts:1-27`).

## Map 3: Runtime Split

```text
tRPC client
  |
  | procedure path starts with stripeRouter/emailRouter/apiKeyRouter/integrationRouter/blob?
  | yes
  v
/api/trpc/lambda

tRPC client
  |
  | all other router prefixes
  v
/api/trpc/edge
```

The dashboard tRPC link stores lambda prefixes in `lambdas` and sends every other procedure to the edge endpoint (`apps/dashboard/src/lib/trpc/shared.ts:38-76`). The API package then exposes `edgeRouter` with `statusReport`, `statusPage`, `monitor`, `page`, and other product routers (`packages/api/src/edge.ts:25-49`), while `lambdaRouter` exposes Stripe, email, API key, integration, and blob routers (`packages/api/src/lambda.ts:1-14`).

Illustrative fake code:

```ts
// Illustrative fake code: endpoint selection, not copied from production.
function chooseEndpoint(procedureName: string) {
  const firstSegment = procedureName.split(".")[0];
  if (["emailRouter", "stripeRouter"].includes(firstSegment)) {
    return "/api/trpc/lambda";
  }
  return "/api/trpc/edge";
}
```

## Map 4: Status Report Feature Map

```text
User presses "Create Status Report"
  apps/dashboard/.../status-reports/page.tsx:75-109
    |
    v
FormStatusReport validates fields
  apps/dashboard/src/components/forms/status-report/form.tsx:62-77
    |
    v
trpc.statusReport.create
  packages/api/src/router/statusReport.ts:58-81
    |
    v
createStatusReport service
  packages/services/src/status-report/create.ts:24-99
    |
    v
status_report + status_report_update + status_report_to_page_component
  packages/db/src/schema/status_reports/status_reports.ts:15-63
  packages/db/src/schema/page_components/page_components.ts:124-157
    |
    v
public statusPage.get computes openEvents/lastEvents/feed data
  packages/api/src/router/statusPage.ts:231-400
    |
    v
public client renders banner/feed
  apps/status-page/.../(public)/client.tsx:143-360
```

## Map 5: Auth And Workspace Gate

```text
Dashboard request
  |
  v
NextAuth session
  apps/dashboard/src/lib/auth/index.ts:15-24
  |
  v
workspace-slug cookie
  apps/dashboard/src/proxy.ts:45-65
  |
  v
protectedProcedure middleware
  packages/api/src/trpc.ts:148-200
  |
  v
ServiceContext actor + workspace
  packages/api/src/service-adapter.ts:18-24
  packages/services/src/context.ts:13-29
```

Important distinction: `guardTRPCSource` checks a convention header and explicitly says it is not a security boundary (`apps/dashboard/src/lib/trpc/shared.ts:17-29`). Real authenticated workspace behavior is in `protectedProcedure` and service ownership checks (`packages/api/src/trpc.ts:148-200`; `packages/services/src/status-report/create.ts:32-46`).

