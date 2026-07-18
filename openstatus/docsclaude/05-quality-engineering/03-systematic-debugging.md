# 03 — Systematic Debugging

A senior debugs faster than a junior not because they know more bugs, but because they have a method. Random `console.log` is the slowest path to fixed code. Method beats intuition.

## The general method

1. **Reproduce.** A bug you can't reproduce is a bug you can't fix.
2. **Narrow.** Halve the search space repeatedly. Where in the stack is it? Where in the function? Which input triggers it?
3. **Form a hypothesis.** "I bet X is happening because of Y."
4. **Test the hypothesis cheaply.** A single log, a breakpoint, a unit test.
5. **Iterate.** Wrong hypothesis is fine — you learned something. Update your model.
6. **Fix the root cause, not the symptom.** A `try/catch` that swallows the error is symptom suppression, not fixing.

If you're spending more than 10 minutes without a working hypothesis, you've skipped step 2.

## Reproduce reliably

- What environment? (Dev, prod, specific browser)
- What user? (Anonymous, workspace member, admin)
- What inputs? (Specific record IDs, specific times)
- What steps? (Click sequence, URL navigation)

Write the repro down. If you can't write it in 5 lines, you don't understand it yet.

For OpenStatus specifically:
- Browser issues: which app (dashboard / status-page / web)? Which route? Open DevTools *before* repro.
- Server issues: which router/procedure? Check the network panel for the exact request body.
- DB issues: any error logs from libsql? Run the suspect query in `sqlite3` directly.

## Narrow ruthlessly

If you have a 200-line render with a bug, the bug is in one of 200 lines. Bisect:

- Comment out the bottom half. Bug gone? It's in the bottom. Bug still there? It's in the top.
- Repeat on the half that still fails.
- In 7-8 steps you've narrowed 200 lines to one.

For across the stack:

- Bug in the UI? Open DevTools network panel. Check what the API returned. Is the data wrong, or is the rendering wrong?
- API returns wrong data? Add a log in the service. Is the service computing wrong, or is the DB returning wrong rows?
- DB returns wrong rows? Run the query in a SQL client with the same params. Is the schema wrong, or are the indexes confusing?

You bisect across layers the same way you bisect within a function.

## Browser tools you should use daily

- **Network tab.** Filter by `Fetch/XHR`. Click the tRPC request, inspect Payload (what you sent) and Response (what came back). superjson encodes Dates as `{__superjsonMeta__:..., json:"..."}` — recognize the shape.
- **React DevTools — Components tab.** Inspect props, state, hooks. Click a component, see the chain to root.
- **React DevTools — Profiler tab.** Record a session, see render times and which props changed.
- **Performance tab.** For non-React perf: flame chart, find tasks > 50ms.
- **Console.** Use `$0` for the currently selected DOM element, `$_` for the last console expression.

If you've never used React DevTools' Profiler, stop reading and go install it. It's the single highest-ROI debugging tool for React.

## Node / server debugging

- **`console.log`** is fine. Senior engineers use it. The trick is making the logs *useful*: include the variables, the context, the request ID.
- **Source maps** — confirm they're being generated. Without them, stack traces point at minified code.
- **VS Code Debugger** — attach to the Node process. Set breakpoints. Step through. Most people skip this; it's a superpower.
- **`debugger;` statement** — pauses execution when DevTools are open. Faster than setting a breakpoint via UI.

