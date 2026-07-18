# 01 — How React Actually Renders

If you only learn one thing about React, learn this. Most "weird React behavior" is predictable once you understand the render model. Most React code is *worse* than it needs to be because it's written by people who don't.

## The mental model in five sentences

1. A **component is a function** that returns a description of what to render (a tree of elements).
2. React calls that function when it thinks something might have changed (state, props, context, parent re-rendered).
3. The returned tree is **compared with the previous tree** (reconciliation).
4. The minimum DOM changes needed to make the live DOM match are **committed**.
5. After commit, **effects run** (in order: cleanup of old effects, then new effects, in roughly child-first order).

That's it. Internalize those five sentences and 80% of React's weirdness disappears.

## Render is not the same as commit

A render is "React called your function and got a tree back."

A commit is "React updated the DOM and ran effects."

Many renders are followed by **zero** commits — when the new tree matches the old, nothing changes. So a `console.log("render")` in the component body firing doesn't mean the user saw anything new.

## What triggers a render

- State change in *this* component (`setState`).
- A re-render of the parent (regardless of whether props changed).
- A context value this component subscribes to changed.

By default, **a parent re-render forces a child re-render** — even if the child's props are reference-equal. To opt out, wrap the child in `React.memo`. To make `memo` actually work, you need *stable* prop references (which is why `useMemo` and `useCallback` exist).

## Reconciliation and keys

When React diffs two trees, it pairs elements by **position + type**. If you render a list `[A, B, C]` and then `[A, C]`, React doesn't realize B was removed — it sees `A→A`, `B→C` (mutated B into C), `C→` (removed C). The DOM state of `B` is now attached to `C`.

That's why list children need a **stable, unique `key`**:

```jsx
{items.map((item) => <Row key={item.id} item={item} />)}
```

The key tells React "this is the same logical element across renders." Don't use array index as a key unless the list never reorders, inserts, or deletes — even then, prefer a real ID.

OpenStatus follows this rigorously. Look at any `.map(...)` in the dashboard — they all key on the row's `id`.

## Why hooks have rules

Hooks store state per component **by call order**. The first `useState` in your component is "slot 0." The second is "slot 1." React uses the *order* of hook calls to match up with the *order* of stored slots.

That's why you can't call hooks conditionally:

```jsx
function Bad() {
  if (someCondition) {
    const [x] = useState(0);   // ❌ first slot only when condition is true
  }
  const [y] = useState(0);      // sometimes slot 0, sometimes slot 1
}
```

Don't. Always call the same hooks in the same order. If you need conditional behavior, *pass a value* to the hook (or short-circuit *inside* an effect's body), don't conditionally call the hook itself.

## Strict Mode in dev: components mount twice

In development, React's Strict Mode double-invokes component bodies and effects on mount. This isn't a bug — it's catching impure renders and effect-cleanup bugs. If your code breaks under Strict Mode, your code has a bug.

- **Render functions must be pure.** Same props/state in, same output out, no side effects.
- **Effects must clean up.** Anything you start in an effect (a subscription, an interval, a fetch in flight) must be torn down by the returned cleanup function. Strict Mode runs `setup → cleanup → setup` to catch leaks.

## The Suspense and concurrency story (in one paragraph)

React 18+ can pause rendering and resume it later. A component that throws a promise (e.g., from `use()` or React Query's `useSuspenseQuery`) is "suspended" — React shows the nearest `<Suspense fallback>` until the promise resolves. Concurrent features (transitions, deferred values) let React keep the UI interactive while expensive renders happen in the background. This is mostly invisible until you opt in, but you'll see `useTransition` in OpenStatus's form submit ([`apps/dashboard/src/components/forms/status-report/form.tsx:108-137`](../../apps/dashboard/src/components/forms/status-report/form.tsx#L108-L137)) — it marks the submit work as non-blocking so the form stays responsive.

## What triggers a *commit*

After a render produces a new tree, React diffs. If anything changed:

1. Run cleanup of effects whose dependencies changed.
2. Commit DOM changes synchronously.
3. After paint, run new/updated effects.

The "after paint" detail matters: `useEffect` runs *after* the browser paints. `useLayoutEffect` runs *before* paint — use it when you need to measure or adjust the DOM before the user sees anything (rare; almost always `useEffect` is correct).

## The actual mental model for re-renders

When you wonder "why did this re-render?", check in order:

1. Did its own state change? (Read `useState` calls and `useReducer`.)
2. Did its parent re-render? (Walk up the tree.)
3. Did a context value it subscribes to change? (Find the `useContext` and check the provider.)
4. Did `React.memo` fail because of a new reference prop? (Inspect props in DevTools.)

If none of those apply, you didn't actually re-render — you re-rendered the component above and your code happens to log on every parent commit.

**Senior aside:** *Most "performance problems" are actually re-render problems. Most re-render problems are state living in the wrong component. Don't reach for `memo`/`useMemo`/`useCallback` first; move the state down or split the component.*

## A worked example from this repo

Look at the status reports page [`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:23-128`](../../apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx#L23-L128).

On first render: `useParams` returns the URL id, `useTRPC`/`useQueryClient` return stable refs, `useQuery` returns `{ data: undefined, ... }` (cache miss), `useMutation` returns a stable mutation object.

React renders, sees `!statusReports || !page`, returns `null`. No DOM commit beyond `null`.

React Query fetches in the background, populates the cache. When the fetch resolves, it notifies subscribers. The component re-renders. Now `statusReports` and `page` exist. The full UI renders.

User clicks Create → `FormSheetStatusReport` opens (local `useState` inside the sheet flips). On submit, `createStatusReportMutation.mutateAsync` runs. The mutation's `onSuccess` fires `refetch()` and `queryClient.invalidateQueries({ queryKey: trpc.page.list.queryKey() })`. The first re-fetches *this* list (because `refetch` is bound to the query options). The second marks the page-list cache as stale; any mounted consumer (e.g., the sidebar) refetches automatically.

That single function is doing four things: a React re-render trigger, an HTTP request, an in-memory cache update, and a cross-page cache invalidation. The render model is what lets it stay coordinated.

## Pitfall checklist

1. **Setting state inside render.** Creates an infinite loop. State changes belong in events or effects.
2. **Side effects in render.** `localStorage.setItem(...)` in the component body fires on every render, including Strict Mode's double-mount. Move it to an effect.
3. **Stable references for object/array props to memoized children.** A new `{}` every render defeats `memo`.
4. **Using array index as `key`.** Breaks animations, focus, and form state when the list reorders.
5. **Reading state inside `setState` callback's closure** instead of using the updater form. Stale.
6. **Updating state in a `useEffect` without a dependency that gates it.** Infinite loop.
7. **Hook calls inside a condition or loop.** Violates the rules of hooks.
8. **Missing cleanup in an effect.** Subscription leaks across re-mounts; `setInterval` runs forever.
9. **Mutating props or state in place.** No re-render, no warning, just silent bugs.
10. **Reaching for `React.memo` to fix correctness.** If your app *needs* memo to be correct, the data flow is wrong.

## Drills

- Add a `console.log("render")` to the top of [`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx`](../../apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx). Click around. Count renders. Predict each one before it happens.
- Find a `.map(...)` rendering React elements without a stable `key`. (Hint: there shouldn't be one — the repo is clean here. But predict what would break if you removed an existing key.)
- Open React DevTools' Profiler. Record a session: load the dashboard, open a status page, open a status report. Identify the single most expensive render. What state or prop change triggered it?
- Explain in writing why `if (!statusReports || !page) return null;` is *correct* but produces a worse user experience than rendering loading/error states. Then fix it without breaking the data flow.
