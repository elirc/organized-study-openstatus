# Runtime & Tooling Map

## Toolchain

| Concern | Tool | Where |
| --- | --- | --- |
| Package manager | pnpm 10.26.0 (`packageManager` pin) | [package.json](../../../package.json#L33) |
| Orchestration | Turborepo 2.6.3 | [turbo.json](../../../turbo.json) |
| Lint/format | Biome (primary) + oxlint/oxfmt (experimental scripts) + knip (dead code) | [package.json](../../../package.json#L7-L14), `biome.jsonc`, `knip.ts` |
| TypeScript | 5.9.3, shared configs in `packages/tsconfig` | — |
| Test runner | `bun test` (server, services); turbo `test` task is `cache: false` | [apps/server/package.json], [turbo.json](../../../turbo.json) |
| Schema/migrations | drizzle-kit (`generate`, `push`, `migrate` via `bun src/migrate.mts`) | `packages/db/package.json` |
| CI | GitHub Actions: `test.yml` (sqld container + migrate + seed + test), `lint.yml`, `go-tests.yml`, deploy workflows per app | [.github/workflows/test.yml](../../../.github/workflows/test.yml) |

All commands collected (with verified/inferred labels) in [09-reference/command-cheatsheet.md](../09-reference/command-cheatsheet.md).

## Runtime boundaries — the part juniors get wrong

There are **five** distinct JS/Go execution environments. Code must respect which one it runs in:

1. **Browser** — dashboard/status-page client components (`"use client"`). Example: [monitors list client.tsx](../../../apps/dashboard/src/app/(dashboard)/monitors/(list)/client.tsx#L1).
2. **Next.js server (Node)** — server components, route handlers, NextAuth.
3. **Next.js Edge** — the dashboard's tRPC edge router. Consequence: `packages/services` and `packages/api` must avoid `node:*` built-ins. This is a real, documented invariant — the hand-rolled `deepEqual` exists *because* `node:util.isDeepStrictEqual` blew up on Edge ([emit.ts:14-24](../../../packages/services/src/audit/emit.ts#L14-L24)). tRPC is split into `edge.ts` / `lambda.ts` routers merged in [root.ts](../../../packages/api/src/root.ts#L5).
4. **Bun servers** — `apps/server` (`bun run --hot src/index.ts`) and `apps/workflows`. Top-level `await` is fine here ([server index.ts:64](../../../apps/server/src/index.ts#L64)); Node APIs available (`AsyncLocalStorage` at [index.ts:1](../../../apps/server/src/index.ts#L1)).
5. **Go** — the checker. Its contract with the JS world is pure JSON-over-HTTP (Cloud Tasks payloads), typed on the TS side by `httpPayloadSchema` and friends ([cron/checker.ts:249-320](../../../apps/workflows/src/cron/checker.ts#L249-L320)) and by Go structs ([request/request.go](../../../apps/checker/request/request.go)).

Transferable: **"what runtime am I in?" is the first debugging question** in any modern fullstack repo. `window is not defined`, `node:fs not available on edge`, "works locally, fails on deploy" are all runtime-boundary bugs.

## Databases and data planes

- **libSQL/Turso over HTTP** — the system of record. One client in [db.ts](../../../packages/db/src/db.ts#L6-L12) (`drizzle-orm/libsql/http`). Local dev runs `turso dev --db-file openstatus-dev.db`; CI runs an `sqld` container ([test.yml](../../../.github/workflows/test.yml#L15-L27)).
- **Tinybird** — append-only analytics for probe results; typed pipes in [client.ts](../../../packages/tinybird/src/client.ts#L15-L120). Note the graceful **noop fallback** when no token/URL is configured (L20–35) — dev works without Tinybird.
- **Upstash Redis** — idempotency/dedupe keys for lifecycle workflows ([cron/monitor.ts:166-194](../../../apps/workflows/src/cron/monitor.ts#L166-L194)).
- **GCP Cloud Tasks** — the queue fabric: one queue per periodicity, plus `alerting` and `workflow` queues.

## Env vars (high level, no secrets)

The `build` task's env allowlist in [turbo.json](../../../turbo.json) is a good inventory: `DATABASE_URL`/`DATABASE_AUTH_TOKEN` (libSQL), `TINY_BIRD_API_KEY`, `UNKEY_TOKEN`, `UPSTASH_REDIS_*`, `STRIPE_SECRET_KEY`, `AUTH_SECRET` (NextAuth), `CRON_SECRET` (internal-pipeline shared secret), `SELF_HOST`. Each app validates its own env via an `env.ts` (t3-env style) — e.g. [apps/workflows/src/env.ts](../../../apps/workflows/src/env.ts). `pnpm env` / `bun env.ts` generates/validates local env files (__inferred__).

## Senior noticing

- `test` is `cache: false` in turbo — tests hit a real shared database (seeded once), so caching would be unsound *and* tests are not hermetic. That tradeoff shows up again in [05-quality-engineering/01-testing-strategy.md](../05-quality-engineering/01-testing-strategy.md).
- Deploys are per-app workflows (`deploy-checker.yml`, `deploy-workflows.yml`…) — the monorepo is a **deployment federation**, not a single release train. Ask: what happens when workflows deploys a payload change before the checker deploys? (Contract skew — see the risk register.)

## Drill

From a cold shell, write the exact command sequence to: (1) start the DB, (2) migrate + seed, (3) run only the services tests, (4) run only the Go tests. Check yourself against the [cheatsheet](../09-reference/command-cheatsheet.md). Strong = you can also explain what `pnpm dx` does *without running it*, purely from [turbo.json](../../../turbo.json).
