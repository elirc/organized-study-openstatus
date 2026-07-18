# 01 — The JavaScript Mental Model

The goal of this file is to give you a model of the language that produces correct predictions. If your current model says "this should work" and JavaScript says "no," your model is wrong. Replace it.

## Values vs references — the foundation everyone gets wrong once

Primitives (`string`, `number`, `boolean`, `bigint`, `symbol`, `null`, `undefined`) are copied by value. Everything else (objects, arrays, functions, `Map`, `Set`, `Date`, class instances) is copied by reference.

```js
let a = { count: 1 };
let b = a;
b.count = 2;
console.log(a.count); // 2
```

If you have ever spent an hour debugging "why did my React state from yesterday come back?" — this is usually it. `useState` compares the *reference*. Mutating an object in place and calling `setState(sameRef)` is a no-op as far as React's diff is concerned.

```js
// Anti-pattern (subtle React bug):
const [items, setItems] = useState([]);
items.push(newItem);   // mutates in place
setItems(items);       // same reference → no re-render

// Correct:
setItems((prev) => [...prev, newItem]);
```

OpenStatus encodes this idea throughout: notice that Drizzle queries return *new* objects ([`packages/services/src/status-report/list.ts:152-161`](../../packages/services/src/status-report/list.ts#L152-L161)) and React Query stores immutable snapshots. The repo doesn't mutate query results — and neither should you.

**Senior aside:** *Immutability isn't a moral position; it's a reliability cheat. Mutable state requires every consumer to think about timing. Immutable state requires only the producer to think.*

## Equality: `==`, `===`, `Object.is`, and `NaN`

- `==` does type coercion. Forget it exists.
- `===` is reference equality for objects, value equality for primitives. Note: `NaN === NaN` is `false`.
- `Object.is(a, b)` is what React uses for re-render checks. Same as `===` *except* `Object.is(NaN, NaN)` is `true` and `Object.is(+0, -0)` is `false`.

```js
[1, 2] === [1, 2]        // false — different references
NaN === NaN              // false
Object.is(NaN, NaN)      // true
```

The pitfall: people write `if (a !== b)` to detect change, where `a` and `b` are objects fetched from a cache. If your cache returns a new object every read, *every comparison is true* — you'll re-render forever. React Query returns the same reference across reads when data hasn't changed (`structuralSharing`); know this so you don't fight it.

## Scope, hoisting, and the TDZ

Three scopes: global, function, block.

- `var` is function-scoped. Don't use it.
- `let` / `const` are block-scoped. Use these.
- Function declarations are hoisted *with their body*. `var` is hoisted *without* (becomes `undefined`). `let`/`const` are hoisted into a *Temporal Dead Zone* — accessing them before declaration throws.

```js
foo();         // works — function hoisted
function foo() {}

bar();         // ReferenceError — TDZ
const bar = () => {};
```

This matters when you read the codebase and see things declared after their use. In React component bodies, you can call a function declared further down — that's hoisting at work. You *cannot* reference a `const` further down — TDZ.

## Closures — the most-used and least-understood feature

A closure is the combination of a function and the lexical scope it was *declared* in. Not where it's called — where it's declared.

```js
function makeCounter() {
  let n = 0;                    // captured
  return () => ++n;             // closes over n
}
const c = makeCounter();
c(); c(); c();                  // 1, 2, 3
```

Two closures from the same `makeCounter()` call share `n`. Closures from *different* calls have independent `n`s.

This is also why **stale closures** are the most common React bug:

```jsx
function Counter() {
  const [n, setN] = useState(0);
  useEffect(() => {
    const id = setInterval(() => {
      console.log(n);            // ❌ always prints 0
    }, 1000);
    return () => clearInterval(id);
  }, []);                        // empty deps → effect captures the n from first render
}
```

Solutions:
- Add `n` to the dep array (effect re-runs, closure refreshes).
- Use the updater form `setN((prev) => prev + 1)` which doesn't depend on the captured `n`.
- Use a `useRef` to hold the latest value if you genuinely need long-lived access.

**Senior aside:** *When someone says "the effect saw stale state," the actual diagnosis is "a closure captured a value from a prior render and the effect didn't re-run." The bug isn't in React — it's in the dependency list. Always name the captured value out loud when debugging hooks.*

## The prototype model (one paragraph, because you'll rarely touch it directly)

Every object has a hidden `[[Prototype]]` link to another object. Property lookups walk the chain. `class` is sugar over this. You won't write prototype code by hand in OpenStatus, but you will see `instanceof` checks — and `instanceof` walks the prototype chain. That's why [`packages/services/src/context.ts:36-38`](../../packages/services/src/context.ts#L36-L38) uses Drizzle's `is()` instead of `instanceof`:

```ts
// from the repo: instanceof breaks across module copies in pnpm,
// because two paths to the same package mean two different
// SQLiteTransaction prototypes. is() uses a symbol tag instead.
export function isTx(db: DB): db is DrizzleTx {
  return is(db, SQLiteTransaction);
}
```

Read that comment until it clicks. It's a senior-level instinct: *don't trust `instanceof` in a monorepo with deep dependencies.*

## `this` — five rules, then forget about it

Modern code uses arrow functions and class methods. You almost never write `function() { this... }` anymore. But when you read library code, remember:

1. Default `this` is `undefined` in strict mode, global in sloppy.
2. `obj.method()` sets `this = obj`.
3. `fn.call(x)`, `fn.apply(x)`, `fn.bind(x)` set `this = x`.
4. Arrow functions don't bind `this`; they capture lexically (just like other variables).
5. Class methods, when extracted, lose `this`. `const fn = obj.method` then `fn()` will fail.

In React: never write `<button onClick={obj.method}>` if `obj.method` uses `this`. That's why classes are out and hooks are in.

## Type coercion you will hit

`+` is special — string concat if either side is a string, else numeric addition.

```js
"5" + 1   // "51"
"5" - 1   // 4
+"5"      // 5  (unary plus coerces to number)
!!"hi"    // true (idiomatic boolean cast)
```

`JSON.stringify` rules: skips `undefined` and functions in objects, but converts them to `null` in arrays. `Date` becomes an ISO string. `Map`/`Set`/`BigInt` aren't handled — they throw or return `{}`. This is why OpenStatus uses **superjson** ([`packages/api/src/trpc.ts:104-119`](../../packages/api/src/trpc.ts#L104-L119)) — it preserves `Date`, `Map`, `Set`, `BigInt`, `undefined` across the wire.

```ts
// From the repo
export const t = initTRPC
  .context<Context>()
  .meta<Meta>()
  .create({
    transformer: superjson,    // <-- because plain JSON loses too much
    errorFormatter({ shape, error }) { /* ... */ },
  });
```

Without superjson, your `date: new Date()` would come back as a string and Zod's `z.date()` validator would reject it. With superjson, it round-trips. Same for `Map` and `BigInt` values you might pass through a procedure.

## Pitfall checklist

For each one, ask yourself "could I produce this bug, and could I diagnose it in 30 seconds?"

1. **Mutated React state.** `arr.push(x); setArr(arr)` does nothing.
2. **Stale closure in an effect.** Empty dep array + captured state = always-zero log.
3. **Object equality.** `useMemo(() => ({}), [])` returns the same object; `() => ({})` in JSX creates a new one every render.
4. **`for ... of` vs `for ... in`.** `for...in` iterates *enumerable string keys* including inherited — use `for...of` for arrays.
5. **`Array.from({length: 3}).map(...)`** vs `new Array(3).map(...)`. The second one — empty slots aren't iterated.
6. **`typeof null === "object"`.** Famous bug. Always also check `!== null` when you mean "an object."
7. **Comparing dates with `===`.** `new Date() === new Date()` is `false`; use `.getTime()` or `Number(date)`.
8. **`JSON.parse(JSON.stringify(x))`** as a "deep clone." Fails on `Date`, `Map`, `Set`, `undefined`, functions, circular refs. Use `structuredClone` for plain data.
9. **`Object.keys(obj)` returns `string[]`** even when the object is keyed by union literals. You usually want `Object.keys(obj) as (keyof typeof obj)[]` — and only if you actually trust the cast.
10. **Floating point.** `0.1 + 0.2 !== 0.3`. Don't store money in floats. Use integer cents or a decimal library.

## Drills

- Predict the output of every snippet in this file *before* running it. Note any miss; that's a hole in your model.
- Find one place in [`apps/dashboard/src/`](../../apps/dashboard/src/) where a closure captures state. Explain in writing what would happen if it captured a stale value.
- Find one `as const` in the repo. Explain (in one sentence) what would change if you removed it. (Hint: try [`packages/db/src/schema/status_reports/status_reports.ts:8-13`](../../packages/db/src/schema/status_reports/status_reports.ts#L8-L13).)
- Open a JS REPL. Type `1 + null`, `1 + undefined`, `[] + []`, `[] + {}`, `{} + []`. Be horrified, then accept it.
