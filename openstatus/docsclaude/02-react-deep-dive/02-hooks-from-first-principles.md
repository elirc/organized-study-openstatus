# 02 — Hooks From First Principles

Hooks aren't magic. They're functions that read and write per-component memory cells indexed by call order. Once you see them that way, you can derive their behavior instead of memorizing it.

## `useState` — a memory cell + a setter

```jsx
const [n, setN] = useState(0);
```

On first render, allocate a cell with `0`. On every later render, return the *current* cell value plus a setter. Calling the setter:

- with a new value: stores the value, schedules a re-render.
- with the same value (`Object.is` check): bails out — no re-render.
- with a function `setN(prev => prev + 1)`: passes the cell's current value into the function and stores the return.

**Use the updater form when** the new value depends on the previous one — *especially* inside effects, intervals, or async handlers. The plain form captures whatever value the closure saw at render time, which may be stale.

```jsx
// ❌ Stale closure
useEffect(() => {
  const id = setInterval(() => setN(n + 1), 1000);
  return () => clearInterval(id);
}, []);   // n captured once, always 0 + 1

// ✅
useEffect(() => {
  const id = setInterval(() => setN(prev => prev + 1), 1000);
  return () => clearInterval(id);
}, []);
```

## `useReducer` — `useState`'s grown-up sibling

When updates depend on the previous state in non-trivial ways, `useReducer` is cleaner:

```jsx
const [state, dispatch] = useReducer(reducer, initialState);
// dispatch({ type: "ADD", payload: 1 });
```

Same memory model — one cell, a setter (`dispatch`) that runs your reducer. Use when:
- State has a few related fields that update together.
- Updates need to be testable independently (extract `reducer` and unit-test it).
- You want to pass `dispatch` deep (it's stable, unlike `setState`'s setter which is also stable but harder to type for complex shapes).

## `useEffect` — synchronization after commit

```jsx
useEffect(() => {
  // setup
  return () => {
    // cleanup
  };
}, [deps]);
```

Mental model: "after commit, if any dep changed since last commit, run cleanup of the previous effect, then run setup." On unmount, run the latest cleanup.

The deps array isn't an optimization — it's a contract. Every value the effect *reads* (variables, props, state) should be in deps, *or* be a stable reference, *or* you must justify why staleness is intentional. The lint rule `react-hooks/exhaustive-deps` enforces this. Disable it only with a comment explaining why.

What effects are for:
- Synchronizing with **external** systems (DOM events, subscriptions, the network, browser APIs).
- Not: deriving state from other state. That's just a calculation in the render body.

```jsx
// ❌
const [filtered, setFiltered] = useState([]);
useEffect(() => setFiltered(items.filter(p)), [items]);

// ✅
const filtered = items.filter(p);   // recalc per render; cheap
```

Or, if filtering is expensive and items rarely changes:

```jsx
const filtered = useMemo(() => items.filter(p), [items]);
```

## `useRef` — a memory cell that doesn't trigger re-render

```jsx
const r = useRef(initialValue);
r.current = "new";    // no re-render
```

Two uses:
- **DOM access:** `<input ref={r} />` lets you read `r.current` to call `.focus()`, etc.
- **Mutable instance variables:** values that need to persist across renders but aren't part of the rendered output. Timeouts, the latest value of a prop, a stored callback.

Don't use refs to "fix" missing dep arrays — that's a code smell. Use refs when *you don't want a re-render*.

## `useMemo` and `useCallback` — same thing, different surface

```jsx
const value = useMemo(() => expensive(input), [input]);
const fn    = useCallback((x) => doIt(x, input), [input]);
// useCallback(fn, deps) === useMemo(() => fn, deps)
```

Both **memoize** a value across renders based on a deps array. Use them when:

1. The value is **expensive to compute** and you'd rather skip the cost (`useMemo`).
2. The value is passed as a **prop to a memoized child** or as a **dep to another hook** — preserving reference identity matters (`useCallback` / `useMemo` returning an object).

Don't sprinkle them everywhere "for performance." They have overhead too (deps comparison + storing the memoized value). The default is *no* memo; reach for it when you have a concrete reason.

## `useContext` — read a value from the nearest provider

```jsx
const value = useContext(MyContext);
```

Three things to know:

1. **Re-render trigger:** any context consumer re-renders when the provider's `value` reference changes. Pass a stable value (memoize the object) or split contexts so updates to one part don't re-render all consumers of the other.
2. **No partial subscriptions:** if you put `{ user, theme }` in one context, theme changes re-render `useContext`-readers that only care about `user`. Solutions: split into two contexts, or use a state library like Zustand for finer-grained updates.
3. **The provider is up the tree.** A `useContext` with no provider gives you the default value passed to `createContext`.

