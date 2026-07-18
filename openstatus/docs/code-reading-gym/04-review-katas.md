# Review Katas

Each kata is a fake pull request. Read the premise, inspect the referenced real code, and write a review comment.

## Kata 1: "Add Severity To Status Reports"

Fake PR summary:

```text
Adds a severity select to the dashboard form and sends severity in tRPC create input.
No DB changes included.
```

Review anchors:

- Current form schema fields (`apps/dashboard/src/components/forms/status-report/form.tsx:62-77`)
- Current tRPC create input (`packages/api/src/router/statusReport.ts:24-32`)
- Current service create input (`packages/services/src/status-report/schemas.ts:12-20`)
- Current DB report columns (`packages/db/src/schema/status_reports/status_reports.ts:15-39`)

Strong review comment:

```text
This adds severity to the UI/API surface but does not persist it. The current status report table has id, status, title, workspaceId, pageId, createdAt, and updatedAt, but no severity column (`packages/db/src/schema/status_reports/status_reports.ts:15-39`). Please add the DB schema/default, update row validation, update `CreateStatusReportInput`, and insert severity in `createStatusReport`; otherwise the form value disappears after submit.
```

## Kata 2: "Filter Public Draft Reports In The React Client"

Fake PR summary:

```text
Adds `report.isDraft ? null : ...` in the public client feed map.
```

Review anchors:

- Public router loads and computes reports before client rendering (`packages/api/src/router/statusPage.ts:80-400`)
- Public client feed filters reports from `page.statusReports` (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:329-360`)

Strong review comment:

```text
Filtering drafts only in the React client is too late. `statusPage.get` already computes `openEvents`, `lastEvents`, page status, and sorted reports before the client renders (`packages/api/src/router/statusPage.ts:231-400`). Drafts could still influence banners or event calculations. Please filter or model draft visibility in the public router/service layer before computed public fields are built.
```

## Kata 3: "Trust x-trpc-source For Internal Mutations"

Fake PR summary:

```text
Adds a mutation as publicProcedure because guardTRPCSource already blocks non-client calls.
```

Review anchors:

- Header guard comment (`apps/dashboard/src/lib/trpc/shared.ts:17-29`)
- Protected procedure middleware (`packages/api/src/trpc.ts:148-264`)

Strong review comment:

```text
`x-trpc-source` is not auth. The guard's own comment says the header is trivially spoofable and that auth is enforced by `protectedProcedure` (`apps/dashboard/src/lib/trpc/shared.ts:17-29`). Any mutation that writes workspace data should use `protectedProcedure` so session and active workspace are resolved (`packages/api/src/trpc.ts:148-200`).
```

## Kata 4: "Loop Over Reports And Query Updates"

Fake PR summary:

```text
Simplifies listStatusReports by querying updates inside a for-loop.
```

Review anchors:

- Batched enrichment comment (`packages/services/src/status-report/list.ts:66-71`)
- Batched update query (`packages/services/src/status-report/list.ts:83-95`)
- Dashboard list sentinel (`packages/api/src/router/statusReport.ts:199-218`)

Strong review comment:

```text
This reintroduces an N+1 pattern. The existing implementation explicitly batches updates, component associations, and page data because dashboard consumers request an effectively unbounded list today (`packages/services/src/status-report/list.ts:66-71`; `packages/api/src/router/statusReport.ts:199-218`). Please keep enrichment batched or add pagination before simplifying this.
```

## Kata 5: "Show Loading By Returning Null"

Fake PR summary:

```text
Adds more `if (!data) return null` branches to dashboard pages.
```

Review anchors:

- Current status-report page null branch (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:54-55`)
- Server layout prefetch and hydration (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/layout.tsx:16-35`)

Strong review comment:

```text
Returning null hides the difference between loading, error, and missing data. This page already has server prefetch/hydration (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/layout.tsx:16-35`), so a persistent null likely means something actionable. Please use explicit loading/error/empty states rather than expanding the blank UI pattern.
```

## Kata 6: "Move Email Notification Into Service Transaction"

Fake PR summary:

```text
Calls subscriber dispatch inside `withTransaction` before returning create result.
```

Review anchors:

- Current create transaction (`packages/services/src/status-report/create.ts:32-98`)
- Current UI notification side effect (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`)
- Subscription test spies (`packages/services/src/status-report/__tests__/status-report.test.ts:41-56`)

Strong review comment:

```text
Moving notification server-side is directionally good, but sending inside the DB transaction risks notifying subscribers for a write that later rolls back. The service transaction currently owns DB rows and audit rows (`packages/services/src/status-report/create.ts:32-98`). Please define an after-commit or durable queue boundary and add tests using the existing subscription spies (`packages/services/src/status-report/__tests__/status-report.test.ts:41-56`).
```

