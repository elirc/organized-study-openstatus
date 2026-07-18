# Trace Tables

Trace tables train you to follow values without losing the thread.

## Trace 1: `pageComponents` Becomes `pageComponentIds`

| Step | Name | Shape | Location |
| --- | --- | --- | --- |
| Form schema | `pageComponents` | `number[]` | `apps/dashboard/src/components/forms/status-report/form.tsx:62-69` |
| Page submit | `pageComponents: values.pageComponents` | `number[]` | `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:93-101` |
| tRPC schema | `pageComponents` | `z.array(z.number())` | `packages/api/src/router/statusReport.ts:24-32` |
| Router adapter | `pageComponentIds: input.pageComponents` | renamed | `packages/api/src/router/statusReport.ts:64-74` |
| Service schema | `pageComponentIds` | `z.array(z.number().int()).default([])` | `packages/services/src/status-report/schemas.ts:12-20` |
| Service validation | `validatePageComponentIds` | workspace/page-safe ids | `packages/services/src/status-report/create.ts:42-52` |
| Join table | `pageComponentId` | persisted relation | `packages/db/src/schema/page_components/page_components.ts:124-157` |

Trace exercise: explain why the rename is healthy. Strong answer: the dashboard can keep an older UI-friendly name while the service uses a domain name that is transport-independent (`packages/api/src/router/statusReport.ts:21-23`; `packages/services/src/status-report/schemas.ts:12-20`).

## Trace 2: Date Value

| Step | Name | Meaning | Location |
| --- | --- | --- | --- |
| Default form value | `date: new Date()` | local browser date object | `apps/dashboard/src/components/forms/status-report/form.tsx:96-105` |
| UI picker | `field.onChange(newDate)` | selected date/time | `apps/dashboard/src/components/forms/status-report/form.tsx:236-249`; `apps/dashboard/src/components/forms/status-report/form.tsx:269-290` |
| tRPC input | `date: z.coerce.date()` | runtime coercion | `packages/api/src/router/statusReport.ts:24-32` |
| Service input | `date: z.coerce.date()` | canonical coercion | `packages/services/src/status-report/schemas.ts:12-20` |
| DB update row | `date` timestamp | persisted incident timeline time | `packages/db/src/schema/status_reports/status_reports.ts:41-59` |
| Public timeline | `statusReportUpdates` ordered by date | visitor sees newest updates | `packages/api/src/router/statusPage.ts:84-91`; `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:171-174` |

Illustrative fake code:

```ts
// Illustrative fake code: how a date crosses boundaries.
const browserValue = new Date("2026-05-17T10:00:00-07:00");
const apiValue = z.coerce.date().parse(browserValue);
await db.insert(statusReportUpdate).values({ date: apiValue });
```

## Trace 3: Workspace Identity

| Step | Value | Owner | Location |
| --- | --- | --- | --- |
| Session user id | `ctx.session.user.id` | NextAuth/tRPC context | `packages/api/src/trpc.ts:71-94`; `packages/api/src/trpc.ts:148-152` |
| Workspace slug | `workspace-slug` cookie | dashboard proxy/client convention | `apps/dashboard/src/proxy.ts:45-65`; `packages/api/src/trpc.ts:171-177` |
| Resolved workspace | `{ user, workspace }` | API auth resolver | `packages/api/src/trpc.ts:178-200` |
| Service context | `actor`, `workspace` | API adapter | `packages/api/src/service-adapter.ts:18-24`; `packages/services/src/context.ts:13-29` |
| DB ownership check | `page.workspaceId === ctx.workspace.id` | service function | `packages/services/src/status-report/create.ts:32-40` |

Trace exercise: say which line prevents a user from creating a report on another workspace's page. Strong answer cites the service page lookup using both `page.id` and `page.workspaceId` (`packages/services/src/status-report/create.ts:32-40`) plus tests for cross-workspace rejection (`packages/api/src/router/statusReport.test.ts:77-99`).

## Trace 4: Public Status From Raw Events

| Step | Data | Computed Result | Location |
| --- | --- | --- | --- |
| Page relations | reports, maintenances, components, incidents | raw inputs | `packages/api/src/router/statusPage.ts:80-113` |
| Component events | `getEvents(...)` | events per component | `packages/api/src/router/statusPage.ts:136-145` |
| Component status | status enum | success/degraded/error/info | `packages/api/src/router/statusPage.ts:147-180` |
| Page events | all reports/maintenances/incidents | page-wide events | `packages/api/src/router/statusPage.ts:231-237` |
| Open events | active incidents/reports/maintenances | banner inputs | `packages/api/src/router/statusPage.ts:249-266` |
| Last events | recent feed candidates | feed inputs | `packages/api/src/router/statusPage.ts:239-247` |
| Public UI | status/feed props | rendered status page | `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:143-360` |

## Trace 5: Error Translation

| Step | Error Source | Translation | Location |
| --- | --- | --- | --- |
| Service throws `NotFoundError` | missing page/report | `TRPCError` with `NOT_FOUND` | `packages/api/src/service-adapter.ts:41-45` |
| Service throws `ForbiddenError` | scope mismatch | `TRPCError` with `FORBIDDEN` | `packages/api/src/service-adapter.ts:45-48` |
| Zod throws | invalid input | `TRPCError` with `BAD_REQUEST` | `packages/api/src/service-adapter.ts:34-40` |
| Unknown error | internal bug | generic internal message | `packages/api/src/service-adapter.ts:79-87` |

Code reading prompt: when reviewing a new router mutation, search for `try/catch` and `toTRPCError`. The status-report router uses that pattern throughout create, update, delete, get, and list procedures (`packages/api/src/router/statusReport.ts:62-80`; `packages/api/src/router/statusReport.ts:106-142`; `packages/api/src/router/statusReport.ts:144-222`).