For tRPC procedures specifically:
- `onError` ([`apps/dashboard/src/lib/trpc/shared.ts:10-15`](../../apps/dashboard/src/lib/trpc/shared.ts#L10-L15)) logs every error server-side. Tail logs while reproducing.
- Add a temporary log inside the service to verify the input matches what you expect.

## Drizzle / SQL debugging

- Drizzle has a query logger you can enable. Wrap the client with `drizzle({ ..., logger: true })` temporarily.
- Or log the raw query: `db.select().toSQL()` returns `{ sql, params }`.
- Run the SQL directly in `sqlite3 path/to/db.sqlite3` (or Turso CLI) with the same params. Does it return what you expect?

The most common Drizzle bugs:
- Missing or wrong `eq()` — the comparison is wrong.
- Wrong table aliased in a join.
- Forgotten `.where(...)` — returns everything.
- `null` semantics: `eq(col, null)` doesn't work; use `isNull(col)`.

## React / hooks debugging

The bug classes by frequency:

1. **Stale closure** — your callback sees an old value. Check dep arrays. Add a log inside the callback printing the suspect var; it should match what you expect.
2. **Wrong cache key** — React Query is returning data for a different key. Open Devtools → React Query → inspect keys.
3. **Re-render loop** — render is calling setState. Find the state update in render, move it.
4. **Stale prop** — a child component captured a prop that should have updated. Check memo equality.
5. **Effect cleanup missing** — subscriptions adding up across re-mounts. Add a cleanup.

For each: read the rendered tree state, not the source code. The tree tells you what React thinks; source tells you what you wrote.

## Logging that helps

Bad: `console.log("here")`, `console.log(x)`.

Good:
```js
console.log("[createStatusReport] input", { pageId: input.pageId, status: input.status, workspaceId: ctx.workspace.id });
```

Include:
- Label (which function).
- Key inputs (with names).
- Context (workspace id, user id, request id if available).

Logs without context are worse than no logs — they tell you something happened but not what.

For production, structured logging (JSON output) feeds aggregators. OpenStatus uses `logtape` in places; respect the existing pattern in the file you're editing.

## When to take a break

If you've been debugging for an hour with no progress, walk away. Coffee, lunch, the next day. The "fresh eyes" effect is real because:

- You drop wrong assumptions.
- You re-read the bug report with new context.
- Your subconscious worked on it.

The seniors I most respect are the first ones to say "I'm too deep, let's pair on this."

## Common false trails

- **"It must be a caching issue."** Sometimes. Often the bug is in plain code.
- **"It works locally, must be the env."** Sometimes. Often the bug is environment-independent and you got lucky locally.
- **"It's intermittent, must be a race."** Sometimes. Often it's a stale closure that reproduces inconsistently because of timing.

When you suspect one of these, *verify* before chasing.

## Real OpenStatus debugging scenarios

### Scenario: "Status page doesn't show a report I just created"

1. Repro: create a report, refresh the public page. Confirm absence.
2. Narrow: is the report in `statusPage.get`'s response? Open the network tab on the public page.
3. If in response but not on UI: bug in the client filter [`apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx:329-360`](../../apps/status-page/src/app/(status-page)/[domain]/[locale]/(public)/client.tsx#L329-L360). Check `report.statusReportUpdates.length > 0` and the `lastEvents` join.
4. If not in response: bug in the router. Check `lastEvents` computation in [`packages/api/src/router/statusPage.ts:239-247`](../../packages/api/src/router/statusPage.ts#L239-L247).
5. If `lastEvents` is empty: trace through `getEvents(...)`. Maybe the report wasn't classified as an event because it has no updates yet.

### Scenario: "Mutation succeeds but UI doesn't update"

1. Repro. Confirm.
2. Open React Query DevTools. Check the query for the data the UI shows.
3. Is the cache key being invalidated after the mutation? Compare with the `onSuccess` callback.
4. If invalidated: is the cache refetching but returning the same data? Maybe the mutation didn't actually update what you think.
5. If not invalidated: add the `invalidateQueries` call. Test.

### Scenario: "TypeScript error I can't understand"

1. Read the *full* error from the top. The first sentence is the actual mismatch.
2. The path inside `is not assignable to ...` tells you exactly where the mismatch is.
3. If the error is in library code, your call is wrong. If it's in your code, the types don't match.
4. Don't reach for `as any`. Hover over each value to see its inferred type; one of them is what you think it isn't.

## Pitfall checklist

1. **Fixing a symptom.** `try/catch` that swallows; default value to mask `undefined`.
2. **Logging without context.** "here" tells you nothing in a busy log.
3. **Chasing the wrong layer.** Bug is in the service; you're rewriting the UI.
4. **Assuming the bug is in your code.** Sometimes it's in the library — but check your code first.
5. **Reading source without observing runtime.** Source tells you the intent; runtime tells you the reality.
6. **No repro.** "Sometimes" is not a repro.
7. **Bisecting too coarsely.** Halving 200 lines should take 8 steps, not 2.
8. **Reverting good changes** to "make the bug go away." Now you have a worse codebase and the bug.
9. **`git reset` over a stash.** Don't lose work to debug; commit first.
10. **Committing the fix without understanding it.** The fix might work by accident; future-you suffers.

## Drills

- Open the dashboard. Cause a real bug (delete an `await`, mistype a Zod schema). Observe how it surfaces. Practice the diagnostic loop.
- Open React DevTools' Profiler. Record loading the dashboard. Identify the slowest render. Predict the cause; verify by inspection.
- Open a tRPC procedure and add a malformed input field via curl. Verify the error response. Decode the `zodError` field in `data`.
- Trigger a `NotFoundError` from a service by passing a bad id. Trace the error from throw to client `TRPCClientError` in the network panel.
- Enable Drizzle's query logger temporarily. Make a single tRPC call from the dashboard. Count the queries it triggers. Are any unnecessary?
