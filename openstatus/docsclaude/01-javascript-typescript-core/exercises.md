# 01 — Exercises

Do these in a scratch file or REPL. Don't ask for the answer; predict first, run second, then explain the gap. The gap is the lesson.

## JS mental model

1. Predict the output of:
   ```js
   const a = { x: 1 };
   const b = a;
   b.x = 2;
   console.log(a.x);
   ```
2. Write a function `clone(obj)` that produces a shallow copy. Then write `deepClone(obj)` without using `structuredClone`. Now compare with `structuredClone(obj)`. Identify three inputs where your `deepClone` and `structuredClone` differ.
3. Write a function that, given an array, returns a new array with duplicates removed using `Set`. Then write the version using `filter`. Which is O(n²)?
4. Predict what `[].length`, `Object.keys([1, 2, 3])`, and `Object.keys({0: "a", 1: "b"})` evaluate to. Why?
5. Why does `[1, 2, 3] == [1, 2, 3]` evaluate to `false`? What about `[] == false`? (Hint: read the abstract equality algorithm spec. Then never use `==` again.)

## Closures and `this`

6. Build a counter factory that returns `{ increment, decrement, get }`. Show that two counters from two factory calls are independent.
7. Take a class with a method that uses `this`. Extract the method into a variable and call it. Watch it break. Then fix it three ways: arrow class property, `.bind(this)` in the constructor, wrap in an arrow at the call site. Which one would you pick in production and why?
8. Find one closure in [`packages/services/src/status-report/list.ts`](../../packages/services/src/status-report/list.ts) (hint: the `enrichReportsBatch` helper closes over `db`). What would break if you tried to call it across two requests?

## Async

9. Without running it, predict the order of logs:
   ```js
   console.log("A");
   setTimeout(() => console.log("B"), 0);
   Promise.resolve().then(() => console.log("C")).then(() => console.log("D"));
   console.log("E");
   ```
10. Write `mapSerial(items, fn)` (await one at a time) and `mapParallel(items, fn)` (await all). Use both on `[1,2,3]` with `fn = (x) => new Promise(r => setTimeout(() => r(x*2), 100))`. Time both with `console.time`.
11. Write `mapWithConcurrency(items, fn, limit)` that runs at most `limit` in parallel. (No external libs.)
12. Write a function that races two fetches and uses an `AbortController` to cancel the loser. Verify with a network panel that the loser's request is actually aborted.
13. Read [`packages/api/src/trpc.ts:213-238`](../../packages/api/src/trpc.ts#L213-L238). Why is `after()` used instead of `await`? What's the user-facing consequence?

## Modules and tooling

14. Add a new file `packages/services/src/_play/foo.ts` exporting a function. Try to import it from `apps/dashboard`. What stops you? Add an `exports` entry to `packages/services/package.json`. Now it works. Remove your changes.
15. Run `pnpm why react`. Walk the dependency tree. Why does `react` appear from multiple paths? Are they the same version?
16. In `turbo.json`, find the `build` task's `outputs`. Predict what happens if you delete the output directory and run `turbo run build` twice in a row.
17. Open the root `package.json`. Identify one script that exists only to compose other scripts. Identify one script that does real work. What's the difference in cacheability?

## TypeScript essentials

18. Type the following without using `any`:
    ```ts
    function pick<T, K extends keyof T>(obj: T, ...keys: K[]): Pick<T, K> { /* ... */ }
    ```
19. Build a discriminated union for a `Shape` (circle, rect, triangle). Write a `area(s: Shape): number` with exhaustive checking via `assertNever`. Add `polygon` to the union — TypeScript should yell at you.
20. Read [`packages/services/src/context.ts:13-20`](../../packages/services/src/context.ts#L13-L20). Extract just the `apiKey` variant's type using a conditional + `infer`. Now do it with `Extract<Actor, { type: "apiKey" }>`. Which is more readable?
21. Open [`apps/dashboard/src/components/forms/status-report/form.tsx:62-77`](../../apps/dashboard/src/components/forms/status-report/form.tsx#L62-L77). Note the `schema` and `updateSchema`. Write `type FormValues = z.infer<typeof schema> | z.infer<typeof updateSchema>` and inspect the resulting type in your editor. Now run `Extract<FormValues, { date: Date }>`. What did you get?
22. Why does this fail?
    ```ts
    const arr = ["a", "b"];
    type X = typeof arr[number];   // string, not "a" | "b"
    ```
    Fix it.

## TypeScript power tools

23. Write `type FunctionPropertyNames<T>` that returns the keys of `T` whose values are functions.
24. Build `Promised<T>` and `Unpromised<T>` (see drill 1 in `05-typescript-power-tools.md`).
25. Read [`packages/api/src/trpc.ts:96`](../../packages/api/src/trpc.ts#L96). Decompose `Context = Awaited<ReturnType<typeof createTRPCContext>>`. What does each utility type do? What would break if you hand-wrote `Context`?
26. Write a branded `WorkspaceId` and `PageId`. Type a `loadPage(id: PageId)` function. Show that passing a `WorkspaceId` errors. Then locate a real spot in the codebase where this brand would have caught a bug.

## Self-grading

For each problem, score yourself:

- **0** — couldn't attempt; missing prerequisite knowledge.
- **1** — got close, off on details.
- **2** — got it right, but had to look something up.
- **3** — got it right, can explain *why* without notes.

You're senior-shaped on this material when you score 2+ on at least 80% of the problems and 3 on the discriminated-union / inference / `as const` ones (those are the load-bearing pieces for the rest of the curriculum).
