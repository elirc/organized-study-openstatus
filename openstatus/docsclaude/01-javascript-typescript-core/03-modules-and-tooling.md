# 03 — Modules, Packages, and the Monorepo Toolchain

The module system is invisible until it breaks. Then it eats your day. This file explains what's actually happening when you write `import { x } from "@openstatus/db"`, why pnpm workspaces are different from npm/yarn, what Turborepo adds, and how to read a `package.json` like a senior would.

## ESM vs CJS — the actual difference

CommonJS (CJS):
```js
const fs = require("fs");      // runtime, synchronous, can be conditional
module.exports = { foo };
```

ES Modules (ESM):
```js
import fs from "node:fs";      // resolved at parse time, async, static
export const foo = ...;
```

The differences that matter:

| Aspect | CJS | ESM |
| --- | --- | --- |
| When imports resolve | runtime | parse / link time |
| Conditional imports | `if (x) require(...)` works | only via dynamic `import()` |
| Top-level `await` | no | yes |
| `__dirname`, `__filename` | yes | no (use `import.meta.url`) |
| Default export | `module.exports = x` | `export default x` |
| Can a CJS require ESM? | no, must `await import()` | — |
| Can an ESM import CJS? | usually yes, with caveats | — |

OpenStatus is ESM throughout. You can see this in `package.json` files declaring `"type": "module"` and using `exports` maps. If you ever see `Cannot use import statement outside a module`, you're running a `.js` file as CJS — fix the file extension to `.mjs` or set `"type": "module"`.

## Reading `package.json` like a senior

Open [`packages/services/package.json`](../../packages/services/package.json). The interesting fields:

- `"name"` — what you import as.
- `"version"` — Semver. Major = breaking, minor = additive, patch = fix.
- `"main"` (CJS) and `"module"` (ESM hint) — legacy entry points.
- `"exports"` — **the modern, authoritative entry point map**. If `exports` is present, `main` is ignored by Node ≥ 12.7.
- `"type"` — `"module"` makes `.js` files ESM.
- `"sideEffects": false` — tells bundlers "tree-shaking is safe; importing me doesn't *do* anything as a side effect."
- `"peerDependencies"` — dependencies the consumer must provide. React, Drizzle, Zod are all common peers.
- `"dependencies"` — actually installed for consumers.
- `"devDependencies"` — only installed for the package's own development.

### The `exports` field

```jsonc
"exports": {
  ".": "./src/index.ts",
  "./status-report": "./src/status-report/index.ts",
  "./monitor": "./src/monitor/index.ts"
}
```

That's what enables `import { createStatusReport } from "@openstatus/services/status-report"`. Without `exports`, consumers could reach into any file. With `exports`, the package controls what's importable. This is encapsulation enforced by the package manager.

**Senior aside:** *If a refactor requires changing what's exported, change the `exports` map first. The map is the contract; the file structure is implementation. Touching files without updating the map either breaks consumers or accidentally exposes new internals.*

## pnpm workspaces (and why npm/yarn classic do this worse)

OpenStatus uses pnpm. Three things to know:

1. **One global content-addressed store.** Every package version exists on disk once. `node_modules` is full of symlinks into the store. No more 4GB `node_modules` for every project.
2. **Strict resolution.** A package can only `import` what it explicitly depends on. Try to import a transitive — pnpm refuses. This catches "works locally because some unrelated dep happens to bring it in" bugs at install time. npm and yarn-classic *don't* enforce this, which is why migrating away from them often surfaces hidden coupling.
3. **Workspaces (`pnpm-workspace.yaml`)** let multiple packages in one repo depend on each other by name. From [`pnpm-workspace.yaml`](../../pnpm-workspace.yaml):
   ```yaml
   packages:
     - "apps/*"
     - "packages/**/*"
     - "packages/config/*"
   ```
   And from the root [`package.json`](../../package.json):
   ```json
   "workspaces": ["apps/*", "packages/**/*", "packages/config/*"]
   ```

When `apps/dashboard` does `import { db } from "@openstatus/db"`, pnpm resolves that to the *local source* in `packages/db` via a symlink — no publish step needed.

### `workspace:*` versions

In a workspace consumer's `package.json`:
```json
"dependencies": {
  "@openstatus/db": "workspace:*"
}
```

That says "use the workspace version, whatever it is." pnpm rewrites this at publish time to a real version. The `workspace:*` syntax is the canonical way to declare an in-monorepo dep.

## Turborepo — what it actually does

Turborepo is a build orchestrator and remote cache. It doesn't build anything itself; it runs *your* scripts in a graph-aware order, caches their outputs by content hash, and shares the cache across machines (optionally).

From [`turbo.json`](../../turbo.json):
```jsonc
"dx": {
  "cache": false,
  "dependsOn": ["seed"]
},
"seed": {
  "dependsOn": ["migrate"],
  "cache": false
},
"migrate": {
  "dependsOn": ["env"],
  "cache": false
},
```

When you run `pnpm dx`, Turbo computes: `env` → `migrate` → `seed` → `dx`, runs them in order, in parallel where it can.

