# 04 — Forms and Validation

Forms are where junior code looks worst and where senior code looks cleanest. The difference is almost entirely about *not* reinventing what react-hook-form + Zod give you for free.

## The four problems a form library solves

1. **Per-field state** without re-rendering the whole form on every keystroke.
2. **Validation** synchronously and asynchronously, with field-level errors.
3. **Submission** with loading, error, and success states.
4. **Dirty tracking** ("you have unsaved changes").

Solving these by hand with `useState` per field is possible but expensive (re-renders the world on every keystroke), bug-prone, and verbose.

OpenStatus uses **react-hook-form** (rhf) for state and **Zod** for validation. The `@hookform/resolvers/zod` adapter bridges them.

## A real form, walked through

From [`apps/dashboard/src/components/forms/status-report/form.tsx:62-145`](../../apps/dashboard/src/components/forms/status-report/form.tsx#L62-L145):

```tsx
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

export type FormValues = z.infer<typeof schema> | z.infer<typeof updateSchema>;

export function FormStatusReport({ defaultValues, onSubmit, items, ... }) {
  const form = useForm<FormValues>({
    resolver: zodResolver(defaultValues ? updateSchema : schema),
    defaultValues: defaultValues ?? {
      status: "investigating",
      title: "",
      message: "",
      date: new Date(),
      pageComponents: [],
      notifySubscribers: true,
    },
  });

  function submitAction(values: FormValues) {
    if (isPending) return;
    startTransition(async () => {
      try {
        const promise = onSubmit(values);
        toast.promise(promise, { loading: "Saving...", success: "Saved", error: ... });
        await promise;
      } catch (error) { console.error(error); }
    });
  }

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(submitAction)}>
        {/* fields */}
      </form>
    </Form>
  );
}
```

Notes:

1. **Two schemas, one form.** `schema` is the "create" shape; `updateSchema = schema.omit({...})` is the "update" shape. Same field rules where they overlap. `FormValues` is the union — and the form picks the right resolver based on whether `defaultValues` exists (create vs edit). Zod's `.omit` is the lever; you don't repeat field definitions.

2. **`zodResolver`** runs the schema on submit (or on change, depending on config). Errors come back as field-level objects; rhf displays them via `<FormMessage />`.

3. **`defaultValues`** are *initial* values — change them after mount and rhf doesn't pick that up unless you call `form.reset()`. This is why edit flows pass `defaultValues={existingRecord}` once.

4. **`handleSubmit`** wraps your submit with validation, error collection, and submit-state management. It only calls your handler if validation passes.

5. **`useTransition`** wraps the async submit so the UI stays responsive while the network call flies.

6. **`toast.promise`** is a UX nicety — show loading, success, error toasts off a single promise.

## Field registration: `Controller` vs uncontrolled

react-hook-form supports both:

- **Uncontrolled** (`register("name")`) — rhf reads input values via refs, native browser inputs only.
- **Controlled** (`<Controller>` / `<FormField render={({ field }) => ...}>`) — rhf passes `value` and `onChange` to your component. Required for shadcn/ui inputs, custom components, anything that isn't a raw `<input>`.

OpenStatus uses the controlled form via shadcn/ui's `<FormField>` wrapper. From the form file, the title field:

```tsx
<FormField
  control={form.control}
  name="title"
  render={({ field }) => (
    <FormItem>
      <FormLabel>Title</FormLabel>
      <FormControl>
        <Input placeholder="Database connection issues" {...field} />
      </FormControl>
      <FormMessage />
    </FormItem>
  )}
/>
```

`{...field}` spreads `value`, `onChange`, `onBlur`, `name`, `ref` — the entire controlled-input contract.

## Validation — the schema as documentation

The Zod schema is the *only* source of truth for what a valid input is:

- Form: `zodResolver(schema)` validates pre-submit.
- tRPC router: `.input(schema)` validates at the API boundary.
- Service: `Schema.parse(input)` validates at the service boundary.
- DB: column constraints (NOT NULL, enum) validate at the row level.

Notice the pattern from [`packages/api/src/router/statusReport.ts:24-32`](../../packages/api/src/router/statusReport.ts#L24-L32) and [`packages/services/src/status-report/schemas.ts`](../../packages/services/src/status-report/schemas.ts) — the form, router, and service all import the canonical enum from the DB schema. One source. No drift.

**Senior aside:** *"Validation at the boundary" is a layered defense. If the form validation is bypassed (browser console, curl, attacker), the router catches it. If somehow the router is bypassed, the service catches it. If somehow the service is bypassed, the DB constraint catches it. Don't trust any one layer.*

## Submission patterns

```tsx
async function submit(values: FormValues) {
  try {
    await mutation.mutateAsync(values);
    onSuccess?.();
  } catch (err) {
    if (isTRPCClientError(err) && err.data?.code === "CONFLICT") {
      form.setError("title", { message: "Already taken" });
    } else {
      toast.error("Something went wrong");
    }
  }
}
```

Three patterns to internalize:

1. **Use `mutateAsync`** when you need to await inside an async handler. Use `mutate` when you'd handle results in `onSuccess`/`onError` callbacks.
2. **Server validation errors → form errors.** `form.setError(field, { message })` displays the error inline. Don't dump them all in a toast.
3. **Disable double-submit.** rhf's `formState.isSubmitting` (or `useTransition`'s `isPending`) tells you if a submission is in flight.

## Dirty tracking and unsaved-changes guards

```tsx
const formIsDirty = form.formState.isDirty;
useEffect(() => { setIsDirty(formIsDirty); }, [formIsDirty, setIsDirty]);
```

That's straight from OpenStatus ([`apps/dashboard/src/components/forms/status-report/form.tsx:107-114`](../../apps/dashboard/src/components/forms/status-report/form.tsx#L107-L114)). The form notifies its parent sheet via a context (`useFormSheetDirty`) so the sheet can prompt "you have unsaved changes" when the user tries to close it.

Pattern: when state about *me* needs to influence *my parent's* behavior, surface it through a callback or context — not by writing through a ref or other backdoor.

## Async validation

Zod's `.refine` lets you validate against any predicate, sync or async:

```ts
const schema = z.object({
  slug: z.string().refine(
    async (slug) => !(await isSlugTaken(slug)),
    "Slug already taken",
  ),
});
```

But async refinements run on every validation pass, which can hammer your API. Debounce on the field level, or only run async checks on submit.

## Controlled vs uncontrolled performance

Per-keystroke re-renders are the #1 form performance problem. rhf is fast by default because each field subscribes only to its own slice of state. If you reach `form.watch()` over a field, *that* component re-renders per keystroke. Use it deliberately — a long form watching every field will feel slow.

```tsx
const watchMessage = form.watch("message");   // re-renders this component on every keystroke
```

That's fine for a small section near the message preview, terrible if used to gate rendering of the whole form.

## Default values gotcha

```tsx
const form = useForm({ defaultValues: props.values });   // ❌
```

If `props.values` is fetched async and not present on first render, the form initializes with `undefined`. When the prop arrives, rhf does **not** re-initialize. Fix:

```tsx
useEffect(() => {
  if (props.values) form.reset(props.values);
}, [props.values]);
```

Or pass a `key` to remount the form when the underlying record changes.

## Pitfall checklist

1. **Per-field `useState`** instead of rhf. Re-renders the world.
2. **Schema duplicated** between form and API. Drift guaranteed.
3. **Server errors swallowed.** Surface them as field errors, not just toasts.
4. **Double-submit allowed.** Disable submit while pending.
5. **`defaultValues` updated** without `form.reset`. Form ignores you.
6. **Validation only client-side.** Always validate server-side. Always.
7. **Async refinements running every keystroke.** Debounce, or run only on submit.
8. **`form.watch()` at the top level** for a field that's deep in the tree. Subscribe locally.
9. **Date inputs** that don't account for timezone. The form sees a local date; the server sees a UTC instant. Be explicit.
10. **Confirm step bypassed** for destructive actions. A `<form>` with a delete button isn't enough; require typed confirmation.

## Drills

- Read [`apps/dashboard/src/components/forms/status-report/form.tsx`](../../apps/dashboard/src/components/forms/status-report/form.tsx) end to end. Identify: schema, resolver, defaults, watched fields, dirty tracking, submit handling, async pending state. Take notes.
- Find one place where `form.watch()` is used. Predict the re-render impact. If you needed to optimize, what would you do?
- Add a synthetic async refinement to one field — e.g., "title must not equal an existing report's title." Debounce it. Then remove it.
- Pull the "create vs update schemas with shared base" pattern out into a generic. Try to abstract it for a different entity in the repo (e.g., a notification form). When does the abstraction help vs hurt?
- Build a tiny `useUnsavedChangesGuard(isDirty)` hook that intercepts back-navigation. Decide where it belongs: in this form, in the sheet wrapper, or in a shared hook. Justify your choice.
