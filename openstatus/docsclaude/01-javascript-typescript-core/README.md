# 01 — JavaScript & TypeScript Core

This module is the language foundation. Most "I can't figure out this bug" moments in modern TS codebases trace back to a shaky mental model of one of: closures, the event loop, ESM resolution, structural typing, or TypeScript inference. Fix the foundation here and everything above it gets easier.

## Why language depth matters at senior level

A senior engineer can answer:

- "Why does this `await` inside a loop serialize requests, but `Promise.all` doesn't?"
- "Why does `useEffect` see a stale `count`?"
- "Why does `JSON.stringify(new Date())` return a string but Drizzle stores a number?"
- "Why does `as const` change what TypeScript infers from this array?"
- "Why does importing from `node:fs` break in the Edge runtime?"

None of these require senior-level *codebase* knowledge. They require senior-level *language* knowledge. That's why this module comes first.

## Files

| # | File | What it covers |
| --- | --- | --- |
| 01 | [`01-mental-model.md`](01-mental-model.md) | Value vs reference, equality, scope, closures, the prototype model, `this`. |
| 02 | [`02-async-mastery.md`](02-async-mastery.md) | Event loop, microtasks, promise mechanics, `async/await` pitfalls, real concurrency patterns in this repo. |
| 03 | [`03-modules-and-tooling.md`](03-modules-and-tooling.md) | ESM vs CJS, `package.json` exports, pnpm workspaces, Turborepo, how this monorepo actually resolves imports. |
| 04 | [`04-typescript-essentials.md`](04-typescript-essentials.md) | Structural typing, narrowing, unions, generics, inference — the 90% you use daily. |
| 05 | [`05-typescript-power-tools.md`](05-typescript-power-tools.md) | Conditional/mapped types, `infer`, template literal types, `satisfies`, branded types, the tricks behind tRPC/Drizzle/Zod. |
| ⨯ | [`exercises.md`](exercises.md) | Drills you can do without writing production code. |

## How to work this module

Read each file with a REPL or scratch file open. Run the examples. *Modify* the examples — break them on purpose and predict the error. The single fastest way to grow language fluency is to be wrong about predictions and then learn why.

Don't read this module passively. Pure reading sticks for about three days. Active prediction-and-correction sticks for years.
