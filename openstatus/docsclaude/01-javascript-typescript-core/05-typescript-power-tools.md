# 05 — TypeScript Power Tools

The features in this file are what separate "I write TypeScript" from "I read library code without flinching." You won't use all of these every day, but you'll read them every day if you work on tRPC, Drizzle, Zod, or React Query.

## Conditional types

```ts
type IsString<T> = T extends string ? true : false;
type A = IsString<"hi">;   // true
type B = IsString<42>;     // false
```

The `T extends X ? Y : Z` shape. When `T` is a union, the conditional distributes:

```ts
type ToArr<T> = T extends unknown ? T[] : never;
type X = ToArr<"a" | "b">;   // "a"[] | "b"[]
```

To prevent distribution, wrap in a tuple:

```ts
type ToArr2<T> = [T] extends [unknown] ? T[] : never;
type Y = ToArr2<"a" | "b">;   // ("a" | "b")[]
```

OpenStatus's adapter [`packages/api/src/service-adapter.ts:8-11`](../../packages/api/src/service-adapter.ts#L8-L11) doesn't use a conditional, but tRPC's internals are full of them — they're how `protectedProcedure.input(...).query(...)` chains stay typed end-to-end without you specifying anything.

## `infer` — extracting types from positions

`infer` introduces a fresh type variable inside a conditional and binds it to whatever matches:

```ts
type MyReturn<F> = F extends (...args: any[]) => infer R ? R : never;
type N = MyReturn<() => number>;   // number
```

Built-in `ReturnType<F>` is exactly this. So is `Awaited<P>`:

```ts
type MyAwaited<T> = T extends Promise<infer U> ? MyAwaited<U> : T;
type Q = MyAwaited<Promise<Promise<string>>>;   // string
```

You'll see `infer` whenever a library needs to "peek inside" a type. The Drizzle types use `infer` to extract the row type from a table definition; tRPC uses it to extract input/output types from a procedure builder chain.

