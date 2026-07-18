# Junior Engineer Guide

## Setup And Run

The root uses pnpm 10.26.0 and workspace globs for apps and packages (`package.json:33-38`). The main scripts are `pnpm dev`, `pnpm dev:web`, `pnpm dev:status-page`, `pnpm dev:dashboard`, `pnpm dx`, `pnpm lint`, and `pnpm test` (`package.json:3-20`).

Manual setup reconstructed from repo docs:

1. Install Node 20+, pnpm, Bun, and Turso CLI because the README lists them as manual requirements (`README.md:93-101`).
2. Run `pnpm install`; the README's manual setup uses that command (`README.md:110-114`).
3. Start libSQL with `turso dev --db-file openstatus-dev.db`; the README and process-compose both use that command (`README.md:116-122`; `process-compose.yaml:9-13`).
4. Run `pnpm dx`; Turbo defines `dx` as depending on `seed`, `seed` on `migrate`, and `migrate` on `env` (`README.md:124-128`; `turbo.json:51-64`).
5. Launch a surface with `pnpm dev:dashboard`, `pnpm dev:status-page`, or `pnpm dev:web` (`package.json:15-17`; `README.md:130-143`).

Environment setup exists but is split. Docker examples include `DATABASE_URL`, `DATABASE_AUTH_TOKEN`, `RESEND_API_KEY`, `AUTH_SECRET`, `SELF_HOST`, and `NEXT_PUBLIC_URL` (`.env.docker-lightweight.example:9-39`; `.env.docker-lightweight.example:47-57`). Package examples include DB values (`packages/db/.env.example:1-2`) and dashboard values (`apps/dashboard/.env.example:1-87`). The repo has setup instructions, but no single root `.env.example`; use the app/package examples that match the surface you run (`README.md:58-143`; `apps/dashboard/.env.example:17-19`; `packages/db/.env.example:1-2`).

## Folder Orientation

- `apps/` contains deployable surfaces and services; this repo has `dashboard`, `status-page`, `web`, `workflows`, Go checkers, and supporting services (`apps/dashboard/package.json:1-6`; `apps/status-page/package.json:1-6`; `apps/web/package.json:1-6`; `README.md:144-153`).
- `packages/api` contains tRPC router definitions and request context (`packages/api/src/root.ts:1-6`; `packages/api/src/trpc.ts:59-119`).
- `packages/services` contains reusable business operations exported by domain (`packages/services/package.json:8-81`).
- `packages/db` contains Drizzle connection, schema, migrations, seed, and DB scripts (`packages/db/package.json:5-14`; `packages/db/src/db.ts:1-12`).
- `packages/ui` contains shared UI components and hooks, including `useIsMobile` (`packages/ui/src/hooks/use-mobile.ts:1-23`).
- `packages/tinybird` contains analytics schemas, endpoints, and pipes; the public status-page router imports Tinybird procedures for uptime data (`packages/api/src/router/statusPage.ts:41-47`).

Three important frontend folders:

