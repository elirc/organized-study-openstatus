# Contract Patterns

## Pattern 1: Enum Source Of Truth

Real code:

```ts
// packages/db/src/schema/status_reports/status_reports.ts:8-13
export const statusReportStatus = [
  "investigating",
  "identified",
  "monitoring",
  "resolved",
] as const;
```

Consumers:

- DB status column uses the enum (`packages/db/src/schema/status_reports/status_reports.ts:15-20`).
- DB validation exports `statusReportStatusSchema` (`packages/db/src/schema/status_reports/validation.ts:10-10`).
- Dashboard form imports `statusReportStatus` and uses `z.enum(statusReportStatus)` (`apps/dashboard/src/components/forms/status-report/form.tsx:17-21`; `apps/dashboard/src/components/forms/status-report/form.tsx:62-69`).
- Service schema reuses `statusReportStatusSchema` (`packages/services/src/status-report/schemas.ts:1-5`; `packages/services/src/status-report/schemas.ts:12-20`).

Illustrative fake code:

```ts
// Illustrative fake code.
export const states = ["draft", "published"] as const;
export const stateSchema = z.enum(states);
type State = z.infer<typeof stateSchema>;
```

Review question: is a PR hardcoding status strings in a component instead of importing the enum?

## Pattern 2: Runtime Schema At Every Boundary

Real code:

```ts
// apps/dashboard/src/components/forms/status-report/form.tsx:62-69
const schema = z.object({
  status: z.enum(statusReportStatus),
  title: z.string(),
  message: z.string(),
  date: z.date(),
  pageComponents: z.array(z.number()),
  notifySubscribers: z.boolean().optional(),
});
```

```ts
// packages/api/src/router/statusReport.ts:24-32
const createStatusReportTRPCInput = z.object({
  title: z.string(),
  status: CreateStatusReportInput.shape.status,
  pageId: z.number(),
  pageComponents: z.array(z.number()),
  date: z.coerce.date(),
  message: z.string(),
  notifySubscribers: z.boolean().nullish(),
});
```

```ts
// packages/services/src/status-report/schemas.ts:12-20
export const CreateStatusReportInput = z.object({
  title: z.string().min(1).max(256),
  status: statusReportStatusSchema,
  message: z.string(),
  date: z.coerce.date(),
  pageId: z.number().int(),
  pageComponentIds: z.array(z.number().int()).default([]),
});
```

Why: each layer has a different trust boundary. The form helps the user, tRPC protects the API boundary, and service schemas protect reusable domain operations.

## Pattern 3: Inferred Router Type

Real code:

```ts
// packages/api/src/root.ts:1-6
export const appRouter = mergeRouters(edgeRouter, lambdaRouter);
export type AppRouter = typeof appRouter;
```

```ts
// apps/dashboard/src/lib/trpc/client.tsx:9-12
import type { AppRouter } from "@openstatus/api";
export const { TRPCProvider, useTRPC, useTRPCClient } =
  createTRPCContext<AppRouter>();
```

Why: client procedure names, inputs, and outputs stay synchronized with the server router table. If `statusReport.create` changes shape, typed callers feel it.

Illustrative fake code:

```ts
// Illustrative fake code.
type ServerRoutes = typeof serverRoutes;
const client = makeTypedClient<ServerRoutes>();
```

Review question: is a caller using raw strings or untyped fetch where typed tRPC helpers exist?

## Pattern 4: Rich Read Model

Real code:

```ts
// packages/services/src/status-report/list.ts:49-59
export type StatusReportWithRelations = StatusReport & {
  updates: StatusReportUpdate[];
  pageComponents: PageComponent[];
  pageComponentIds: number[];
  page: (Page & { pageComponents: PageComponent[] }) | null;
};
```

Why: read operations often return richer shapes than a single DB row. The service list builds this read model with batched queries (`packages/services/src/status-report/list.ts:66-161`).

Illustrative fake code:

```ts
// Illustrative fake code.
type OrderWithLines = Order & {
  lines: OrderLine[];
  customer: Customer;
};
```

Review question: is this a table row, an API response, or a service read model?