A real OpenStatus example, [`packages/services/src/context.ts:7-11`](../../packages/services/src/context.ts#L7-L11):

```ts
export type DrizzleClient = typeof defaultDb;
export type DrizzleTx = Parameters<
  Parameters<DrizzleClient["transaction"]>[0]
>[0];
```

Read it inside-out:
1. `DrizzleClient["transaction"]` — the type of `db.transaction`.
2. `Parameters<...>[0]` — its first argument, which is a callback.
3. `Parameters<callback>[0]` — that callback's first argument, which is the `tx` handle.

You've now extracted a type that the library never named. That's the power of utility types built on `infer`.

## Mapped types

```ts
type Mutable<T> = { -readonly [K in keyof T]: T[K] };
type Optional<T> = { [K in keyof T]?: T[K] };
type Stringify<T> = { [K in keyof T]: string };
```

`+` and `-` modifiers add or remove `readonly` and `?`. Combined with conditional types, you can transform shapes arbitrarily:

```ts
type NonNullableProps<T> = {
  [K in keyof T]: NonNullable<T[K]>
};
```

Zod's `.partial()`, `.required()`, `.deepPartial()` are all mapped-type operations under the hood.

## Template literal types

```ts
type Greeting = `hello, ${string}`;
type Status = "investigating" | "identified" | "monitoring" | "resolved";
type Event = `status_report.${Status}`;
// "status_report.investigating" | "status_report.identified" | ...
```

Combined with mapped types, you can generate event names, route paths, or column aliases at the type level:

```ts
type GettersOf<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K]
};
type X = GettersOf<{ id: number; title: string }>;
// { getId: () => number; getTitle: () => string }
```

`Capitalize`, `Uppercase`, `Lowercase`, `Uncapitalize` are built-in helpers.

OpenStatus's audit log uses literal-template-style discriminated actions (`status_report.create`, `monitor.update`, etc.) — they're regular string literals today, but you could constrain them with a template type to prevent typos at the type level.

## `satisfies` — the operator you wish you'd known sooner

```ts
const config = {
  retries: 3,
  backoff: "exponential",
} satisfies { retries: number; backoff: "linear" | "exponential" };
```

`satisfies` checks that the value matches the constraint *without widening the value's type*. Compare:

```ts
const a: { backoff: "linear" | "exponential" } = { backoff: "exponential" };
// a.backoff is "linear" | "exponential" — widened.

const b = { backoff: "exponential" } satisfies { backoff: "linear" | "exponential" };
// b.backoff is "exponential" — preserved.
```

When you want both type safety *and* literal precision, reach for `satisfies`. From [`apps/dashboard/src/lib/trpc/shared.ts:51-56`](../../apps/dashboard/src/lib/trpc/shared.ts#L51-L56):

```ts
const sharedOpts = {
  headers: opts?.headers,
  fetch: opts?.fetch,
  transformer: superjson,
} satisfies Partial<HTTPBatchLinkOptions<any>>;
```

The author wanted "check that I'm passing valid HTTPBatchLinkOptions, but keep the precise type of `superjson` so the link's transformer slot type-checks correctly downstream."

## Branded / nominal types

Structural typing means `type UserId = number` and `type WorkspaceId = number` are interchangeable — a foot-gun for IDs. Branding fixes it:

```ts
type Brand<T, B> = T & { readonly __brand: B };
type UserId = Brand<number, "UserId">;
type WorkspaceId = Brand<number, "WorkspaceId">;

function load(id: UserId) { /* ... */ }
const wid = 5 as WorkspaceId;
load(wid);   // ❌ TypeScript catches it
```

OpenStatus doesn't brand IDs today (a tradeoff against readability). A senior making the case for branding would point at IDOR-class bugs (like the test in [`packages/api/src/router/statusReport.test.ts:77-99`](../../packages/api/src/router/statusReport.test.ts#L77-L99)): branding `WorkspaceId` and `PageId` would let the compiler catch some cross-workspace mistakes before they reach the test layer.

## The Drizzle / Zod / tRPC type tricks

You don't need to memorize how they work; you need to recognize the patterns.

**Drizzle:** schema is values (`sqliteTable(...)`), but `typeof statusReport.$inferSelect` gives you a row type derived from the schema. From [`packages/db/src/schema/status_reports/status_reports.ts:15-39`](../../packages/db/src/schema/status_reports/status_reports.ts#L15-L39), the table declaration. Elsewhere in the schema validation file, `createSelectSchema(statusReport)` turns the table into a Zod schema. The chain: table → row type, table → zod schema, zod schema → input/output types. One source of truth, three derived views.

**Zod:** `z.infer<typeof Schema>` is your friend. `Schema.shape.status` lets you pluck out a subschema (used heavily in [`packages/api/src/router/statusReport.ts:24-32`](../../packages/api/src/router/statusReport.ts#L24-L32) to reuse the canonical enum). `z.coerce.date()` accepts strings/numbers and returns Dates — what makes cross-network date round-trips work.

**tRPC:** the entire client type comes from `typeof appRouter`. Every helper (`queryOptions`, `mutationOptions`, `useQuery`, `useMutation`) is generic over that router type. You don't write a single hand-typed API client — the inference walks the router shape, finds each procedure's input/output, and exposes typed methods on the client object.

## Type-level testing

You can write tests that fail at *compile* time:

```ts
type AssertEqual<A, B> =
  (<T>() => T extends A ? 1 : 2) extends
  (<T>() => T extends B ? 1 : 2) ? true : false;

type T1 = AssertEqual<Awaited<Promise<number>>, number>;   // true
type _check1 = T1 extends true ? unknown : never;
```

The `_check1` line is a no-op at runtime but errors at compile time if `T1` is `false`. Useful for library code where you want to lock in type guarantees.

## `never` for exhaustiveness

```ts
function describe(a: Actor): string {
  switch (a.type) {
    case "user": return `user ${a.userId}`;
    case "apiKey": return `key ${a.keyId}`;
    // ... handle every variant
    default: return assertNever(a);
  }
}
function assertNever(x: never): never {
  throw new Error(`Unhandled actor: ${JSON.stringify(x)}`);
}
```

If you add a new `Actor` variant to [`packages/services/src/context.ts:13-20`](../../packages/services/src/context.ts#L13-L20) and forget to handle it in a switch elsewhere, `a` won't narrow to `never` in the `default` branch, and `assertNever(a)` won't compile. The exhaustive check turns into a compile error.

This is the cheapest, most senior-shaped TypeScript pattern in the language. Use it whenever you switch over a discriminated union.

## `unknown` instead of `any` — always

```ts
function handle(err: unknown) {
  if (err instanceof ServiceError) { /* narrow */ }
  if (err instanceof ZodError)    { /* narrow */ }
  throw new InternalError("Unhandled", { cause: err });
}
```

Compare to [`packages/api/src/service-adapter.ts:32-87`](../../packages/api/src/service-adapter.ts#L32-L87) — that's exactly this pattern. `toTRPCError(err: unknown)` narrows step by step. The day you change `unknown` to `any`, you also lose the compiler's reminders to handle every branch.

## Pitfall checklist

1. **Distributive conditional surprise.** `T extends ... ? ... : ...` distributes over unions. Wrap in `[T]` to prevent.
2. **`infer` outside a conditional.** It only works inside `extends ? :`.
3. **Mapped type over a primitive.** `{ [K in keyof string]: ... }` gives you `string`'s instance methods — usually not what you wanted.
4. **`satisfies` vs `as` confusion.** `as` casts (can lie). `satisfies` checks (can't lie). When in doubt, `satisfies`.
5. **Branded types defeated by `as`.** A single careless cast undoes the brand. Centralize the cast in a `toUserId` helper.
6. **`type` cycles.** `type A = B; type B = A` is a circular alias; can lock up the type checker.
7. **Heavy types in hot paths.** Library-grade conditional types are slow. Don't put `DeepReadonly<DeepPartial<...>>` on every form prop; you'll pay in editor lag.
8. **Forgetting `readonly` for arrays from `as const`.** You can't `.push` to a readonly array; use spread.
9. **Using `enum` for new code.** Prefer `as const` literal unions. They're erased; enums aren't.
10. **`ts-ignore` or `ts-expect-error`** without a comment explaining why. The latter at least fails when the issue is fixed, but both decay into folklore quickly.

## Drills

- Write `type Promised<T> = { [K in keyof T]: Promise<T[K]> }`. Use it on `{ a: number; b: string }`.
- Write `type Unpromised<T>` that does the inverse. (Hint: `infer`.)
- Read [`packages/api/src/router/statusReport.ts:24-32`](../../packages/api/src/router/statusReport.ts#L24-L32). Identify every place that `CreateStatusReportInput.shape.X` is reused. Why is this better than re-declaring the field?
- Write an `assertNever` for the `Actor` union. Add a comment about *why* it's a compile-time guard.
- Open [`packages/api/src/trpc.ts:96`](../../packages/api/src/trpc.ts#L96): `export type Context = Awaited<ReturnType<typeof createTRPCContext>>`. Decompose it: what does each utility type do? What would break if you wrote `Context` by hand instead?
