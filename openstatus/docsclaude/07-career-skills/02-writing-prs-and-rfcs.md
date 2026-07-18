# 02 — Writing PRs and RFCs

Your writing is your career. Code is read; PR descriptions are *forwarded*. RFCs decide direction for months. Senior engineers are good writers because they had to be.

## A good PR description

Three sections, each a few sentences:

### What changed

Not a file list — a behavior summary. "Status reports now persist a `severity` field, default `medium`. The dashboard form exposes it; the public feed renders a badge."

If the change is purely refactoring, say so. "Extracted the report enrichment into `enrichReportsBatch`. No behavior change."

### Why

The motivation, in the user's or the team's terms. "Users couldn't tell at a glance which incidents were the most serious; severity rolls up to the page status banner so customers see priority info." Or "the existing inline loop was an N+1 — a 200-row list took 4 seconds."

### How to test (or: what I tested)

If the change is risky: "I tested by … and confirmed …". Specific scenarios, not "I tested it."

If there's nothing to manually test: say so. "Pure refactor; existing tests cover this." That tells the reviewer what they don't have to recheck.

## A good PR is small

Median PR for a senior: 50-200 lines of diff. Outliers (refactors, migrations) might be larger. If your PR is 1000+ lines, ask: can this be split?

Split strategies:

- **Refactor first, feature second.** PR 1 reshapes the existing code. PR 2 adds the feature using the new shape. Reviewing the feature is easy because the refactor is already in.
- **Schema first, code second.** PR 1 adds the new column with a default. PR 2 uses it. PR 3 removes the default (if it was temporary).
- **Vertical slice.** Don't ship "the whole feature half-done across 20 files." Ship one path end-to-end (DB → service → router → UI), then iterate.

## When to write an RFC

Some changes are big enough that the PR review is the wrong place to discuss them. Sketch the plan first, get alignment, then build. Triggers:

- New service or app.
- Significant schema change.
- Cross-cutting concern (rate limiting, observability, auth model).
- A choice between two non-trivial approaches.
- A user-facing change that affects pricing or expectations.

OpenStatus doesn't have a formal RFC process in the OSS repo, but the `CLAUDE.md` files and inline comments serve a similar purpose for ongoing patterns.

## A good RFC

Five sections, in order:

### Problem

The user-facing or system-facing pain. Concrete. Quotes from real users or real incidents if you have them. *Not* a description of the solution — that comes later.

> "When subscribers are notified about an incident, the notification triggers from the dashboard's `onSuccess` callback after the create mutation. If the user closes the tab between the create returning and the notification firing, no email is sent. We've seen this happen at least 3 times in the past month based on support tickets."

### Goals and non-goals

What the RFC is *trying* to accomplish, and explicitly what it's *not* trying to accomplish. The latter prevents scope creep.

> Goals:
> - Notifications fire reliably after a status report is created, regardless of browser state.
> - Failed notifications are retried automatically.
>
> Non-goals:
> - Real-time delivery (we'll keep the current "best effort within seconds" SLA).
> - Per-subscriber configuration of channels (separate project).

### Proposal

The design. Diagrams help. Be specific about tables, columns, function names, file paths.

### Alternatives considered

The strongest section. List 2-3 alternatives you rejected and why. This shows you thought about it. It also pre-empts the "did you think of X?" comments.

> **Alternative 1: in-band synchronous send.** Pros: simplest. Cons: user waits for SMTP; partial failure mode (DB write succeeds, email fails, user sees success but no email). Rejected because partial failure is the bug we're trying to fix.
>
> **Alternative 2: Next.js `after()`.** Pros: cheap to add. Cons: still depends on Edge runtime keeping the function alive; no retry; errors are silent. Rejected because reliability is the goal.

### Migration plan

How do you ship this without breaking anything? What's the rollback if it goes wrong?

> Phase 1: add outbox table (no behavior change).
> Phase 2: start writing to outbox alongside existing path.
> Phase 3: enable worker to drain outbox; verify dispatches match the existing path's behavior.
> Phase 4: remove the dashboard `onSuccess` path.
>
> Rollback: at any phase, disabling the worker reverts to the previous behavior with no data loss.

## Writing for an audience

Two audiences for any RFC:

- **Reviewers** want to know "is this right? what did you miss?"
- **Future maintainers** want to know "why did we choose this?"

Optimize for the second. Reviewers move on; the doc stays. In two years, someone considering changing this will read the RFC and your "Alternatives considered" section is the gift that keeps giving.

## Decision logs (the lightweight RFC)

Sometimes you don't need a full RFC, but the decision still matters. A decision log entry:

```
2026-05-17 — Notification path: outbox

Problem: notifications fire from the dashboard onSuccess, lost on tab close.
Decision: introduce a notification_outbox table; worker drains.
Alternatives: in-band sync (slow), after() hook (unreliable).
Open questions: dispatcher retry policy, dead-letter handling.

Owner: @elirc3. Implementation tracked in #1234.
```

One page. Date it. Sign it. Keep them in a `docs/decisions/` folder or equivalent.

## Commit messages

The first line is the summary; max ~50 chars. Imperative mood ("add", not "added"). The body explains why.

Bad:
```
fix bug
```

Good:
```
fix(status-report): batch enrich relations to avoid N+1

The previous list path loaded updates and component associations
per-report, producing 1+2N queries for a list of N reports.
At the dashboard's effective sentinel limit (10_000), this
exceeded the libsql request budget on workspaces with > ~50
active reports.

This change collapses the per-row reads into three batched
queries (updates, associations, pages) regardless of list size.
```

The body is what tomorrow's debugger reads after running `git blame`.

## How to disagree productively

Senior engineers disagree often. The trick is doing it well:

- **Lead with the goal you share.** "I want fast deploys too. I'm worried that …"
- **State the concrete concern.** Not "this feels off" — "this introduces a 100ms latency to every request because of the auth re-fetch."
- **Propose an alternative or ask a question.** Disagreement without a path forward is just venting.
- **Accept defeat gracefully.** If the team decides against you, support the outcome. Resentment is corrosive.

## Drills

- Read a recent PR description in this repo. Rewrite it using the what/why/test template. Notice what was missing.
- Pick a hypothetical RFC topic from the [senior katas](../06-practice-tracks/04-senior-katas.md). Write the RFC. 1-3 pages.
- Find a commit message in your past that didn't explain "why." Rewrite it now (as if you were writing it then).
- Practice giving and receiving disagreement. Ask a teammate "tell me why my approach to X is wrong." Listen without defending.