- `apps/dashboard/src/app`: Next App Router route tree; `status-pages/[id]/status-reports/page.tsx` is a client page for creating and listing reports (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:1-128`).
- `apps/dashboard/src/components`: composed UI, forms, data tables, and navigation; the status-report form lives at `components/forms/status-report/form.tsx` (`apps/dashboard/src/components/forms/status-report/form.tsx:79-417`).
- `apps/status-page/src/app`: public status-page route tree; the public page delegates to a client component (`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/page.tsx:23-25`).

Three important backend folders:

- `packages/api/src/router`: tRPC routers; `statusReport.ts` exposes create, update, delete, get, and list procedures (`packages/api/src/router/statusReport.ts:58-222`).
- `packages/services/src/status-report`: service implementation; create/list/update/delete/notify are separate files (`packages/services/src/status-report/create.ts:24-99`; `packages/services/src/status-report/list.ts:164-226`).
- `packages/db/src/schema`: Drizzle tables and Zod schemas; `status_reports` and `page_components` define the report/update/join-table shape (`packages/db/src/schema/status_reports/status_reports.ts:15-89`; `packages/db/src/schema/page_components/page_components.ts:124-157`).

## Entry Points With Annotated Code

Frontend dashboard tRPC provider (`apps/dashboard/src/lib/trpc/client.tsx:39-64`):

```tsx
export function TRPCReactProvider({ children }: { children: React.ReactNode }) {
  const queryClient = getQueryClient(); // React Query cache for this browser/server context.
  const [trpcClient] = useState(() => // useState keeps one tRPC client per mounted provider.
    createTRPCClient<AppRouter>({
      links: [
        loggerLink({ /* dev/error logging only */ }),
        endingLink({
          headers: {
            "x-trpc-source": "client", // route handler rejects requests without this convention header.
          },
        }),
      ],
    }),
  );

  return (
    <QueryClientProvider client={queryClient}> // makes useQuery/useMutation work below.
      <TRPCProvider trpcClient={trpcClient} queryClient={queryClient}>
        {children}
      </TRPCProvider>
    </QueryClientProvider>
  );
}
```

Backend tRPC route (`apps/status-page/src/app/api/trpc/edge/[trpc]/route.ts:11-24`):

```ts
const handler = (req: NextRequest) => {
  const blocked = guardTRPCSource(req); // cheap convention filter; real auth lives in procedures.
  if (blocked) return blocked;

  return fetchRequestHandler({
    endpoint: "/api/trpc/edge", // URL surface exposed by the Next route.
    router: edgeRouter, // router table from packages/api/src/edge.ts.
    req: req,
    createContext: () => createTRPCContext({ req, auth }), // attaches session, db, request metadata.
    onError: createOnError("edge"),
  });
};
```

## TypeScript Orientation

- `AppRouter` is inferred from merged routers, then used by tRPC clients for typed procedure calls (`packages/api/src/root.ts:1-6`; `apps/dashboard/src/lib/trpc/client.tsx:9-12`).
- Zod schemas define runtime input validation and TypeScript types; `CreateStatusReportInput` is both a parser and an inferred type (`packages/services/src/status-report/schemas.ts:12-20`).
- Drizzle/Zod select schemas turn table definitions into row types; `StatusReport` and `StatusReportUpdate` come from `selectStatusReportSchema` and `selectStatusReportUpdateSchema` (`packages/db/src/schema/status_reports/validation.ts:38-55`).
- Literal enum arrays keep UI and backend values synchronized; the status-report statuses live in the DB schema (`packages/db/src/schema/status_reports/status_reports.ts:8-13`) and the form imports them (`apps/dashboard/src/components/forms/status-report/form.tsx:17-21`; `apps/dashboard/src/components/forms/status-report/form.tsx:62-69`).

## React Component Anatomies

Simple component: `StatusBlankTitle` (`packages/ui/src/components/blocks/status-blank.tsx:63-73`):

```tsx
export function StatusBlankTitle({
  children, // caller supplies the visible title.
  className, // caller may add styling without replacing defaults.
  ...props // normal div props pass through.
}: React.ComponentProps<"div">) {
  return (
    <div className={cn("font-medium", className)} {...props}>
      {children}
    </div>
  );
}
```

Intermediate component: `FormSheetStatusReport` (`apps/dashboard/src/components/forms/status-report/sheet.tsx:22-74`):

```tsx
export function FormSheetStatusReport({ children, defaultValues, onSubmit, items, warning }: Props) {
  const [open, setOpen] = useState(false); // sheet visibility is local UI state.

  return (
    <FormSheetWithDirtyProtection open={open} onOpenChange={setOpen}>
      <FormSheetTrigger asChild>{children}</FormSheetTrigger> // caller controls the button.
      <FormSheetContent className="sm:max-w-lg">
        {warning ? <p className="px-4 py-4 text-sm text-warning">{warning}</p> : null}
        <FormStatusReport
          id="status-report-form"
          onSubmit={async (values) => {
            await onSubmit(values); // delegates actual mutation to the page.
            setOpen(false); // closes only after save succeeds.
          }}
          defaultValues={defaultValues}
          items={items}
        />
        <Button type="submit" form="status-report-form">Submit</Button>
      </FormSheetContent>
    </FormSheetWithDirtyProtection>
  );
}
```

Complex component: dashboard status reports page (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:23-128`):

```tsx
export default function Page() {
  const { id } = useParams<{ id: string }>(); // route segment [id] becomes a string param.
  const trpc = useTRPC();
  const queryClient = useQueryClient();

  const { data: page } = useQuery(trpc.page.get.queryOptions({ id: Number.parseInt(id) }));
  const { data: statusReports, refetch } = useQuery(
    trpc.statusReport.list.queryOptions({ pageId: Number.parseInt(id) }),
  );

  const createStatusReportMutation = useMutation(
    trpc.statusReport.create.mutationOptions({
      onSuccess: async (statusReport) => {
        if (statusReport.notifySubscribers) {
          await sendStatusReportUpdateMutation.mutateAsync({ id: statusReport.id });
        }
        refetch(); // refreshes the report list after creation.
        queryClient.invalidateQueries({ queryKey: trpc.page.list.queryKey() }); // updates sidebar/list summaries.
      },
    }),
  );

  if (!statusReports || !page) return null; // no loading UI yet.
  return <DataTable columns={columns} data={statusReports} />;
}
```

## Backend Route/Controller Anatomies

