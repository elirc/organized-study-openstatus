# Tooling & Build System

## pnpm workspaces: the dependency graph is the architecture

Mental model: `workspace:*` deps ([db package.json deps like `@openstatus/assertions`](../../../packages/db/package.json)) create compile-time edges between packages; pnpm symlinks them, so imports resolve to source-in-repo. The workspace globs live at [package.json:38](../../../package.json#L38).

Sharp edge worth quoting: pnpm's strict node_modules can produce **two instances of the same class** through different resolution paths — the documented reason `isTx` avoids `instanceof` ([context.ts:33-38](../../../packages/services/src/context.ts#L33-L38)). When you see "works in app A, fails in app B" with `instanceof` involved, suspect module duplication.

## Turborepo: cached task graph

`turbo run build` executes each package's `build` in topological order (`dependsOn: ["^build"]`), caching outputs keyed by inputs+env. Read [turbo.json](../../../turbo.json) closely:
- `dx → seed → migrate → env` encodes setup order as a graph — running `pnpm dx` is running documentation.
- `test` and `dev` are `cache: false` — tests hit a shared DB (unsound to cache), dev is a long-running process.
- The `build.env` allowlist is a **cache-correctness** device: env vars listed there invalidate the cache when they change; unlisted vars are invisible to hashing. A build that reads an unlisted env var is a stale-cache bug waiting to happen.
- Filtered dev: `dev:dashboard` runs only dashboard+db ([package.json:15-17](../../../package.json#L15-L17)) — monorepo ergonomics 101.

## Biome (+ oxlint experiments)

One tool for lint+format ([package.json:7-14](../../../package.json#L7-L14)), config in `biome.jsonc`. Suppressions are inline and visible — e.g. the deliberate `biome-ignore lint/correctness/useExhaustiveDependencies` at [client.tsx:81](../../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L81). The parallel `lint:oxc*` scripts show a team *evaluating* a faster linter without committing — tool migrations run both in parallel first.

## drizzle-kit: schema-as-code migrations

Schema TS files are the source of truth; `pnpm generate` (in packages/db) diffs them into SQL migrations; `pnpm migrate` applies via [migrate.mts](../../../packages/db/src/migrate.mts); `pnpm push` (dev only) skips migration files. CI order: migrate → seed → test ([test.yml:51-60](../../../.github/workflows/test.yml#L51-L60)).

Safe-change discipline (details in [03/02-data-model](../03-architecture-and-patterns/02-data-model-and-persistence.md)): additive columns nullable-or-defaulted first; destructive changes need a deploy-order plan because three apps share this schema.

## CI as the source of truth for "what must pass"

[.github/workflows](../../../.github/workflows): `test.yml` (sqld + migrate + seed + turbo test), `lint.yml`, `go-tests.yml` (checker), `proto-check.yml` (ConnectRPC contracts), per-app deploys. When docs and CI disagree about how to run tests, **CI wins** — it's executable.

## Drills

1. Predict what `pnpm dev:web` starts, from [package.json:15](../../../package.json#L15) + turbo.json — note the extra `@openstatus/react#build` dependency for web dev in [turbo.json](../../../turbo.json).
2. You add `NEW_FLAG` read via `process.env` in the dashboard build. List each file you must touch for turbo caching to stay correct.
3. Explain why `test` being uncached means CI time grows linearly with test count, and what you'd need to make tests cacheable (hermetic DB per package).

## Interview angle

- "How do monorepos scale builds?" → task graph + content-addressed caching, with the env-allowlist correctness caveat (rarely known — differentiator).
- "How do you run DB migrations in CI/CD?" → this repo's migrate-seed-test order + [08/03 Q10](../08-interview-prep/03-api-and-data-modeling-questions.md).
- "Tool migration strategy?" → the Biome/oxlint parallel-run story.
