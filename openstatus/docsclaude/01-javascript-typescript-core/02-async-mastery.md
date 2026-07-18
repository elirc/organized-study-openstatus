# 02 — Async Mastery

Asynchronous bugs are the most expensive bugs you'll write. They pass review, pass tests, and only fail under load or in race conditions. This file gives you the model to predict them before you write them.

## The event loop in one paragraph

JavaScript is single-threaded. There is one call stack. There are several queues:

- **Microtask queue** — promise callbacks (`.then`, `await` continuations), `queueMicrotask`, `MutationObserver`.
- **Macrotask queue** — `setTimeout`, `setInterval`, I/O callbacks, UI events.

After each macrotask finishes and the stack drains, the runtime processes **all** pending microtasks before picking up the next macrotask. That's why `await` "happens before" the next `setTimeout(fn, 0)` — even though both are nominally "later."

```js
console.log(1);
setTimeout(() => console.log(2), 0);
Promise.resolve().then(() => console.log(3));
console.log(4);
// 1, 4, 3, 2
```

If your prediction was different, stop and re-read. This is the single most important diagram in async JS.

## Promises — three states, three rules

A promise is `pending`, then exactly one of `fulfilled` (resolved with a value) or `rejected` (with a reason). It never changes again.

Three rules every senior knows:

1. **`.then` returns a new promise.** Chains are *composed*; the original isn't mutated.
2. **Throwing inside `.then` rejects the returned promise.** Synchronous `throw` and `Promise.reject` both become rejections.
3. **An unhandled rejection is a bug.** In Node it warns; in newer Node it crashes. Always handle, always.

```js
fetch("/api/x")
  .then((res) => res.json())          // returns a new promise
  .then((data) => process(data))      // chained on that
  .catch((err) => log(err));          // catches anywhere upstream
```

## `async/await` is sugar over `.then`

```js
async function load() {
  try {
    const res = await fetch("/api/x");
    const data = await res.json();
    return process(data);
  } catch (err) {
    log(err);
  }
}
```

Equivalent semantics, much more readable. But `await` has one consequence you must internalize: **it pauses the surrounding function**. Code after `await` runs as a microtask after the awaited promise settles. The rest of the program keeps going.

## The single most common async bug: serial when you meant parallel

```js
// ❌ Serial. Each await blocks the next.
for (const id of ids) {
  results.push(await fetchOne(id));
}
// 1000 ids × 50ms = 50 seconds.
```

```js
// ✅ Parallel.
const results = await Promise.all(ids.map(fetchOne));
// 1000 ids in ~50ms (subject to connection limits).
```

