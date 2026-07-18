# Risk Heatmaps

## Heatmap Legend

```text
Low       -> stable or locally contained
Medium    -> common source of bugs, but tests/patterns help
High      -> boundary, side effect, performance, or security pressure
```

## Heatmap 1: Status Report Create Path

```text
Dashboard form                Medium
tRPC create schema            Medium
protectedProcedure workspace  High
service transaction           High
DB schema                     Medium
client notification side fx   High
React Query invalidation      Medium
```

Why:

- The form uses a union create/update schema, so changes can accidentally affect both modes (`apps/dashboard/src/components/forms/status-report/form.tsx:62-89`).
- `protectedProcedure` owns authentication and workspace resolution, so mistakes can become tenant-boundary bugs (`packages/api/src/trpc.ts:148-200`).
- `createStatusReport` writes report, associations, update, and audit rows in one transaction (`packages/services/src/status-report/create.ts:32-98`).
- Subscriber notification is currently triggered from client `onSuccess` after create success, which creates partial-failure risk (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`).

## Heatmap 2: Public Status Page Query

```text
db.query.page.findFirst with relations   High
component status computation             Medium
openEvents/lastEvents derivation         High
tracker grouping                         Medium
public output parsing                    Medium
client render mapping                    Medium
```

Why:

- The router itself says SQL performance needs improvement (`packages/api/src/router/statusPage.ts:49-55`).
- It loads many relations in one query (`packages/api/src/router/statusPage.ts:80-113`).
- It derives several user-visible concepts in memory (`packages/api/src/router/statusPage.ts:136-400`).
- The client maps reports and maintenances into feed props, so shape changes can break public display (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:329-360`).

## Heatmap 3: Auth And Authorization

```text
NextAuth dashboard providers        Medium
workspace-slug cookie               Medium
protectedProcedure                  High
guardTRPCSource                     Low as security, Medium as routing convention
requireScope for apiKey/mcp actors  High
requireScope for user actors        High gap
```

Evidence:

- Dashboard providers include GitHub, Google, and dev/self-host Resend (`apps/dashboard/src/lib/auth/index.ts:15-24`; `apps/dashboard/src/lib/auth/providers.ts:5-27`).
- The proxy sets `workspace-slug` when the authenticated user lacks it (`apps/dashboard/src/proxy.ts:45-65`).
- `guardTRPCSource` says its header is spoofable and not a security boundary (`apps/dashboard/src/lib/trpc/shared.ts:17-29`).
- `requireScope` checks only `apiKey` and `mcp` actors, while user actors are a documented no-op for now (`packages/services/src/auth/require-scope.ts:11-30`).

## Heatmap 4: Performance Risks

```text
statusReport.list service enrichment  Low/Medium
dashboard 10_000 list request         Medium
statusPage.get public query           High
status-page client render sorting     Medium
```

Evidence:

- `enrichReportsBatch` deliberately avoids per-report relation queries (`packages/services/src/status-report/list.ts:66-161`).
- tRPC status-report list passes `limit: 10_000` for dashboard consumers because there is no paging UI today (`packages/api/src/router/statusReport.ts:199-218`).
- Public `statusPage.get` has a TODO to improve SQL performance and does large in-memory processing (`packages/api/src/router/statusPage.ts:49-55`; `packages/api/src/router/statusPage.ts:80-400`).
- The client sorts report updates in render even though the router orders them descending (`packages/api/src/router/statusPage.ts:87-90`; `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:171-174`).

## Risk-To-Test Matrix

| Risk | Best Test Layer | Existing Anchor |
| --- | --- | --- |
| Cross-workspace create | Router and service | `packages/api/src/router/statusReport.test.ts:77-207`; `packages/services/src/status-report/__tests__/status-report.test.ts:165-180` |
| Notification partial failure | Service or integration | Current UI side effect at `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`; spies at `packages/services/src/status-report/__tests__/status-report.test.ts:41-56` |
| Public event ordering | Router | Sort/event code at `packages/api/src/router/statusPage.ts:239-266`; `packages/api/src/router/statusPage.ts:364-379` |
| SSRF in webhook-like URLs | Utility/unit | `packages/utils/src/ssrf.test.ts:26-160` |
| Auth context missing workspace | API middleware | `packages/api/src/trpc.ts:148-200` |

## How To Read Heatmaps

Do not read "High" as "bad code." Read it as "change here with instrumentation, tests, and attention." For example, `createStatusReport` is high risk because it is important, not because it is sloppy: it checks scope, validates page ownership, updates associations, inserts an initial update, and emits audit rows (`packages/services/src/status-report/create.ts:24-99`).

