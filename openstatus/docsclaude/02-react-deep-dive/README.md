# 02 — React Deep Dive

The goal of this module is to make React stop being mysterious. When you understand the render model, hooks become rules-of-thumb you can derive instead of memorize, and "why does this re-render?" becomes a 30-second answer.

## What separates levels in React

| Level | They can | They struggle with |
| --- | --- | --- |
| Junior | Build a form, add state, fetch data | Stale closures, unnecessary re-renders, lifting state, when to memo |
| Mid | All of the above + Context, custom hooks, basic perf | RSC boundaries, server actions, query cache invalidation timing, suspense |
| Senior | All of the above + design state architectures, choose between local/server/URL state, debug 5-second renders | Knowing when *not* to use React, when to drop to refs, when to accept the cost |

This module is built to push you up two levels. The biggest leap is internalizing **the render model**. Everything else follows from it.

## Files

| # | File | What it covers |
| --- | --- | --- |
| 01 | [`01-how-react-actually-renders.md`](01-how-react-actually-renders.md) | Render vs commit, reconciliation, keys, why hooks have rules. |
| 02 | [`02-hooks-from-first-principles.md`](02-hooks-from-first-principles.md) | `useState`/`useEffect`/`useRef`/`useMemo`/`useCallback`/`useReducer` — what they actually do. |
| 03 | [`03-state-architecture.md`](03-state-architecture.md) | Local, lifted, server, derived, URL state — picking the right home for each piece of state. |
| 04 | [`04-forms-and-validation.md`](04-forms-and-validation.md) | react-hook-form + Zod, controlled vs uncontrolled, async submission patterns. |
| 05 | [`05-rsc-and-data-flow.md`](05-rsc-and-data-flow.md) | Server vs client components, `"use client"`, hydration, server actions, Next.js App Router. |
| 06 | [`06-react-query-and-trpc.md`](06-react-query-and-trpc.md) | The actual mental model of React Query, cache keys, invalidation, the tRPC integration. |
| ⨯ | [`exercises.md`](exercises.md) | Drills, including "predict the re-render" exercises grounded in this repo. |

## How to work this module

For files 01–02, read with a small repro project (or sandbox) open. *Modify* the examples. Add a `console.log` to a render and another to an effect and watch what happens when you click a button.

For files 03–06, alternate between reading and opening the corresponding OpenStatus files. The status-report flow stays the touchstone, but we'll branch into monitors, notifications, and the public status page so the patterns generalize.
