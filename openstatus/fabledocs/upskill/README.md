# OpenStatus Upskill Curriculum

A training lab built from this repository for one learner: a **junior fullstack JS engineer** (React/Node/TS CRUD background) who wants to reach mid-level fast, build senior judgment, and pass interviews for mid-level fullstack roles. Every page teaches two things at once: *where things live in this repo* and *the transferable pattern you can name in any codebase — and in an interview*.

## What this repo is

OpenStatus is an open-source **status page + uptime monitoring platform** (AGPL-3.0). It is a pnpm/Turborepo monorepo: a Next.js **dashboard** talks to a tRPC layer ([packages/api](../../packages/api)) whose routers stay thin and delegate all business logic to service verbs in [packages/services](../../packages/services); every mutation is workspace-scoped, transactional, and writes an **audit row** in the same transaction ([emit.ts](../../packages/services/src/audit/emit.ts#L102-L149)). A Hono **API server** ([apps/server](../../apps/server)) exposes a versioned REST API (API keys with read/write **scopes**), MCP tools, ConnectRPC, and a Slack agent. Monitors are probed by a **Go checker** ([apps/checker](../../apps/checker)) from 28 regions across three cloud providers; a **workflows** service ([apps/workflows](../../apps/workflows)) fans out probe jobs via GCP Cloud Tasks and turns failed probes into incidents and notifications. Relational data lives in **libSQL/Turso** via Drizzle; high-volume ping results go to **Tinybird**. Public **status pages** ([apps/status-page](../../apps/status-page)) are served per custom domain through a Next.js proxy that enforces password/SSO/IP access rules.

That combination — CRUD dashboard + distributed background pipeline + multi-tenant security + real reliability concerns — is why this repo is an unusually good interview gym.

## How to use it

- **One weekend** → [00-fast-track.md](00-fast-track.md). Run it, trace two flows, make one safe change.
- **Two weeks (interview soon)** → [08-interview-prep/07-two-week-cram-plan.md](08-interview-prep/07-two-week-cram-plan.md), which pulls from everything else.
- **Eight weeks (level up for real)** → modules 01→05 in order, doing every drill; then work tickets from [06-contribution-practice](06-contribution-practice/README.md).
- **Ongoing contribution** → live in [06-contribution-practice](06-contribution-practice/README.md) and [07-career-and-collaboration](07-career-and-collaboration/README.md); use 09-reference as your desk material.

### Recommended paths by profile

| You are | Do this |
| --- | --- |
| Brand-new junior | 00 → 01 (all) → 02 → 04 (drills) → 06/01-good-first-tickets |
| Junior who knows React/Node | 00 → 01/05-key-flows → 03 (all) → 05 → 06/02-mid-level-tickets |
| Mid-level, new to this repo | 01/01-system-map + 01/05-key-flows → 03/06-architecture-critique → 06/03-senior-build-projects |
| Senior doing architecture review | 03/06-architecture-critique → 09-reference/risk-register → 05/06-observability |
| **Interview in two weeks** | [08-interview-prep/07-two-week-cram-plan.md](08-interview-prep/07-two-week-cram-plan.md) — start today |

## Modules

1. **[01-codebase-cartography](01-codebase-cartography/README.md)** — maps: system, reading order, glossary, tooling, key flows.
2. **[02-stack-and-language-mastery](02-stack-and-language-mastery/README.md)** — TS/Node/Bun runtime models, Next/React/tRPC mental models, type-level contracts, build tooling.
3. **[03-architecture-and-patterns](03-architecture-and-patterns/README.md)** — boundaries, persistence, authz, async reliability, pattern catalog, critique.
4. **[04-code-reading-gym](04-code-reading-gym/README.md)** — annotation drills, trace tables, fake-code contrasts, review katas.
5. **[05-quality-engineering](05-quality-engineering/README.md)** — testing strategy and recipes, debugging method, performance, security, operations.
6. **[06-contribution-practice](06-contribution-practice/README.md)** — junior tickets, mid-level features, senior projects, refactor katas.
7. **[07-career-and-collaboration](07-career-and-collaboration/README.md)** — review mindset, PRs/RFCs, maintainer communication.
8. **[08-interview-prep](08-interview-prep/README.md)** — 40+ question cards (mostly repo-anchored), system design from this repo, timed rounds, STAR stories, cram plan.
9. **[09-reference](09-reference/verification-log.md)** — command cheatsheet, risk register, rubrics, verification log.

## Conventions used throughout

- **File anchors.** Every claim about this codebase links to a real file and line range, e.g. [create.ts](../../packages/services/src/monitor/create.ts#L16-L63). Line numbers were verified against the working tree on 2026-07-11; if the code moves, trust the symbol name over the number.
- **Verified vs inferred.** Commands are marked __verified__ only if they were actually run while writing these docs; otherwise __inferred__ (from manifests and CI). See [09-reference/verification-log.md](09-reference/verification-log.md) — almost everything here is inferred because nothing was executed.
- **Fake code is labeled.** Any snippet not from the repo starts with `// Illustrative fake code: not from this repo`.
- **Suspicions are labeled.** "Investigate" / "possible risk" means exactly that — not a confirmed bug.
- **Drills come with self-grading rubrics.** Grade yourself Basic / Solid / Strong honestly; the deltas between those levels are the actual curriculum.
- **Interview angle.** Wherever a concept is common interview material, the page says so and contrasts junior/mid/senior answers.

## Senior vocabulary (also interview vocabulary)

You will see these words used precisely: **invariant** (a condition that must always hold, e.g. "every mutation emits an audit row in the same transaction"), **boundary** (where one component's responsibility ends), **contract** (the agreed shape/behavior at a boundary), **ownership** (who is allowed to change a piece of state), **idempotency** (safe to run twice), **isolation** (tenant A can never read tenant B), **authorization** (is this actor allowed), **consistency** (does the data agree with itself), **latency**, **observability** (can you tell what happened from the outside), **migration/rollback**, **blast radius** (how much breaks if this goes wrong). Each is defined again in context the first time a module leans on it.

## The mindset ladder

- **Junior asks:** "How do I make it work?"
- **Mid-level asks:** "Is this the right pattern? What breaks it?"
- **Senior asks:** "What does this commit us to, who pays the cost, and how do we reduce risk?"

Interviews for mid-level roles are testing exactly the second and third questions. When a page here pushes you past "it works" into tradeoffs and failure modes, that is the interview prep — the two are the same skill.
