# 04 — TypeScript Essentials

This is the 90% of TypeScript you use daily. If you're comfortable with everything here, you can read most of the OpenStatus codebase without squinting. The advanced 10% lives in [`05-typescript-power-tools.md`](05-typescript-power-tools.md).

## Structural typing — the model

TypeScript's type system is **structural**: a value is assignable to a type if its shape matches, regardless of nominal identity.

```ts
type Point = { x: number; y: number };
type Position = { x: number; y: number };

const p: Point = { x: 1, y: 2 };
const q: Position = p;   // ✅ same shape, different name. Both are fine.
```

Contrast with Java/C# where you'd need `Position extends Point` or an interface declaration. TypeScript doesn't care about lineage; it cares about fit.

**Consequence:** "extra properties are fine on assignment, except in object literals." `const q: Position = { x: 1, y: 2, z: 3 }` is rejected because object *literals* are checked strictly (a guard against typos). The same object via a variable would be accepted.

## The primitives you'll actually use

`string`, `number`, `boolean`, `bigint`, `symbol`, `undefined`, `null`, `void`, `never`, `unknown`, `any`.

- `unknown` is the type-safe `any`. You can hold any value but can't use it until you narrow.
- `any` opts out of type checking entirely. Treat it as a last resort with a comment explaining why.
- `never` is the bottom type — a value that can never exist. Returned by functions that always throw or loop forever. Used for exhaustiveness checks (see below).
- `void` is the return type of functions that don't return anything useful (their actual return is `undefined`, but `void` signals "ignore me").

## Type vs interface — pick one and move on

```ts
type Page = { id: number; title: string };
interface Page { id: number; title: string }
```

Practical differences:

- `interface` supports declaration merging (multiple `interface X { ... }` in the same scope combine). Useful for module augmentation; a footgun otherwise.
- `type` supports unions, intersections, mapped types, conditional types, tuples — everything.
- Performance: TypeScript caches interfaces slightly better. Negligible in app code.

**Convention in this repo:** `type` for almost everything, `interface` when augmenting a third-party module. Pick one convention per project and stick to it.

## Literal types and `as const`

```ts
let a = "investigating";        // type: string
const b = "investigating";      // type: "investigating"
const arr = ["a", "b"];         // type: string[]
const arr2 = ["a", "b"] as const; // type: readonly ["a", "b"]
```