Simple route: `statusReport.get` (`packages/api/src/router/statusReport.ts:175-187`):

```ts
get: protectedProcedure
  .input(z.object({ id: z.number() })) // runtime boundary: client must send numeric id.
  .query(async ({ ctx, input }) => {
    try {
      const result = await getStatusReport({
        ctx: toServiceCtx(ctx), // turns tRPC user/workspace into service actor context.
        input: { id: input.id },
      });
      return narrowPage(result); // guards dashboard assumption that every report has a page.
    } catch (err) {
      toTRPCError(err); // maps service errors to tRPC error codes.
    }
  }),
```

Create route: `statusReport.create` (`packages/api/src/router/statusReport.ts:58-81`):

```ts
create: protectedProcedure
  .meta({ track: Events.CreateReport }) // analytics runs after successful request in middleware.
  .input(createStatusReportTRPCInput) // preserves dashboard wire shape.
  .mutation(async ({ ctx, input }) => {
    const { initialUpdate } = await createStatusReport({
      ctx: toServiceCtx(ctx),
      input: {
        title: input.title,
        status: input.status,
        pageId: input.pageId,
        pageComponentIds: input.pageComponents, // adapts legacy name to service name.
        date: input.date,
        message: input.message,
      },
    });
    return { ...initialUpdate, notifySubscribers: input.notifySubscribers };
  }),
```

Public route: `statusPage.get` (`packages/api/src/router/statusPage.ts:65-113`):

```ts
get: publicProcedure
  .input(z.object({ slug: z.string().toLowerCase(), cardType: z.enum([...]).nullish(), barType: z.enum([...]).nullish() }))
  .output(selectPublicPageSchemaWithRelation.nullish())
  .query(async (opts) => {
    if (!opts.input.slug) return null;
    const _page = await opts.ctx.db.query.page.findFirst({
      where: sql`lower(${page.slug}) = ${opts.input.slug} OR lower(${page.customDomain}) = ${opts.input.slug}`,
      with: {
        workspace: true,
        statusReports: { with: { statusReportUpdates: { orderBy: (reports, { desc }) => desc(reports.date) } } },
        maintenances: { with: { maintenancesToPageComponents: { with: { pageComponent: true } } } },
        pageComponents: { with: { monitor: { with: { incidents: true } }, group: true } },
      },
    });
  });
```

## Domain Glossary

- Workspace: tenant boundary for dashboard users and resources; tRPC resolves it from the `workspace-slug` cookie (`packages/api/src/trpc.ts:171-200`).
- Page/status page: public communication surface with slug, custom domain, access settings, components, subscribers, reports, and maintenances (`packages/db/src/schema/pages/page.ts:13-96`).
- Page component: item shown on a status page; can represent a monitor or static component (`packages/db/src/schema/page_components/page_components.ts:20-63`).
- Status report: incident-style communication object with a title and current status (`packages/db/src/schema/status_reports/status_reports.ts:15-39`).
- Status report update: timeline entry attached to a status report; it stores status, date, and message (`packages/db/src/schema/status_reports/status_reports.ts:41-63`).
- Status report to page component: join table connecting a report to affected page components (`packages/db/src/schema/page_components/page_components.ts:124-157`).
- Monitor: uptime-checking resource shown through page components; public status-page code treats monitor components differently from static components (`packages/api/src/router/statusPage.ts:134-180`).
- Open event: computed public event still active now; public UI uses it to render banners (`packages/api/src/router/statusPage.ts:249-266`; `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:143-272`).
- Last event: computed recent feed event; public UI filters status reports and maintenances against it (`packages/api/src/router/statusPage.ts:239-247`; `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:329-360`).
- Subscriber notification: optional side effect after creating a report; dashboard currently triggers email send after create success (`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:33-51`).

## Junior Socratic Checkpoint

1. What command starts dashboard dev mode, and where is it defined?
2. Why does `useTRPC()` not fetch anything by itself?
3. What is the difference between a status report and a status report update?
4. Which file decides whether `statusReport.create` goes to edge or lambda?
5. Why does the create form check `"date" in values` before mutating?
6. Where is workspace ownership checked during status-report creation?
7. How does a public status page know which reports are visible in the feed?

How to self-grade:

- Strong answers cite `package.json:15-17`, `apps/dashboard/src/lib/trpc/client.tsx:11-12`, `packages/db/src/schema/status_reports/status_reports.ts:15-63`, `apps/dashboard/src/lib/trpc/shared.ts:38-76`, `apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:89-103`, `packages/services/src/status-report/create.ts:32-46`, and `apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:329-360`.
- Strong answers describe boundaries: UI state, HTTP/tRPC, service context, database rows, public rendering.
- Weak answers only name files without explaining what each file owns.

