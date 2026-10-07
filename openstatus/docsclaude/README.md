# docsclaude — Junior to Senior, Fast, in JS/TS/React/Node

This is a learning curriculum that uses the OpenStatus codebase as a *lab*, not a *subject*. The goal is to make you a senior software engineer in modern JS/TS/React/Node. OpenStatus is one of the best public examples you can study because it pairs a typed full-stack TypeScript app (Next.js + tRPC + Drizzle + React Query) with a real services-layer architecture, multi-tenant authorization, audit logging, and a public read model — all the things a senior engineer is expected to reason about. We'll keep returning to it as a worked example.

## What's different about this guide

Most onboarding docs teach you *where things are*. That gets you to "junior who can find the file." Senior engineers need three additional capabilities:

1. **Language depth.** You should know *why* `useEffect` re-runs, *why* `as const` matters, *why* a `for await` loop serializes work that `Promise.all` parallelizes, *why* tRPC's type inference even works.
2. **Architectural taste.** You should be able to look at a router file and tell whether the team will regret it in six months — and why.
3. **Quality engineering.** You should be able to debug a flaky test, profile a slow render, and write a security checklist for a feature before it ships.

This curriculum is structured around those three. Every module mixes **concept → OpenStatus example → drill → pitfall checklist**. You don't just read; you do.

## How to use it

You can start anywhere — modules are independent. But the recommended fast path for a junior who wants to be senior-shaped within ~8 weeks is:

| Week | Module | Goal |
| --- | --- | --- |
| 1 | `01-javascript-typescript-core` | Stop fighting the language. Build correct mental models for closures, async, modules, and TypeScript inference. |
| 2 | `02-react-deep-dive` | Understand the render model, hooks, server components, and React Query at a level where bugs are obvious, not mysterious. |
| 3 | `03-node-server-mastery` | Know what runs where (Node vs Edge), how requests become DB rows, and why tRPC's types travel end-to-end. |
| 4 | `04-architecture-and-patterns` | Read a feature and see the *layers*, not the files. Learn to enforce multi-tenant isolation, audit, and transactions by reflex. |
| 5 | `05-quality-engineering` | Tests, debugging, performance, and security as everyday habits, not afterthoughts. |
| 6 | `06-practice-tracks` (reading + modify) | Apply the above on real diffs. |
| 7 | `06-practice-tracks` (build) | Ship a non-trivial feature end-to-end. |
| 8 | `07-career-skills` | Reviews, PRs, RFCs, interviews. Translate skill into impact. |

If you're already comfortable with the language and just want the codebase, skim modules 1–3 and dive into 4. If you only have a weekend, do `00-fast-track.md`.

## Conventions in this guide

- **File anchors** look like [`packages/services/src/status-report/create.ts:24-99`](../packages/services/src/status-report/create.ts#L24-L99). Open them as you read. Reading commentary about code you haven't opened is wasted time.
- **Illustrative code** is labeled. Real code from the repo is quoted with a file path; everything else is a teaching sketch.
- **Senior-mindset asides** are short, italicized notes about what a senior would notice that a junior wouldn't — often the thing that separates levels.
- **Drills** are open-ended. They aren't quizzes with answer keys; the goal is to build the reflex of asking the question.
- **Pitfall checklists** are concrete bug patterns. If you can explain why each one is a bug, you've earned the section.

## Module index

| # | Folder | What you'll learn |
| --- | --- | --- |
| 01 | [`01-javascript-typescript-core/`](01-javascript-typescript-core/README.md) | The JS runtime model, async semantics, the module system, and TypeScript from inference through advanced types. |
| 02 | [`02-react-deep-dive/`](02-react-deep-dive/README.md) | How React actually renders, hooks from first principles, state architecture, RSC, and React Query / tRPC client. |
| 03 | [`03-node-server-mastery/`](03-node-server-mastery/README.md) | Node vs Edge, HTTP handlers, Drizzle, tRPC internals, auth and multi-tenancy. |
| 04 | [`04-architecture-and-patterns/`](04-architecture-and-patterns/README.md) | Layering, validation, transactions, derived data, and how to recognize good architecture in unfamiliar code. |
| 05 | [`05-quality-engineering/`](05-quality-engineering/README.md) | Testing strategy, writing tests in this repo, debugging by deduction, performance, and security. |
| 06 | [`06-practice-tracks/`](06-practice-tracks/README.md) | Reading katas, modify tickets, build projects, and senior-level refactor/design exercises. |
| 07 | [`07-career-skills/`](07-career-skills/README.md) | Code review, PR writing, RFCs, and interviews — using this codebase as material. |
| ⨯ | [`CRITIQUE-codex-vs-claude.md`](../CRITIQUE-codex-vs-claude.md) (moved to the repo root in this snapshot) | Where the prior `docs/` (Codex) effort succeeded and where it falls short — included so you can read the docs critically and learn from both. |

## The senior mindset, in one paragraph

A junior engineer asks "how do I make this work?" A mid asks "is this the right pattern?" A senior asks "what does this commit me to, and who pays the cost?" — and then writes code that minimizes the cost. Every module here is trying to nudge you toward the last question. The fastest way to grow is to read code constantly, write code daily, and explain what you read out loud to someone (or to a rubber duck). This guide gives you targets for all three.