The cache magic: if you run `pnpm build` twice with no file changes, the second run is instant because Turbo replays cached outputs. With **remote caching** (Turbo's hosted service or self-hosted), CI builds become incremental across the whole team.

**Senior aside:** *A cache is correct only if the cache key includes every input. Forgetting to add `inputs: ["src/**", "tsconfig.json"]` to a task means Turbo will gleefully serve a stale build. When you add a task to `turbo.json`, list its inputs and outputs explicitly.*

## Module resolution in TypeScript

Two settings in `tsconfig.json` you need to know:

- `"moduleResolution": "bundler"` (or `"node16"` / `"nodenext"`) — controls how `import` strings become file paths. `bundler` is permissive and matches what Vite/Webpack/esbuild do.
- `"paths"` — workspace path aliases. `@/components/x` in the dashboard maps to `apps/dashboard/src/components/x` via a `paths` entry.

When you see `import { Foo } from "@/lib/trpc/client"` in `apps/dashboard`, the `@/` prefix is a `tsconfig.json` paths alias — it's not standard ESM. Other tools (Next.js, Biome, your IDE) read the same `tsconfig.json` so the alias works across the toolchain.

## Edge runtime, Node runtime, and why some imports break

OpenStatus deploys some tRPC routers to the Edge runtime (a constrained, V8-isolate environment) and others to a Node Lambda. The split is here:

- Edge: [`packages/api/src/edge.ts`](../../packages/api/src/edge.ts) — status reports, monitors, pages, etc.
- Node Lambda: [`packages/api/src/lambda.ts`](../../packages/api/src/lambda.ts) — Stripe, email, API keys, integrations, blob.

Edge can't `import` from `node:*` modules (`node:fs`, `node:net`, `node:crypto` partially), can't use long-lived state, has a tiny bundle budget. Stripe and Resend (email) ship Node SDKs, so they live in the Lambda router.

The link selector lives in [`apps/dashboard/src/lib/trpc/shared.ts:38-77`](../../apps/dashboard/src/lib/trpc/shared.ts#L38-L77):

```ts
const lambdas = ["stripeRouter", "emailRouter", "apiKeyRouter", "integrationRouter", "blob"];
// ...
const endpoint = lambdas.includes(path[0]) ? "lambda" : "edge";
```

That's it — a hardcoded prefix list. Add a new procedure that needs Node? You must add its router name to this array *and* register it in `lambdaRouter`. Forget the latter and the client routes to a non-existent endpoint; forget the former and you'll get an Edge runtime error like `Module not found: node:tls`.

## Biome, ESLint, and formatting

OpenStatus uses **Biome** ([`biome.jsonc`](../../biome.jsonc)) — a single fast tool that replaces Prettier + ESLint. Run `pnpm lint` and `pnpm format`. Biome's defaults are sane; don't fight them.

**Senior aside:** *The fastest team I know has one rule: no PR merges with a lint warning. Warnings rot into errors. Treat the linter as a free pair of eyes, not an annoyance.*

## Pitfall checklist

1. **`import` from a transitive dep.** pnpm catches it; npm doesn't. Always declare deps explicitly.
2. **CJS file in an ESM package.** Either rename to `.cjs` or convert. Don't fight the runtime.
3. **Forgetting to add a router to `lambdaRouter` after adding it to `lambdas`.** 404s in production.
4. **Importing `node:fs` in an Edge file.** Bundler will inline it; runtime will crash.
5. **Forgetting an `exports` entry.** New file works locally if consumed inside the package, fails when consumed externally.
6. **`workspace:*` in a published package.** pnpm rewrites it on publish, but if you bypass publish (e.g., tarball), consumers will see a literal `workspace:*` and fail.
7. **`type: "module"` in a package with `.ts` files compiled to CJS.** Mixed module formats; debugging this is a special hell.
8. **Stale Turbo cache.** Add inputs to `turbo.json`. Verify with `turbo run build --dry-run`.
9. **Circular imports.** ESM handles them by exposing the partially-initialized module; you'll get `undefined` values that look like bugs.
10. **Side-effectful imports.** If a module *does* something at import time (registers a hook, mutates a global), `sideEffects: false` is a lie and tree-shaking will remove your registration. Declare sideful files in `sideEffects: ["./register.ts"]`.

## Drills

- Open the root [`package.json`](../../package.json). Find three workspace-level scripts and explain when you'd run each.
- Open `apps/dashboard/package.json` and `packages/db/package.json`. Identify each declared dep as "really used," "peer," or "dev only." For one of them, predict what happens if you remove it.
- Find a `workspace:*` reference. Trace it through `pnpm install`'s symlinks to the source.
- Add a fake router to `lambdas` in [`apps/dashboard/src/lib/trpc/shared.ts`](../../apps/dashboard/src/lib/trpc/shared.ts) and call a procedure. What error do you get? Now register it in [`packages/api/src/lambda.ts`](../../packages/api/src/lambda.ts). Different error?
- Run `pnpm why some-package-name` and read the result. What's the dependency tree saying?
