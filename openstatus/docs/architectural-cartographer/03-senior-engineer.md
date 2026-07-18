# Senior Engineer Guide

## Architectural Critique

Scores are 1 low to 5 high.

- Scalability: 3/5. The status-report service list avoids N+1 enrichment (`packages/services/src/status-report/list.ts:66-161`), but public `statusPage.get` loads many relations and performs heavy in-memory transformations in one request (`packages/api/src/router/statusPage.ts:80-400`).
- TypeScript discipline: 4/5. The repo uses inferred routers, Zod service schemas, and Drizzle/Zod row types (`packages/api/src/root.ts:1-6`; `packages/services/src/status-report/schemas.ts:12-92`; `packages/db/src/schema/status_reports/validation.ts:38-55`). Some `any` suppressions remain in server helpers (`apps/dashboard/src/lib/trpc/server.tsx:69-84`).
- Separation of concerns: 3/5. Service extraction is strong for status reports (`packages/services/src/status-report/create.ts:24-99`), but subscriber email is still triggered from dashboard UI after mutation success (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`).
- Testability: 4/5. Service tests use transactions and helpers (`packages/services/src/status-report/__tests__/status-report.test.ts:116-208`), and router tests cover IDOR-like cases (`packages/api/src/router/statusReport.test.ts:77-207`).
- Maintainability: 3/5. Domain packages and service exports help (`packages/services/package.json:8-81`), but public status-page computation is concentrated in one large router (`packages/api/src/router/statusPage.ts:65-400`).
- Security posture: 3/5. Cross-workspace tests exist (`packages/api/src/router/statusReport.test.ts:77-207`) and SSRF tests exist (`packages/utils/src/ssrf.test.ts:26-160`), but `requireScope` currently skips dashboard user actors and member-role enforcement is unfinished (`packages/services/src/auth/require-scope.ts:11-24`; `packages/services/src/auth/require-scope.ts:25-30`).
- Performance: 3/5. Batched enrichment is good (`packages/services/src/status-report/list.ts:66-161`); unbounded dashboard list sentinel and public page all-in-one query deserve monitoring (`packages/api/src/router/statusReport.ts:199-218`; `packages/api/src/router/statusPage.ts:49-55`).

## Performance Audit

Finding 1: Public status-page router does large data loading and transformation. The code itself says SQL query performance should be improved (`packages/api/src/router/statusPage.ts:49-55`), and the procedure loads workspace, reports/updates, maintenances, components, monitors/incidents, and groups (`packages/api/src/router/statusPage.ts:80-113`).

Corrected direction:

```ts
// Current anchor: packages/api/src/router/statusPage.ts:80-113
// Safer direction: split "shell", "events", and "uptime" reads so embed-only
// pages do not always pay for full relation loading.
const pageShell = await ctx.db.query.page.findFirst({ /* page + workspace only */ });
const events = await loadPageEvents({ pageId: pageShell.id, includeReports: true });
const trackers = componentsVisible
  ? await loadPageTrackers({ pageId: pageShell.id })
  : [];
```

Finding 2: Dashboard status-report list requests "effectively unlimited" rows with `limit: 10_000` because there is no paging UI (`packages/api/src/router/statusReport.ts:199-218`). The service can paginate (`packages/services/src/status-report/schemas.ts:75-87`; `packages/services/src/status-report/list.ts:184-206`), so the UI is the missing layer.

Corrected direction:

```tsx
// Current anchor: apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:30-32
const { data } = useQuery(
  trpc.statusReport.list.queryOptions({
    pageId: Number.parseInt(id),
    // Add page state and pass limit/offset after router exposes them.
  }),
);
```

Finding 3: Public client sorts `report.statusReportUpdates` in render when building banner content (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:165-174`). The router already orders updates descending (`packages/api/src/router/statusPage.ts:87-90`), so the UI can avoid mutating/sorting in render.

Corrected snippet:

```tsx
// Current anchor: apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:171-174
const lastUpdate = report.statusReportUpdates[0]; // server already orders desc by date.
if (!lastUpdate) return null;
```

