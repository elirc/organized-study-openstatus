# 08 — Interview Prep

First-class module: everything else in this curriculum feeds it. Target: **mid-level fullstack JS interviews**, answered with concrete evidence from a production-grade codebase instead of memorized trivia.

## How mid-level loops are structured (typical)

1. **Recruiter/hiring-manager screen** — your narrative + 2–3 crisp project stories.
2. **Technical deep-dive** — JS/TS/Node + your stack ([01](01-js-ts-node-deep-dive.md), [02](02-frontend-framework-questions.md), [03](03-api-and-data-modeling-questions.md)).
3. **Practical coding** — build/extend a small feature; your habits from [05-quality-engineering](../05-quality-engineering/README.md) are the differentiator (states handled, tests named).
4. **System design (light)** — [04-system-design-from-this-repo.md](04-system-design-from-this-repo.md).
5. **Debugging / code review round** (increasingly common) — [05-debugging-and-code-review-rounds.md](05-debugging-and-code-review-rounds.md).
6. **Behavioral** — [06-behavioral-star-stories.md](06-behavioral-star-stories.md).

Two weeks out? Go straight to [07-two-week-cram-plan.md](07-two-week-cram-plan.md).

## The golden rule

Every answer = **concrete example + tradeoff + failure mode.** Not "zod validates input" but "in the codebase I studied, input is parsed at the wire *and* re-parsed in the service layer because five transports call the same verbs — the cost is duplicated schemas, which they manage by keeping wire and domain schemas deliberately separate." That sentence is the difference between junior and mid in an interviewer's notes.

## Using this repo as your portfolio of talking points

You didn't just read OpenStatus — you can discuss: a services layer with fail-closed audit, three idempotency implementations, quorum-based alerting, multi-tenant isolation machinery, two data planes, API-key RBAC with scopes, wide-event observability, and the honest gaps (races, at-most-once alerting, missing member roles). Question cards across files 01–03 are ≥60% anchored to real lines here, so your practice answers are *automatically* evidence-based.

If you've also completed tickets/projects from [06-contribution-practice](../06-contribution-practice/README.md), each carries a ready-made story — the mapping lives in [06-behavioral-star-stories.md](06-behavioral-star-stories.md).
