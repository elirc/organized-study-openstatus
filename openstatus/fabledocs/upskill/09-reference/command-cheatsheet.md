# Command Cheatsheet

**Nothing here was executed while writing these docs** — every command is __inferred__ from [package.json](../../../package.json), [turbo.json](../../../turbo.json), per-package manifests, [.github/workflows](../../../.github/workflows), [README.md](../../../README.md), [DOCKER.md](../../../DOCKER.md), and [CLAUDE.md](../../../CLAUDE.md). CI-exercised commands are marked; those are as close to verified as reading gets. Update this file's markers as you run things.

## Setup

| Command | Where | Notes | Status |
| --- | --- | --- | --- |
| `pnpm install` | root | pnpm 10.26.0 pinned | inferred (CI-exercised) |
| `cp .env.docker.example .env.docker && docker compose up -d` | root | full stack; dashboard :3002, status pages :3003 | inferred |
| `turso dev --db-file openstatus-dev.db` | root | local libSQL on :8080 | inferred |
| `pnpm dx` | root | env → migrate → seed chain (turbo graph) | inferred |
| `pnpm env` / `bun env.ts` | root/per-app | env validation/scaffolding | inferred |

## Dev servers

| Command | Runs | Status |
| --- | --- | --- |
| `pnpm dev:dashboard` | dashboard (+db) — Next+Turbopack on :3000 | inferred |
| `pnpm dev:status-page` / `pnpm dev:web` | status-page / marketing (+db) | inferred |
| `bun run dev` in `apps/server` | Hono API, `--hot`, :3000 | inferred |
| `bun run dev` in `apps/workflows` | workflows Hono app | inferred |
| `just` targets / `go run ./cmd/server` in `apps/checker` | Go checker (see its justfile) | inferred |

## Test

| Command | Scope | Status |
| --- | --- | --- |
| `pnpm test` | root — turbo fan-out, `cache: false` | inferred (CI-exercised: [test.yml:60](../../../.github/workflows/test.yml#L60)) |
| `cd packages/db && pnpm migrate && pnpm seed` | test-DB prep (CI order) | inferred (CI-exercised) |
| `cd packages/services && bun test` | services suite | inferred |
| `bun test src/monitor` / `bun test --test-name-pattern "audit"` | targeted | inferred |
| `cd apps/server && bun test` | REST/MCP/slack routes | inferred |
| `cd apps/checker && go test ./...` | Go | inferred (CI-exercised: go-tests.yml) |

## Lint / format / types

| Command | Notes | Status |
| --- | --- | --- |
| `pnpm lint` | biome lint | inferred (CI lint.yml) |
| `pnpm format` | biome format+check --write | inferred |
| `pnpm lint:turbo` | per-package lint tasks | inferred |
| `pnpm tsc` (root) / `pnpm tsc` in dashboard/server | typecheck (`--noEmit` in apps) | inferred |
| `pnpm lint:oxc` / `format:oxc` | experimental oxlint/oxfmt | inferred |
| `npx knip` | dead-code scan (knip.ts config) | inferred |

## Database

| Command | Where | Notes | Status |
| --- | --- | --- | --- |
| `pnpm generate` | packages/db | drizzle-kit migration from schema diff | inferred |
| `pnpm migrate` | packages/db | applies via `bun src/migrate.mts` | inferred (CI-exercised) |
| `pnpm push` | packages/db | schema push, dev only | inferred |
| `pnpm seed` | packages/db | seeds fixture workspaces (id 1 team…) | inferred (CI-exercised) |
| `pnpm studio` | packages/db | drizzle studio UI | inferred |

## Docker / deploy

| Command | Notes | Status |
| --- | --- | --- |
| `docker compose up -d` | full local stack ([DOCKER.md](../../../DOCKER.md)) | inferred |
| `docker compose -f docker-compose-lightweight.yaml up` | reduced stack | inferred |
| Fly deploys | per-app `fly.toml` (checker, workflows) via GH Actions | inferred |
| `ghcr.io/openstatushq/openstatus-*` images | prebuilt for self-host/Coolify | inferred |

## Gotchas (from manifests, not runs)

- Bun is required even for "Node" apps — db scripts (`migrate`, `seed`) and server/workflows run under Bun.
- `pnpm dev:*` filters auto-start the libSQL client on :8080 — kill a separately-running `turso dev` first (README notes this).
- Tests expect a **seeded** DB; a fresh DB without `pnpm seed` fails fixture lookups ([helpers.ts:107-119](../../../packages/services/test/helpers.ts#L107-L119) throws "Did you seed the db?").
- Windows: prefer Docker; several scripts assume POSIX shells.