## Security Audit

Finding 1: `guardTRPCSource` is only a convention filter. The comment says the header is spoofable and auth is enforced by procedures (`apps/dashboard/src/lib/trpc/shared.ts:17-29`). Any new protected operation must use `protectedProcedure` and service authorization (`packages/api/src/trpc.ts:148-264`; `packages/services/src/auth/require-scope.ts:25-48`).

Corrected snippet:

```ts
// Use this pattern for protected writes.
create: protectedProcedure
  .input(schema)
  .mutation(async ({ ctx, input }) => {
    return serviceWrite({ ctx: toServiceCtx(ctx), input }); // service still checks scope/ownership.
  });
```

Finding 2: Dashboard user actors bypass `requireScope`, so member-role checks are not enforced there yet (`packages/services/src/auth/require-scope.ts:11-30`). A senior owner should avoid assuming "write" means dashboard role authorization until that project lands.

Corrected direction:

```ts
// Current anchor: packages/services/src/auth/require-scope.ts:25-30
if (actor.type === "user") {
  // Future direction: load/check workspace membership role before returning.
  // Do not treat this as complete authorization today.
  return;
}
```

Finding 3: Dashboard OAuth providers enable dangerous email account linking (`apps/dashboard/src/lib/auth/providers.ts:5-10`). That may be intentional for this product, but it needs explicit threat-model ownership.

Corrected direction:

```ts
// Current anchor: apps/dashboard/src/lib/auth/providers.ts:5-10
// Require a documented reason before keeping allowDangerousEmailAccountLinking: true.
export const GitHubProvider = GitHub({
  allowDangerousEmailAccountLinking: true,
});
```

## TypeScript Discipline Review

Strong patterns:

- Inferred `AppRouter` creates typed clients (`packages/api/src/root.ts:1-6`; `apps/dashboard/src/lib/trpc/client.tsx:9-12`).
- Zod service schemas define canonical inputs (`packages/services/src/status-report/schemas.ts:12-92`).
- Drizzle/Zod validation schemas infer row types (`packages/db/src/schema/status_reports/validation.ts:38-55`).

Weak spots:

- Some server helpers use `any` suppressions around generic query options (`apps/dashboard/src/lib/trpc/server.tsx:69-84`).
- Some public router-local types are nested inside a large procedure, making them harder to reuse/test (`packages/api/src/router/statusPage.ts:309-327`).

## Custom Abstractions Inventory

- `createTRPCRouter`, `protectedProcedure`, `publicProcedure`: tRPC factory and auth boundary (`packages/api/src/trpc.ts:132-142`; `packages/api/src/trpc.ts:255-264`).
- `toServiceCtx`, `toTRPCError`: API-to-service adapter (`packages/api/src/service-adapter.ts:18-87`).
- `ServiceContext`, `withTransaction`: service execution context and transaction wrapper (`packages/services/src/context.ts:22-47`).
- `requireScope`, `matchesScope`: actor-scope authorization (`packages/services/src/auth/require-scope.ts:25-48`; `packages/services/src/auth/matches-scope.ts:19-28`).
- `endingLink`: client-side router-to-runtime selector (`apps/dashboard/src/lib/trpc/shared.ts:38-76`).
- `fetchQueryOrNotFound`: server-side query helper for resource-gated layouts (`apps/dashboard/src/lib/trpc/server.tsx:99-117`).
- `useIsMobile`: shared UI responsive hook (`packages/ui/src/hooks/use-mobile.ts:7-23`).

## Testing Assessment And One Runnable Test

Existing coverage is meaningful around status-report workspace boundaries and service behavior (`packages/api/src/router/statusReport.test.ts:77-207`; `packages/services/src/status-report/__tests__/status-report.test.ts:116-208`). A risky untested behavior is public status-page sorting when a report has no updates. The comparator has suspicious empty-array handling (`packages/api/src/router/statusPage.ts:364-379`).

Suggested runnable test file: `packages/api/src/router/statusPage.sort.test.ts`

