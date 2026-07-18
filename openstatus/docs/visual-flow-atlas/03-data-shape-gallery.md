# Data Shape Gallery

This file shows how the same product idea changes shape as it crosses layers.

## Shape 1: Form Values

Real code:

```ts
// apps/dashboard/src/components/forms/status-report/form.tsx:62-77
const schema = z.object({
  status: z.enum(statusReportStatus),
  title: z.string(),
  message: z.string(),
  date: z.date(),
  pageComponents: z.array(z.number()),
  notifySubscribers: z.boolean().optional(),
});

const updateSchema = schema.omit({
  message: true,
  date: true,
  notifySubscribers: true,
});
```

Visual shape:

```text
Create form values
  title: string
  status: "investigating" | "identified" | "monitoring" | "resolved"
  message: string
  date: Date
  pageComponents: number[]
  notifySubscribers?: boolean

Update form values
  title: string
  status: same enum
  pageComponents: number[]
```

Teaching note: the union type exists because the same form component supports create and update modes (`apps/dashboard/src/components/forms/status-report/form.tsx:71-89`).

## Shape 2: tRPC Create Input

Real code:

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

Visual shape:

```text
tRPC create input
  title
  status
  pageId
  pageComponents        -> legacy/dashboard-facing field name
  date                  -> coerced to Date
  message
  notifySubscribers     -> API receives it, service create does not own it
```

Illustrative fake code:

```ts
// Illustrative fake code: why the tRPC shape can differ from service shape.
const dashboardPayload = { pageComponents: [1, 2, 3] };
const servicePayload = { pageComponentIds: dashboardPayload.pageComponents };
```

## Shape 3: Service Create Input

Real code:

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

Visual shape:

```text
Service create input
  title                 -> constrained length
  status                -> canonical DB-backed enum
  message
  date                  -> coerced to Date
  pageId                -> integer
  pageComponentIds      -> canonical service name
```

Teaching note: services should be reusable by other transports, so their schemas should not inherit dashboard-only names unless the domain actually uses them (`packages/api/src/router/statusReport.ts:21-23`; `packages/services/package.json:8-81`).

## Shape 4: Database Rows

Real code:

```ts
// packages/db/src/schema/status_reports/status_reports.ts:15-34
export const statusReport = sqliteTable("status_report", {
  id: integer("id").primaryKey(),
  status: text("status", { enum: statusReportStatus }).notNull(),
  title: text("title", { length: 256 }).notNull(),
  workspaceId: integer("workspace_id").references(() => workspace.id),
  pageId: integer("page_id").references(() => page.id, { onDelete: "cascade" }),
  createdAt: integer("created_at", { mode: "timestamp" }).default(sql`(strftime('%s', 'now'))`),
  updatedAt: integer("updated_at", { mode: "timestamp" }).default(sql`(strftime('%s', 'now'))`),
});
```

```ts
// packages/db/src/schema/status_reports/status_reports.ts:41-59
export const statusReportUpdate = sqliteTable("status_report_update", {
  id: integer("id").primaryKey(),
  status: text("status", { enum: statusReportStatus }).notNull(),
  date: integer("date", { mode: "timestamp" }).notNull(),
  message: text("message").notNull(),
  statusReportId: integer("status_report_id")
    .references(() => statusReport.id, { onDelete: "cascade" })
    .notNull(),
});
```

Visual shape:

```text
status_report
  id
  status
  title
  workspace_id
  page_id
  timestamps

status_report_update
  id
  status
  date
  message
  status_report_id
  timestamps
```

Key insight: the report stores the current state and title; updates store the timeline messages (`packages/db/src/schema/status_reports/status_reports.ts:15-63`).

## Shape 5: Join Table

Real code:

```ts
// packages/db/src/schema/page_components/page_components.ts:124-143
export const statusReportsToPageComponents = sqliteTable(
  "status_report_to_page_component",
  {
    statusReportId: integer("status_report_id")
      .notNull()
      .references(() => statusReport.id, { onDelete: "cascade" }),
    pageComponentId: integer("page_component_id")
      .notNull()
      .references(() => pageComponent.id, { onDelete: "cascade" }),
    createdAt: integer("created_at", { mode: "timestamp" }).default(sql`(strftime('%s', 'now'))`),
  },
  (t) => [primaryKey({ columns: [t.statusReportId, t.pageComponentId] })],
);
```

Visual shape:

```text
Report 7 affects Component 3
  status_report_to_page_component
    status_report_id = 7
    page_component_id = 3
```

Analogy: a report is the incident bulletin; the join table is the list of services named in the bulletin.

## Shape 6: Enriched Service Return

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

Visual shape:

```text
StatusReportWithRelations
  raw status_report columns
  updates[]
  pageComponents[]
  pageComponentIds[]
  page or null
```

Teaching note: this is not a DB table. It is a service-level read model assembled by `enrichReportsBatch` (`packages/services/src/status-report/list.ts:66-161`).

## Shape 7: Public Page Return

The public `statusPage.get` query returns a parsed public page with computed values such as `status`, `lastEvents`, `openEvents`, `trackers`, `statusReports`, and `maintenances` (`packages/api/src/router/statusPage.ts:385-400`). The client then consumes that shape to render banners, trackers, and feed entries (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:136-414`).

Visual shape:

```text
Public page
  title, description, slug, configuration
  status: "success" | "degraded" | "error" | "info"
  openEvents[]
  lastEvents[]
  trackers[]
  statusReports[]
  maintenances[]
  pageComponents[]
```

Illustrative fake code:

```ts
// Illustrative fake code: simplified public read model.
const publicPage = {
  title: "Acme Status",
  status: "degraded",
  openEvents: [{ type: "report", id: 42 }],
  trackers: [{ type: "component", component: { name: "API", status: "degraded" } }],
  statusReports: [{ id: 42, updates: [{ message: "Investigating" }] }],
};
```

