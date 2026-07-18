# Code Review Mindset

## The five layers (review in this order)

1. **Does it work?** — happy path, stated acceptance criteria.
2. **Is it correct?** — edge cases, concurrency, failure modes. (Most review value lives here.)
3. **Will it stay correct?** — tests pinning the invariant, not just the instance; types that make the wrong thing unrepresentable.
4. **Does it fit?** — house patterns: would a reader of [create.ts](../../../packages/services/src/monitor/create.ts#L16-L63) recognize this code as the same codebase?
5. **Is it kind to future maintainers?** — naming, comment discipline (why-only, per [CLAUDE.md](../../../CLAUDE.md)), blast radius of the diff.

Junior reviewers live at layer 1 and nitpick layer 5 syntax. Mid-level owns layer 2–3. Senior adds "should this change exist at all, and is there a smaller version?"

## The repo-specific checklist

For any services-layer mutation PR:
- [ ] `requireScope(ctx, "write")` first line ([require-scope.ts:21-23](../../../packages/services/src/auth/require-scope.ts#L21-L23))
- [ ] Input parsed at verb top; `withTransaction`; all writes on `tx`
- [ ] Workspace filter on every read/write (or fetch-or-throw helper)
- [ ] `emitAudit` in-tx; `before`+`after` for updates; secrets stripped; action added to the validation union
- [ ] Tests: happy + audit + cross-tenant + read-only-actor + quota (where gated)
- [ ] No `node:*` imports (Edge invariant)

For pipeline PRs (workflows/checker): idempotency key identified; retry safety argued; wide-event enrichment for new outcomes; no external calls inside transactions.

For UI PRs: loading/empty/error distinguished; cache invalidation named; URL vs component state justified.

For public-API PRs: status codes and shapes unchanged or versioned; OpenAPI schema updated with the handler.

## Comment language that works

Formula: **observation (anchored) → consequence → question or suggestion.** Kind = specific + attributed-to-code-not-author.

> **Blocking:** This `sendDiscord` runs inside `withTransaction` — a slow provider holds the tx open, and a provider failure rolls back the maintenance row. Could we move it after commit (and note: the repo currently lacks an outbox, so a comment on the crash window would help)?

> **Important:** `insertNotificationTrigger`'s catch treats *any* error as "already sent" (alerting.ts:131–139 has the same shape) — a connection blip would suppress a page. Can we match on the unique-violation error specifically?

> **Optional (non-blocking):** `notifType` branches are identical except the method name — fine to land as-is; happy to pair on the typed-dispatch cleanup later.

Anti-patterns: "why didn't you just…", unanchored vibes ("this feels off"), rewriting the PR in comments, and silence on the good parts (name one thing that's well done — it calibrates trust for the criticism).

## Drill

Review your own last real PR (any project) against the five layers, in writing. Which layer did your self-review stop at? Then do [review kata 6](../04-code-reading-gym/04-review-katas.md) and compare your comment phrasing to the formula above.

## Interview angle

"Tell me about a review where you pushed back" — structure the answer as layer-2 finding + consequence + kind phrasing + outcome. Interviewers are listening for whether your pushback was about *invariants* or *taste*; this repo trains you to make it invariants.