OpenStatus encodes this pattern when loading status report relations — three queries in parallel, not serial. From [`packages/services/src/status-report/list.ts:184-202`](../../packages/services/src/status-report/list.ts#L184-L202):

```ts
const [countRow, rows] = await Promise.all([
  db.select({ count: sql<number>`count(*)` }).from(statusReport).where(whereClause).get(),
  db.select().from(statusReport).where(whereClause).orderBy(...).limit(...).offset(...).all(),
]);
```

Two independent queries, one round of latency. A junior writes:
```ts
const countRow = await db.select(...);
const rows = await db.select(...);
```
which doubles latency for no reason.

**When to use serial:** when later work depends on the earlier result, OR when you need to bound concurrency to avoid hammering an upstream. For bounded concurrency use a library (`p-limit`) or chunk with `Promise.all` per chunk.

## `Promise.all` vs `Promise.allSettled` vs `Promise.race` vs `Promise.any`

- `Promise.all([a, b, c])` — fulfills with `[av, bv, cv]` once all fulfill. Rejects on the first rejection, throwing away the other results.
- `Promise.allSettled` — fulfills with `[{status, value|reason}, ...]`. Never rejects. Use when "I want results from whatever succeeded."
- `Promise.race` — fulfills/rejects with whichever settles first.
- `Promise.any` — fulfills with the first fulfilled value, rejects only if *all* reject.

Use case in OpenStatus: notifications. If you send to 5 channels and one fails, you don't want the other 4 to be "discarded." `allSettled` lets you record the partial outcome.

## `await` in a loop with side effects you can't parallelize

Sometimes you need serial — DB writes that depend on prior IDs, rate-limited APIs. Then `for ... of` with `await` inside is correct. Don't use `forEach`:

```js
// ❌ Useless. forEach ignores returned promises; the callback runs, but
//    the outer function doesn't wait. Sequencing is lost.
arr.forEach(async (x) => await doIt(x));
```

```js
// ✅
for (const x of arr) await doIt(x);
```

## The "fire and forget" trap

```js
async function handler(req) {
  someAuditCall();   // ❌ not awaited — if it throws, you get an unhandled rejection
  return ok();
}
```

If you genuinely want to start work and return, do it deliberately:

```js
void (async () => {
  try { await someAuditCall(); } catch (err) { logger.error(err); }
})();
```

Next.js gives you `after()` for this pattern — defer work until after the response is sent. OpenStatus uses it ([`packages/api/src/trpc.ts:213-238`](../../packages/api/src/trpc.ts#L213-L238)):

```ts
// from the repo: analytics tracking after a successful request,
// non-blocking, with errors swallowed inside the async block.
after(async () => {
  if (meta?.track) {
    const analytics = await setupAnalytics(identify);
    await analytics.track({ ...meta.track, ...additionalProps });
  }
});
```

**Senior aside:** *Any time you start async work that the caller doesn't wait for, you have committed to handling its errors yourself. The runtime will not save you.*

## Race conditions and stale responses

Classic React pattern:

```jsx
function Search({ query }) {
  const [results, setResults] = useState([]);
  useEffect(() => {
    fetch(`/api?q=${query}`).then((r) => r.json()).then(setResults);
  }, [query]);
}
```

If the user types `a`, then `ab` quickly, and the `a` response is slower, you'll show `a`'s results after `ab`'s — a stale write. Fixes:

- **AbortController** to cancel the older fetch:
  ```jsx
  useEffect(() => {
    const ctrl = new AbortController();
    fetch(url, { signal: ctrl.signal }).then(...);
    return () => ctrl.abort();
  }, [query]);
  ```
- **Stale check** with a ref or version counter.
- **Use React Query.** It handles this for you because each cache key gets its own slot, and you observe the latest matching query.

OpenStatus avoids this category entirely by using React Query for every data fetch. When you see a `useEffect` with a raw `fetch` in a PR, push back hard.

## Microtask starvation

A tight loop of microtasks can starve macrotasks. Example: a recursive `Promise.resolve().then(...)` that never yields. The page becomes unresponsive even though "nothing is running." Rare in app code, common in poorly written stream processors.

## `try/catch` and `await`

`try/catch` around `await` catches the rejection. Without `await`, it does *not* — the promise is returned, and any rejection becomes the *caller's* problem.

```js
async function bad() {
  try {
    return doIt();   // returns the promise; try doesn't catch its rejection
  } catch {}
}

async function good() {
  try {
    return await doIt();   // now catch sees the rejection
  } catch {}
}
```

The `await` matters even when you're just "passing it through."

## Top-level await

In ESM, you can `await` at the top of a module. Node and modern bundlers support it. It blocks importing modules until the awaited promise settles — useful for config, dangerous if you depend on a slow network. OpenStatus avoids top-level await in app code; library code uses it sparingly.

## Pitfall checklist

1. **Serial loops over independent work.** `Promise.all`.
2. **`forEach(async)`.** Use `for...of` or `Promise.all`.
3. **Returning a promise without `await` inside a try/catch.** Catch does nothing.
4. **Unhandled rejections.** Always end every detached chain with `.catch` or `try/await/catch`.
5. **Stale responses.** Use a cancellation token or React Query.
6. **Mixing throwing and returning promises.** Don't do `throw fooAsync()` — `throw` expects a sync value.
7. **`await new Promise(resolve => {...})` without ever calling `resolve`.** Hangs forever. The "lost" promise.
8. **Awaiting inside a `forEach` callback** thinking the outer function waits. It doesn't.
9. **Setting state inside a `setTimeout` after unmount.** Memory leak warning; capture cleanup in a `useEffect` return.
10. **`async` event handlers in React** that the user can re-trigger before the first finishes. Disable the button or guard with a ref.

## Drills

- Predict the order of three logs in [`packages/api/src/trpc.ts:213-238`](../../packages/api/src/trpc.ts#L213-L238) if the request takes 100ms and the analytics call takes 200ms. Why doesn't the user wait for analytics?
- Rewrite the serial DB writes in [`packages/services/src/status-report/create.ts:54-95`](../../packages/services/src/status-report/create.ts#L54-L95) to use `Promise.all`. Then explain why the original code is *correct* and your "improvement" is a bug. (Hint: transactions, audit ordering.)
- Open a REPL and write a function that returns the result of the fastest of three URL fetches and ignores the others. Use `Promise.race`. Then change it to "the first one that *succeeds*" using `Promise.any`. Then try the implementation that handles "the fastest 2 of 3" — a senior interview classic.