```ts
import { describe, expect, test } from "bun:test";

type Report = {
  id: number;
  statusReportUpdates: { date: Date }[];
};

function currentComparator(a: Report, b: Report) {
  if (a.statusReportUpdates.length === 0 && b.statusReportUpdates.length === 0)
    return 0;
  if (a.statusReportUpdates.length === 0) return -1;
  if (b.statusReportUpdates.length === 0) return -1;
  return (
    b.statusReportUpdates[b.statusReportUpdates.length - 1].date.getTime() -
    a.statusReportUpdates[a.statusReportUpdates.length - 1].date.getTime()
  );
}

describe("statusPage report sorting", () => {
  test("puts reports without updates first, then newest updated reports", () => {
    const reports: Report[] = [
      { id: 1, statusReportUpdates: [{ date: new Date("2026-01-01") }] },
      { id: 2, statusReportUpdates: [] },
      { id: 3, statusReportUpdates: [{ date: new Date("2026-02-01") }] },
    ];
    expect(reports.sort(currentComparator).map((r) => r.id)).toEqual([2, 3, 1]);
  });
});
```

This test is intentionally documentation-only here; production code was not modified.

## Bug Injection Exercise

1. Symptom: Creating a status report succeeds, but subscribers are not emailed. Test scenario: mock `emailRouter.sendStatusReport`, create with `notifySubscribers: true`, verify the client calls the email mutation after create success (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`).
2. Symptom: Public page shows an old incident above a newer report. Test scenario: seed two reports with different update dates, then inspect `statusPage.get` sorting (`packages/api/src/router/statusPage.ts:364-379`).
3. Symptom: A user can attach another workspace's component to a report. Test scenario: reproduce the existing router test for cross-workspace component rejection (`packages/api/src/router/statusReport.test.ts:181-207`).
4. Symptom: A public status page returns 500 for malformed configuration. Test scenario: seed invalid `page.configuration`, call `statusPage.get`, expect null rather than crash (`packages/api/src/router/statusPage.ts:122-129`).
5. Symptom: A dashboard page silently hangs on missing data. Test scenario: make `statusReport.list` fail and verify the UI does more than `return null` (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:54-55`).

## Git History Learning Exercise

1. `extract status report writes into service layer` - implies router logic was too coupled and services became reusable across tRPC, Connect, and background flows (`packages/api/src/router/statusReport.ts:21-23`; `packages/services/package.json:8-81`).
2. `batch enrich status reports to remove N+1 queries` - implies dashboard list scale exposed repeated relation lookups (`packages/services/src/status-report/list.ts:66-161`).
3. `split tRPC routes between edge and lambda runtimes` - implies some dependencies are edge-safe while Stripe/email-style code needs Node runtime (`apps/dashboard/src/lib/trpc/shared.ts:38-76`; `packages/api/src/lambda.ts:7-14`).
4. `add workspace isolation tests for status reports` - implies IDOR risk was important enough to encode in tests (`packages/api/src/router/statusReport.test.ts:77-207`).
5. `add public status page component groups` - implies public rendering evolved from flat components into grouped trackers (`packages/api/src/router/statusPage.ts:268-360`).

## If I Owned This Codebase

- Move status-report subscriber notification to a server-owned workflow. Effort: medium. Impact: high. Anchor: client TODO and email side effect (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`).
- Split `statusPage.get` into smaller pure helpers and add tests for event computation and sorting. Effort: medium-high. Impact: high. Anchor: large public procedure (`packages/api/src/router/statusPage.ts:65-400`).
- Add dashboard member-role enforcement for `user` actors. Effort: high. Impact: high. Anchor: no-op scope behavior (`packages/services/src/auth/require-scope.ts:11-30`).
- Add pagination to dashboard status reports. Effort: medium. Impact: medium. Anchor: sentinel limit (`packages/api/src/router/statusReport.ts:199-218`).
- Remove `any` suppressions in tRPC server helpers by introducing typed wrapper helpers. Effort: low-medium. Impact: medium. Anchor: `any` suppressions (`apps/dashboard/src/lib/trpc/server.tsx:69-84`).

