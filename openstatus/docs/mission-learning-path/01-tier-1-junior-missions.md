# Tier 1: Junior Missions

### Mission 1: The App's Heartbeat
**Tier:** Junior
**Time Estimate:** 25 minutes
**Goal:** Learn how this repo is installed, started, checked, and tested.
**The Concept:** Scripts are the heartbeat of a repo. In OpenStatus, they are like the status-page pulse: if you cannot read them, you cannot tell whether the system is alive.
**Design Intent Before You Read the Code:** Root scripts should reveal package manager, dev commands, test commands, and setup flow. Poor script design makes onboarding unreliable.
**Find It In The Code:** Open `package.json:3-20`, `package.json:33-38`, `turbo.json:51-64`, `README.md:93-143`.

```json
"dev:dashboard": "turbo run dev --filter='./apps/dashboard' --filter='./packages/db'",
"dx": "turbo run dx",
"test": "turbo run test"
// dev:dashboard starts the dashboard and DB package together.
// dx is a Turbo task, not a package script by itself.
// test delegates to every package with a test task.
```

```json
"dx": {
  "cache": false,
  "dependsOn": ["seed"]
}
// dx depends on seed, seed depends on migrate, migrate depends on env.
// Setup is encoded in the task graph, not only in README prose.
```

**The Aha Moment:** Before reading app code, read how the repo expects to be operated.
**Socratic Checkpoint:**
1. Which package manager is required?
2. Which command starts dashboard dev?
3. What does `dx` depend on?
4. Where is Turso started manually?
5. Which command runs all tests?

How to self-grade: Strong answers cite `package.json:15-20`, `package.json:33-38`, `turbo.json:51-64`, and `README.md:116-138`.
**Connects To:** Mission 2, because scripts tell you which folders matter.

### Mission 2: The Folder Mental Map
**Tier:** Junior
**Time Estimate:** 30 minutes
**Goal:** Build a map of app surfaces and shared packages.
**The Concept:** A monorepo is like a status page with many components: each component has a job, and the page only works when their boundaries are clear.
**Design Intent Before You Read the Code:** Apps should deploy; packages should be reused. Poor boundaries create circular ownership.
**Find It In The Code:** Open `apps/dashboard/package.json:1-36`, `apps/status-page/package.json:1-28`, `apps/web/package.json:1-19`, `packages/services/package.json:8-81`.

```json
"name": "@openstatus/dashboard",
"scripts": {
  "dev": "next dev --turbopack"
}
// This is a deployable Next app, not a shared library.
```

```json
"exports": {
  "./status-report": {
    "import": "./src/status-report/index.ts"
  }
}
// packages/services exposes domain modules that API routes can import.
```

**The Aha Moment:** `apps/*` are product surfaces; `packages/*` are the system's shared organs.
**Socratic Checkpoint:**
1. Which package owns dashboard UI?
2. Which package owns business services?
3. Which package owns DB schema?
4. Which app renders public status pages?
5. Why is `packages/ui` not a deployable app?

How to self-grade: Strong answers cite package names and scripts from `apps/dashboard/package.json:1-36`, `apps/status-page/package.json:1-28`, and exports from `packages/services/package.json:8-81`.
**Connects To:** Mission 3 and Mission 4, because folder ownership explains type and component placement.

### Mission 3: TypeScript Is a Contract
**Tier:** Junior
**Time Estimate:** 35 minutes
**Goal:** See how TypeScript and Zod protect data crossing boundaries.
**The Concept:** Types are the incident template for code: everyone agrees what fields mean before the alert goes out.
**Design Intent Before You Read the Code:** UI, API, service, and DB contracts should line up without copying random strings.
**Find It In The Code:** Open `packages/db/src/schema/status_reports/status_reports.ts:8-20`, `packages/services/src/status-report/schemas.ts:12-20`, `apps/dashboard/src/components/forms/status-report/form.tsx:62-77`.

```ts
export const statusReportStatus = [
  "investigating",
  "identified",
  "monitoring",
  "resolved",
] as const;
// One enum source gives UI and backend the same allowed statuses.
```

