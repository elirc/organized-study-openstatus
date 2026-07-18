# 02 — Exercises

## Rendering and reconciliation

1. Make a component with `console.log("render")` at the top. Add three children: one with stable props, one with a new `{}` prop on every render, one wrapped in `React.memo`. Make the parent re-render. Predict and verify which children render.
2. Render a list of three items with `key={index}`. Add an item at position 0. Inspect the DOM (or use DevTools). Which DOM nodes were updated? Now switch to `key={item.id}` and repeat. Difference?
3. Open React DevTools' Profiler. Record loading the dashboard's status reports page. Identify the most expensive render. Could it have been avoided?

## Hooks

4. Build a counter with `setInterval` that increments once per second. Use `setCount(count + 1)` and verify the bug (always 0 + 1). Fix it three ways (updater, dep array, ref).
5. Write `useDebounce(value, delay)`. Test by typing fast in a sandbox input.
6. Write `usePrevious(value)`. (Hint: `useRef`.)
7. Find a `useEffect` in this repo that's missing a dep or has an `eslint-disable`. Decide: is it correct, lazy, or wrong?
8. Replace a `useEffect`-driven derived value with a direct compute. Verify the behavior matches.

## State architecture

9. Inventory the state in [`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx`](../../apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx). For each piece, name its home (local, lifted, URL, server, derived). Could any be moved?
10. Add a filter to the status reports list. Decide where its state lives. Implement it. Then move it to the URL via `nuqs`. Notice the additional benefit.
11. Find a piece of derived state that's stored. (Hint: probably nowhere in this repo — the team is disciplined. Make one up and immediately remove it.)

## Forms

12. Read [`apps/dashboard/src/components/forms/status-report/form.tsx`](../../apps/dashboard/src/components/forms/status-report/form.tsx). Identify: schema, resolver, default values, watched fields, dirty tracking, submission, async pending state. Could anything be simpler?
13. Add a synthetic server-side error (throw a `CONFLICT` from the router). Verify the form receives it. Display it as a field-level error using `form.setError`.
14. Convert a single field from controlled to uncontrolled. What broke and why? Revert.
15. Build a "you have unsaved changes" warning that intercepts back navigation when `form.formState.isDirty`. Compare your design with what the repo does via `useFormSheetDirty`.

## RSC and data flow

16. Walk the call sequence: user navigates to `/status-pages/3/status-reports`. List every server function and every client function in order. What data is fetched server-side vs client-side?
17. Find a server component that *could* be a client component. Convert it (just locally; revert after). What changed in the network panel?
18. Cause a hydration mismatch on purpose by rendering `new Date().toLocaleString()` in a server component. Watch the console warning. Fix it without changing the desired behavior.
19. Open [`apps/dashboard/src/lib/trpc/server.tsx`](../../apps/dashboard/src/lib/trpc/server.tsx). Why is `getQueryClient = cache(makeQueryClient)`? What would break with a plain factory?

## React Query / tRPC

20. After creating a status report, list every cache key that *could* be affected. Compare with what the page actually invalidates. Is there a case where the UI shows stale data?
21. Manually invalidate a query from a different component. Verify with React Query DevTools.
22. Add `staleTime: 0` to one query, then `staleTime: Infinity` to another. Click around. Describe the difference in network behavior.
23. Implement an optimistic update for one mutation in this repo. Include `onMutate`, `onError` revert, and `onSettled` invalidation.
24. Read [`apps/dashboard/src/lib/trpc/shared.ts:38-77`](../../apps/dashboard/src/lib/trpc/shared.ts#L38-L77). Walk through what happens when you call `trpc.statusReport.create.mutate(...)`. What if you renamed `statusReport` to `report` on the server but not in the client `lambdas` list?

## Self-grading

You're senior-shaped on React when:
- You can predict re-renders before they happen.
- You instinctively reach for `useMemo`/`useCallback` only when there's a concrete reason.
- You can identify the home of any piece of state in 5 seconds.
- You can debug a "stale data after mutation" bug by checking cache keys, not by adding `window.location.reload()`.
- You know which one of `useEffect`, `useLayoutEffect`, `useRef`, `useState` to reach for without thinking — and which to *avoid* by computing in render.