OpenStatus uses a tRPC context provider [`apps/dashboard/src/lib/trpc/client.tsx:11-12`](../../apps/dashboard/src/lib/trpc/client.tsx#L11-L12):

```ts
export const { TRPCProvider, useTRPC, useTRPCClient } =
  createTRPCContext<AppRouter>();
```

Every component that uses `useTRPC()` is a consumer. The provider value is stable (set once in `useState`), so consumers don't re-render when the provider re-renders unless the wrapped query subscription fires.

## `useTransition` and `useDeferredValue` — concurrency hooks

```jsx
const [isPending, startTransition] = useTransition();
startTransition(() => setHeavyState(x));
```

Marks the wrapped state update as *non-urgent*. React keeps the UI responsive (responding to other input) and processes the heavy update in the background. `isPending` tells you whether the transition is still going.

OpenStatus uses this on form submit [`apps/dashboard/src/components/forms/status-report/form.tsx:108-137`](../../apps/dashboard/src/components/forms/status-report/form.tsx#L108-L137) so the form stays interactive while the mutation flies.

`useDeferredValue(value)` returns a "lagged" version of a value — React updates it when it has spare time. Useful for big lists driven by a fast-changing input (e.g., a search box).

## Custom hooks — just functions that call hooks

```jsx
function useIsMobile() {
  const [isMobile, setIsMobile] = useState(false);
  useEffect(() => {
    const mql = window.matchMedia("(max-width: 768px)");
    setIsMobile(mql.matches);
    const handler = (e) => setIsMobile(e.matches);
    mql.addEventListener("change", handler);
    return () => mql.removeEventListener("change", handler);
  }, []);
  return isMobile;
}
```

(That's roughly what [`packages/ui/src/hooks/use-mobile.ts:7-23`](../../packages/ui/src/hooks/use-mobile.ts#L7-L23) does.)

Rules:
- Name starts with `use` (so the linter can apply the rules of hooks).
- Can call other hooks.
- Same call-order rule applies.
- Don't return JSX from a hook — that's a component.

Custom hooks are how you *deduplicate* logic without abstracting it badly. They're the right tool for "two components do the same thing with state."

## `useImperativeHandle` — for libraries, not apps

Lets a parent call methods on a child via `ref`. Almost always a sign that data flow is being inverted; prefer lifting state up. The exception: wrapping an imperative third-party widget (a video player, a map).

## `useSyncExternalStore` — for libraries integrating non-React state

If you're integrating Redux, Zustand, or any external store with React 18's concurrent rendering, this is the hook. App code rarely calls it directly.

## Effects that you don't need

Common "I need an effect" cases that you actually don't:

| Anti-pattern | Replace with |
| --- | --- |
| Derive state from props/state in an effect | Just compute it in render |
| Reset state when a prop changes | Pass a `key` prop to remount |
| Sync state to URL via `useEffect` + `router.push` | Use the URL state hook / library directly |
| Fetch data | Use React Query / tRPC |
| Adjust a parent's state when child renders | Lift state up, or use an event handler |

The official "you might not need an effect" rule of thumb: *if removing the effect doesn't break anything observable, remove it.*

## Pitfall checklist

1. **Stale closure** capturing render-time value. Use updater form, refs, or correct deps.
2. **Forgotten effect cleanup.** Subscriptions, intervals, abort controllers.
3. **Lying dep arrays.** `// eslint-disable` without a comment is a future bug.
4. **Effects to derive state.** Compute in render; memoize if expensive.
5. **Effects on every render.** Forgot the deps array? Or deps contain a fresh object?
6. **Inline objects/arrays in render as deps.** Stable refs only.
7. **`useState(expensive())`** — `expensive()` runs every render. Use the lazy initializer: `useState(() => expensive())`.
8. **`useEffect` for data fetching** without a cancellation guard. Use React Query.
9. **`useMemo`/`useCallback` everywhere.** Adds overhead; defeats the point. Use when needed.
10. **`useRef` to share between components.** Refs aren't reactive. Use state or context.

## Drills

- Read [`packages/ui/src/hooks/use-mobile.ts`](../../packages/ui/src/hooks/use-mobile.ts). Identify the cleanup. Predict what happens if you remove it.
- Add a stale-closure bug to a small test page: a setInterval that prints `count` without using the updater form. Watch it always print `0`. Now fix it three ways: updater form, dep array, ref.
- Open [`apps/dashboard/src/components/forms/status-report/form.tsx:107-114`](../../apps/dashboard/src/components/forms/status-report/form.tsx#L107-L114). Why does the effect set `isDirty` instead of computing it in render? (Hint: dirty state lives in the parent.)
- Find a `useMemo` in the codebase and decide if it's load-bearing (a real cost is being avoided) or cargo-culted (cheap computation).
- Write your own `useDebounce(value, ms)` custom hook. Use it to delay a search input by 300ms. Include cleanup.