```ts
const schema = z.object({
  status: z.enum(statusReportStatus),
  title: z.string(),
  message: z.string(),
  date: z.date(),
});
// The form validates the same status set before the API sees it.
```

**The Aha Moment:** TypeScript helps at compile time, but Zod protects runtime inputs.
**Socratic Checkpoint:**
1. Where are report statuses declared?
2. Where does the form import them?
3. Which service schema validates create input?
4. Why is `z.coerce.date()` useful at API boundaries?
5. What bug appears if UI invents a fifth status?

How to self-grade: Strong answers cite `packages/db/src/schema/status_reports/status_reports.ts:8-20`, `apps/dashboard/src/components/forms/status-report/form.tsx:17-21`, and `packages/services/src/status-report/schemas.ts:12-20`.
**Connects To:** Mission 6 and Mission 12, because props and API contracts are typed contracts too.

### Mission 4: Your First React Component
**Tier:** Junior
**Time Estimate:** 25 minutes
**Goal:** Understand a small reusable React component.
**The Concept:** A simple component is like a reusable status-page block: it owns a tiny visual contract.
**Design Intent Before You Read the Code:** Small UI components should pass normal props through and keep styling predictable.
**Find It In The Code:** Open `packages/ui/src/components/blocks/status-blank.tsx:63-73`.

```tsx
export function StatusBlankTitle({
  children, // visible content from the caller
  className, // caller extension point
  ...props // keeps normal div props available
}: React.ComponentProps<"div">) {
  return (
    <div className={cn("font-medium", className)} {...props}>
      {children}
    </div>
  );
}
```

**The Aha Moment:** Good reusable components hide styling defaults but keep HTML flexibility.
**Socratic Checkpoint:**
1. Why use `React.ComponentProps<"div">`?
2. Why merge `className` with `cn`?
3. What does `children` represent?
4. What happens if `{...props}` is removed?
5. Why is this component in `packages/ui`?

How to self-grade: Strong answers cite `packages/ui/src/components/blocks/status-blank.tsx:63-73` and mention reuse across app surfaces.
**Connects To:** Mission 16, because composition starts with small components.

### Mission 5: Your First Node.js Route
**Tier:** Junior
**Time Estimate:** 30 minutes
**Goal:** Understand a tRPC HTTP route handler.
**The Concept:** The route handler is the front desk: it checks the request shape, creates context, then hands the work to the router.
**Design Intent Before You Read the Code:** A route should be thin. Business rules belong behind routers/services.
**Find It In The Code:** Open `apps/status-page/src/app/api/trpc/edge/[trpc]/route.ts:11-24`.

```ts
const handler = (req: NextRequest) => {
  const blocked = guardTRPCSource(req); // convention filter, not real auth
  if (blocked) return blocked;

  return fetchRequestHandler({
    endpoint: "/api/trpc/edge",
    router: edgeRouter, // actual procedure table
    req: req,
    createContext: () => createTRPCContext({ req, auth }), // request/session/db context
    onError: createOnError("edge"),
  });
};
```

**The Aha Moment:** The HTTP route is thin because tRPC and services own the real behavior.
**Socratic Checkpoint:**
1. Which router does this route expose?
2. Where is context created?
3. What does `guardTRPCSource` check?
4. Why is this not enough for security?
5. Which exports make GET and POST work?

How to self-grade: Strong answers cite `apps/status-page/src/app/api/trpc/edge/[trpc]/route.ts:11-24` and `apps/dashboard/src/lib/trpc/shared.ts:17-29`.
**Connects To:** Mission 13, because middleware explains what happens after the route handler.

### Mission 6: Props Are a Typed Contract
**Tier:** Junior
**Time Estimate:** 35 minutes
**Goal:** Read component props as a contract between parent and child.
**The Concept:** Props are like affected components on a status report: the parent chooses what to send, the child promises how to use it.
**Design Intent Before You Read the Code:** `FormSheetStatusReport` should own sheet behavior while letting the page own the mutation.
**Find It In The Code:** Open `apps/dashboard/src/components/forms/status-report/sheet.tsx:22-74`.

