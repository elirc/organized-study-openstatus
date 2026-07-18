# 03 — State Architecture

The defining skill of a senior React engineer is putting each piece of state in *one* right place. Get this wrong and you'll fight bugs forever; get it right and your code is half the size.

## The five homes of state

| Home | What lives there | Examples |
| --- | --- | --- |
| **Local** | UI state used by one component | dropdown open/closed, hover, form field text |
| **Lifted** | Shared by a small subtree | selected row, active tab, modal open state passed down |
| **URL** | Anything a user should be able to bookmark, share, or back-button | filters, search query, current page, selected entity |
| **Server** | The source of truth — DB-backed data | status reports list, monitor details, user workspace |
| **Derived** | Computed from any of the above | "report has updates" — never store, always compute |

If your state lives in the wrong home, you'll work around it for the rest of the codebase's life.

## Local — `useState` in the component that owns it

```jsx
function FormSheet() {
  const [open, setOpen] = useState(false);
  return <Sheet open={open} onOpenChange={setOpen}>...</Sheet>;
}
```

If only this component cares whether the sheet is open, keep it local. Lifting it up "in case someone else needs it later" is YAGNI.

Real example: [`apps/dashboard/src/components/forms/status-report/sheet.tsx:22-74`](../../apps/dashboard/src/components/forms/status-report/sheet.tsx#L22-L74) keeps the `open` state local to the sheet because only the sheet's trigger and content care.

## Lifted — moved up to the nearest common ancestor

When two siblings need the same state, the parent owns it.

```jsx
function Parent() {
  const [selected, setSelected] = useState<Id | null>(null);
  return (
    <>
      <List selected={selected} onSelect={setSelected} />
      <Detail id={selected} />
    </>
  );
}
```

Don't go higher than necessary. If `Parent` is the smallest component that contains both consumers, that's where state goes — not in the page or app root.

## URL — for shareable, navigation-respecting state

If a user could reasonably want to share a link, refresh, or hit back/forward and have the same view, the state belongs in the URL. OpenStatus uses [`nuqs`](https://nuqs.dev/) for typed URL search params.

From [`apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx:30, 47`](../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L30-L47):

```tsx
import { useQueryStates } from "nuqs";
import { searchParamsParsers } from "./search-params";

const [searchParams, setSearchParams] = useQueryStates(searchParamsParsers);
```

Now the user's filter selections survive a refresh and become shareable. The cost is one round of plumbing; the benefit is permanent.

**Senior aside:** *If you find yourself syncing local state to the URL via `useEffect`, you're reinventing a URL state library badly. Use the library; it handles serialization, multiple back-button navigations, and SSR hydration for you.*

## Server — owned by React Query (via tRPC here)

The data that lives in your database is **server state**. It's special because:

- It can become stale (someone else modified it).
- It's expensive to fetch (network + DB).
- Multiple components need the same data at the same time.
- You need to *invalidate* it when you mutate.

React Query exists to solve those four problems. You give it a *query key*, a *fetch function*, and configuration; it gives you a cache, deduplication, stale-while-revalidate, retries, and invalidation.

OpenStatus's status reports page demonstrates the full lifecycle ([`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:30-52`](../../apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx#L30-L52)):

```tsx
const { data: statusReports, refetch } = useQuery(
  trpc.statusReport.list.queryOptions({ pageId: Number.parseInt(id) }),
);

const createStatusReportMutation = useMutation(
  trpc.statusReport.create.mutationOptions({
    onSuccess: async (statusReport) => {
      if (statusReport.notifySubscribers) {
        await sendStatusReportUpdateMutation.mutateAsync({ id: statusReport.id });
      }
      refetch();
      queryClient.invalidateQueries({ queryKey: trpc.page.list.queryKey() });
    },
  }),
);
```

Three things happen on success:
1. A side-effect mutation (the email send).
2. `refetch()` — re-runs *this* query.
3. `invalidateQueries({ queryKey: trpc.page.list.queryKey() })` — marks the page-list cache as stale across the whole app; any mounted consumer refetches.

You don't need to manually update the cache or push events around — the cache is the source of truth.

We'll go deeper on React Query in [`06-react-query-and-trpc.md`](06-react-query-and-trpc.md).

## Derived — compute, don't store

If you can compute it, compute it. Don't store it.

```jsx
// ❌
const [count, setCount] = useState(items.length);
useEffect(() => setCount(items.length), [items]);

// ✅
const count = items.length;
```

Storing derived state introduces a class of bugs: "the derived value didn't update when its source did." OpenStatus does this right almost everywhere; one example is `hasUnresolvedIssue` ([`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:56-58`](../../apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx#L56-L58)):

```tsx
const hasUnresolvedIssue = statusReports.some(
  (report) => report.status !== "resolved",
);
```

Computed at render time from the source. No effect, no storage, no sync bugs possible.

If the derivation is expensive, wrap in `useMemo`. If it's not, don't bother.

## State machines — for complex flows

When state has more than ~5 valid combinations and clear transitions, reach for a state machine (XState, or a hand-rolled discriminated union + reducer). OpenStatus doesn't use one here, but a feature like "incident workflow" (investigating → identified → monitoring → resolved) is a perfect candidate.

```ts
type IncidentState =
  | { status: "draft" }
  | { status: "open"; reportedAt: Date }
  | { status: "resolved"; reportedAt: Date; resolvedAt: Date };
```

Encoding invalid states as *unrepresentable* (you can't have `resolvedAt` without `reportedAt`) eliminates whole classes of bugs.

## Form state — its own animal

Form state has special needs: validation, dirty tracking, async submission, field-level errors. Don't roll your own; use react-hook-form (which OpenStatus does). Cover this in [`04-forms-and-validation.md`](04-forms-and-validation.md).

## How to decide where state goes — the questions

1. **Who reads it?** If only one component, local. If two siblings, lifted to the parent. If many components in different subtrees, context or a store.
2. **Should it survive refresh?** Yes → URL or server. No → local/lifted.
3. **Is the DB the source of truth?** Yes → React Query, not `useState`.
4. **Is it computable?** Yes → compute, don't store.
5. **Will users share or link to this state?** Yes → URL.

Walk those five questions on every piece of state you add. Most state lives in the wrong home not because the developer didn't know better but because they never asked.

## Context — used carefully

Context is a transport mechanism, not a state store. It re-renders all subscribers when its value changes. That makes it great for:

- Theme, locale, auth user — values that change rarely.
- Stable references (the tRPC client, a query client).

It's bad for:
- High-frequency updates (mouse position, scroll, input every keystroke).
- Large objects where consumers only care about a slice.

If you need fine-grained subscriptions, use a store library (Zustand, Jotai, Redux Toolkit) — or split your context into multiple contexts.

## The "you might not need state" check

Before adding `useState`, ask: can I compute this from props or other state? Can it live in the URL? Is this just a CSS thing (hover, focus) the browser handles? Most "state" added to React components shouldn't be there.

## Pitfall checklist

1. **Server state in `useState`.** You lose caching, deduplication, invalidation. Use React Query.
2. **Lifted state too high.** A modal's `open` flag belongs in the modal, not the app root.
3. **Lifted state too low.** Two siblings reading + writing → state goes to their parent.
4. **Derived state stored and synced.** Always a bug source.
5. **Form state in `useState`** for non-trivial forms. Use react-hook-form.
6. **URL state forgotten.** Refresh loses the user's filters → the filters belonged in the URL.
7. **One giant context.** Updating any field re-renders every consumer.
8. **`useState` for a value that needs to drive `useEffect` deps that fire fetches.** That's server state — let React Query do it.
9. **Updating state in render** ("just this once") triggering a render loop.
10. **`prop → state` mirror.** `useState(props.x)` reads `props.x` once; later changes don't sync. Either use `props.x` directly or `key` the component to remount.

## Drills

- Find a piece of `useState` in the codebase. Decide if it should live somewhere else (URL? server? lifted?). Write your reasoning.
- Refactor the status reports filter (if/when added) to use `useQueryStates` so it survives navigation.
- Pick a flow with 4+ states (e.g., status report workflow). Sketch a discriminated union that makes invalid combinations unrepresentable.
- Remove `hasUnresolvedIssue` and reintroduce it as stored state with a `useEffect`. Then introduce a bug by forgetting to sync. Revert. Notice how much you had to write to recreate a bug the original avoided entirely.
- Find the largest `useContext` consumer in this repo. Predict what would happen if its provider value changed once per second. (You don't have to verify — just predict.)