`as const` "freezes" the literal types. From [`packages/db/src/schema/status_reports/status_reports.ts:8-13`](../../packages/db/src/schema/status_reports/status_reports.ts#L8-L13):

```ts
export const statusReportStatus = [
  "investigating",
  "identified",
  "monitoring",
  "resolved",
] as const;
```

Without `as const`, the type would be `string[]` and Drizzle's `text("status", { enum: statusReportStatus })` would complain because the enum option needs a `readonly string[]` of literal types, not `string[]`. Then the Zod form schema uses `z.enum(statusReportStatus)` — same dependency.

This one `as const` makes the status enum *the* single source of truth. UI, form, validation, service, DB schema, and audit log all read from it.

## Unions, narrowing, discriminated unions

```ts
type Result<T> = { ok: true; value: T } | { ok: false; error: string };

function handle(r: Result<number>) {
  if (r.ok) return r.value;       // narrowed to { ok: true; value: number }
  return r.error;                  // narrowed to { ok: false; error: string }
}
```

The `ok` field is a **discriminator** — a literal-typed property that lets TypeScript narrow which branch you're in. Discriminated unions are how you encode "many possible shapes, exactly one at a time" — and TypeScript can check exhaustiveness automatically:

```ts
function fmt(r: Result<number>) {
  switch (r.ok) {
    case true: return `ok ${r.value}`;
    case false: return `err ${r.error}`;
  }
  // If you add a third variant to Result and forget to handle it,
  // TS infers `never` here, and `assertNever(r)` won't compile.
}
function assertNever(x: never): never { throw new Error(String(x)); }
```

OpenStatus uses a discriminated union for the actor type ([`packages/services/src/context.ts:13-20`](../../packages/services/src/context.ts#L13-L20)):

```ts
export type Actor =
  | { type: "user"; userId: number }
  | { type: "apiKey"; keyId: string; userId?: number; scopes: Scope[] }
  | { type: "mcp"; keyId: string; userId?: number; scopes: Scope[] }
  | { type: "slack"; teamId: string; slackUserId: string; userId?: number }
  | { type: "system"; job: string }
  | { type: "webhook"; source: string; externalId?: string }
  | { type: "subscriber"; subscriberId: number };
```

And [`packages/services/src/context.ts:60-76`](../../packages/services/src/context.ts#L60-L76) exhaustively switches over it. Add a new actor variant and TypeScript will pin down every site you need to update.

**Senior aside:** *When a codebase has more than three "modes" or "kinds," reach for a discriminated union before reaching for separate functions or a `kind: string` with optional fields. The compiler becomes your refactoring buddy.*

## Narrowing techniques

- `typeof x === "string"` — narrows primitives.
- `x instanceof Foo` — narrows objects (with the prototype caveat from `01-mental-model.md`).
- `"prop" in x` — narrows union members that have `prop`.
- User-defined type guards: `function isFoo(x: unknown): x is Foo { ... }`.
- Equality narrowing: `if (kind === "user") ...`.
- Truthiness: `if (x) ...` removes `null | undefined` (and `""`, `0`, `false` if those were members — careful).

The form submit in [`apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx:89-103`](../../apps/dashboard/src/app/(dashboard)/status-pages/[id]/status-reports/page.tsx#L89-L103) uses the `"in"` narrow:

```tsx
onSubmit={async (values) => {
  // values is a union of FormValues, narrowed by presence of `date`
  if ("date" in values) {
    await createStatusReportMutation.mutateAsync({ ... });
  }
}}
```

That `"date" in values` is doing real work — it tells TypeScript which branch of `FormValues` we're in (create vs update), so the call below compiles only with the create-only fields.

## Generics — read them, then write them

```ts
function identity<T>(x: T): T { return x; }
identity(42);        // T inferred as number
identity<string>("hi"); // T explicit
```

Constraints:
```ts
function pluck<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
```

Generic defaults:
```ts
type ApiResult<T = unknown> = { data: T };
```

Bounded inference is the most common surprise. `T extends string` means "T is a subtype of string" — including string literals. So `pluck({a: 1}, "a")` infers `K` as the literal `"a"`, and the return type is exactly `number`, not `unknown`.

When you read tRPC or Drizzle code, almost every helper is generic. You don't have to memorize how — you just have to read the call site and trust the inferred types.

## `keyof`, `typeof`, indexed access

```ts
type Page = { id: number; title: string; createdAt: Date };
type K = keyof Page;            // "id" | "title" | "createdAt"
type T = Page["title"];         // string
type Vals = Page[keyof Page];   // number | string | Date

const status = { investigating: 0, resolved: 1 };
type StatusKey = keyof typeof status;   // "investigating" | "resolved"
```

`typeof x` in a type position turns a value into its type. This is how you turn an `as const` array into a union:

```ts
const statuses = ["a", "b", "c"] as const;
type Status = typeof statuses[number];   // "a" | "b" | "c"
```

That's exactly the pattern Drizzle and Zod use. Once you see it, you'll see it everywhere.

## Utility types you'll use weekly

- `Partial<T>` — every property optional.
- `Required<T>` — every property required.
- `Readonly<T>` — every property readonly.
- `Pick<T, K>` — keep only listed keys.
- `Omit<T, K>` — drop listed keys.
- `Record<K, V>` — `{ [k in K]: V }`.
- `ReturnType<F>`, `Parameters<F>`, `Awaited<P>` — extract from function/promise types.
- `NonNullable<T>` — drops `null | undefined`.

Real usage from this repo, [`packages/api/src/service-adapter.ts:8-11`](../../packages/api/src/service-adapter.ts#L8-L11):

```ts
type AuthedContext = Context & {
  user: { id: number };
  workspace: NonNullable<Context["workspace"]>;
};
```

This says "an `AuthedContext` is a `Context` *and* a `user` exists *and* `workspace` is not null." Intersections (`&`) combine types; `NonNullable` strips nullishness. The result is a type that perfectly encodes the invariant `protectedProcedure` has already enforced at runtime.

## Inference — the secret of "magical" libraries

tRPC's type inference is not magic — it's `typeof`, generics, and `as const` composed deeply. You write:

```ts
export const appRouter = createTRPCRouter({
  page: pageRouter,
  monitor: monitorRouter,
  // ...
});
export type AppRouter = typeof appRouter;
```

And the client gets `trpc.page.list.queryOptions()` typed end-to-end. The "magic" is just `typeof` capturing the router object's shape, generics propagating the input/output types of each procedure, and the client wrapping that shape in helper types. Once you understand the pattern, you can build similar APIs.

## Common type errors and what they actually mean

- `Type 'X' is not assignable to type 'Y'.` — Read top-down. The mismatch is usually in a nested property; TS shows the full path. Don't cast — fix the data or the type.
- `Property 'x' does not exist on type 'Y'.` — Either you forgot to narrow a union, or `Y` doesn't actually have `x`. Don't reach for `as any`; widen the type or add a guard.
- `Object is possibly 'undefined'.` — `strictNullChecks` is on. Use `?.`, `??`, or an explicit guard. Or `!` if you can *prove* it's not undefined — but `!` is a moral debt.
- `Argument of type 'string' is not assignable to parameter of type '"a" | "b"'.` — You're passing a wider type into a narrower slot. Either narrow the source (`if (x === "a" || x === "b") ...`) or assert with a comment explaining the safety.
- `Index signature is missing in type 'X'.` — Your type has known keys, but the consumer wants `[k: string]: ...`. Add the index signature or use `Record`.

## Pitfall checklist

1. **`as any` to silence the compiler.** You will pay this debt with interest. Use `unknown` and narrow, or fix the type.
2. **Non-null assertion (`!`) on values that can actually be null.** It's a silent runtime crash waiting.
3. **Casting between unrelated types.** `x as Foo` works if `x`'s type *overlaps* with `Foo`. For unrelated types you need `as unknown as Foo` — and if you're doing that, write a helper with a comment about why it's safe.
4. **Lying with declared types.** `const x: Foo = bar as Foo` is worse than `as any`. You've poisoned the type system.
5. **Object literal "excess property" surprises.** Move to a variable to bypass; *but think about whether the excess is a typo first*.
6. **Generic inference with `any`.** If you ever see `function foo<T = any>` you have an escape hatch that defeats inference. Default to `unknown`.
7. **Missing `as const` on enum arrays.** Type widens to `string[]`, all downstream literal narrowing breaks.
8. **Reaching for `enum`.** TS `enum` has weird runtime semantics (bidirectional maps, generated objects). Prefer `as const` literal unions.
9. **Mutating a `Readonly` type by casting it away.** You're lying to every reader.
10. **Forgetting that TypeScript types are erased at runtime.** `typeof x` (value) and `typeof x` (type) are different things. You can't `instanceof MyType` — that doesn't exist at runtime.

## Drills

- Open [`packages/services/src/context.ts:13-20`](../../packages/services/src/context.ts#L13-L20). Add a new actor variant `cron` and try to compile. Which sites does TypeScript force you to update? Now revert.
- Find one `as` cast in the codebase. Decide if it's load-bearing (the comment explains a runtime guarantee) or a smell (no comment, no guarantee). Two examples to start: [`packages/services/src/context.ts:46`](../../packages/services/src/context.ts#L46) and [`apps/dashboard/src/lib/trpc/server.tsx:53`](../../apps/dashboard/src/lib/trpc/server.tsx#L53).
- Write a type-level test: `type _Check = AssertEqual<keyof typeof statusReportStatus, "0" | "1" | "2" | "3" | "length" | ...>`. (Half-trick — that's how `keyof` of an array works. The element type is `(typeof statusReportStatus)[number]`.)
- Read [`packages/api/src/router/statusReport.ts:24-32`](../../packages/api/src/router/statusReport.ts#L24-L32) and explain in writing what `CreateStatusReportInput.shape.status` does and why the schema can reuse it. (Hint: Zod gives you `.shape` to access subschemas.)