```tsx
export function FormSheetStatusReport({
  children,
  defaultValues,
  onSubmit,
  items,
  warning,
}: Omit<React.ComponentProps<typeof FormSheetTrigger>, "onSubmit"> & {
  defaultValues?: FormValues; // create vs edit mode
  onSubmit: (values: FormValues) => Promise<void>; // parent owns persistence
  items: CheckboxTreeItem[]; // affected page components
  warning?: React.ReactNode; // optional unresolved-report warning
}) {
  const [open, setOpen] = useState(false);
}
```

**The Aha Moment:** The sheet manages UI state; the page manages data mutation.
**Socratic Checkpoint:**
1. Which prop sends selected components into the form?
2. Which prop lets the parent save data?
3. Why is `warning` optional?
4. Why does the child close the sheet after `await onSubmit(values)`?
5. What would break if `onSubmit` were synchronous?

How to self-grade: Strong answers cite `apps/dashboard/src/components/forms/status-report/sheet.tsx:22-74` and connect it to page mutation lines `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:75-103`.
**Connects To:** Mission 9, because prop contracts feed state transitions.

### Mission 7: Following Data Into the App
**Tier:** Junior
**Time Estimate:** 40 minutes
**Goal:** Trace dashboard data fetches from route params into React Query.
**The Concept:** Data enters like a status signal from a monitor: first identify the source, then follow where it is interpreted.
**Design Intent Before You Read the Code:** The page id should control which status page and reports are loaded.
**Find It In The Code:** Open `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:23-32`, `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/layout.tsx:16-35`.

```tsx
const { id } = useParams<{ id: string }>(); // URL segment [id]
const { data: page } = useQuery(
  trpc.page.get.queryOptions({ id: Number.parseInt(id) }),
);
const { data: statusReports, refetch } = useQuery(
  trpc.statusReport.list.queryOptions({ pageId: Number.parseInt(id) }),
);
// Same route id feeds both the page query and report-list query.
```

**The Aha Moment:** Dynamic route params become typed query inputs.
**Socratic Checkpoint:**
1. Why is `id` parsed as a number?
2. Which query loads page details?
3. Which query loads reports?
4. Where does server prefetch happen?
5. Why use `HydrateClient`?

How to self-grade: Strong answers cite `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:23-32` and `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/layout.tsx:16-35`.
**Connects To:** Mission 14, because end-to-end trace starts with route data.

### Mission 8: Navigation Is the App's Skeleton
**Tier:** Junior
**Time Estimate:** 30 minutes
**Goal:** Understand how layouts frame nested dashboard routes.
**The Concept:** Navigation is the status page frame: users need stable context while content changes.
**Design Intent Before You Read the Code:** Layouts should fetch required resources, render shared header/tabs/sidebar, and pass nested pages through `children`.
**Find It In The Code:** Open `apps/dashboard/src/app/(dashboard)/status-pages/[id]/layout.tsx:18-50`.

```tsx
export default async function Layout({ children, params }: Props) {
  const { id } = await params;
  await fetchQueryOrNotFound(
    trpc.page.get.queryOptions({ id: Number.parseInt(id) }),
  ); // gate route on page existence

  return (
    <HydrateClient>
      <AppHeader>{/* shared route chrome */}</AppHeader>
      <Tabs /> {/* status page sub-navigation */}
      <main className="flex-1">{children}</main> {/* nested route output */}
    </HydrateClient>
  );
}
```

**The Aha Moment:** In App Router, layouts own stable UI and data gates; pages own route-specific work.
**Socratic Checkpoint:**
1. What does this layout fetch before rendering?
2. Why use `fetchQueryOrNotFound`?
3. Which components are shared route chrome?
4. Where does the child page render?
5. What user experience breaks if this layout disappears?

How to self-grade: Strong answers cite `apps/dashboard/src/app/(dashboard)/status-pages/[id]/layout.tsx:18-50` and `apps/dashboard/src/lib/trpc/server.tsx:99-117`.
**Connects To:** Mission 14, because the feature trace includes layout prefetch and page rendering.

