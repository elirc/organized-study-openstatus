# 01 — Code Review Mindset

A code review is two jobs: catch bugs, and *teach the team*. Reviews that only catch bugs make the same bugs come back. Reviews that teach raise the bar for the whole team.

## The senior review checklist

For every PR, walk these in order. They take 30 seconds for a 1-line change, 30 minutes for a thousand-line change. Don't skip steps for "small" PRs — the small ones bite.

### Layer 1: Does it work?

- Read the PR description. If there's no description or "fix bug," ask for one before reviewing.
- Look at the test diff. New behavior should have new tests. If not, why not?
- Pull the branch and run it. (Optional for tiny changes; required for anything user-visible.)
- Read the diff in order: schema → service → router → UI. The order forces you to see the boundary.

### Layer 2: Is it correct?

- Are workspace filters present in every query?
- Does every write start with `requireScope`?
- Is every mutation wrapped in `withTransaction`?
- Does every mutation emit an audit row inside the tx?
- Are inputs validated at the service boundary, not just at the form?
- Are errors mapped through the adapter?

### Layer 3: Will it stay correct?

- Are there tests for the rejection paths (cross-workspace, scope mismatch, validation failure)?
- Are type assertions justified with comments?
- Is there a clear path from "user reports a bug" → "find the code that handles it"?
- Could a junior change this code without breaking the invariants?

### Layer 4: Is it a good citizen of the codebase?

- Does it follow the existing patterns (one service file per verb, adapter for ctx, etc.)?
- Does it duplicate something that already exists?
- Does it leave dead code, commented-out blocks, or speculative abstractions?
- Are new dependencies justified?

### Layer 5: Is it kind?

- Does it leave the codebase easier or harder to read for the next person?
- Are commit messages and PR description focused on *why*, not just *what*?
- Are there comments explaining non-obvious choices?

## How to give review that teaches

There are roughly three kinds of comments:

### Blocking ("change before merge")

Use sparingly. Reserve for correctness, security, and significant design issues. Be specific:

> Bad: "This looks wrong."
>
> Good: "The `WHERE` clause is missing `workspaceId`. As-is, this query returns rows from every workspace — see [`packages/services/src/status-report/list.ts:172-181`](../../packages/services/src/status-report/list.ts#L172-L181) for the pattern."

Cite the pattern, not just the bug. The reader walks away knowing both what to fix and why.

### Suggestions ("consider")

For style, naming, light refactors. Mark them as such — `nit:`, `consider:`, `suggestion:`. The author should feel free to defer.

> "nit: this could use `Object.fromEntries` to drop the intermediate accumulator. Not blocking."

### Questions ("help me understand")

When something looks off but you're not sure if you're missing context. Lead with curiosity:

> "I expected this to invalidate `monitor.list` too — does the existing cache already cover that, or is it intentional?"

Questions are a kindness; they create space for the author to teach the reviewer something. Sometimes the answer is "you're right, I missed that" — that's a win. Sometimes it's "it's fine because of X" — also a win, and you learned something.

## What not to do

- **Don't bikeshed.** Naming and tab width matter, but they're not the most important thing on the PR. Comment once, accept whatever the author chose.
- **Don't rewrite the PR in comments.** "Why didn't you do X, Y, Z instead?" overwhelms. Pick the one most important suggestion.
- **Don't moralize.** "You should have known better" doesn't teach; it shames.
- **Don't review while irritated.** Walk away first. Tone shows up in writing.
- **Don't approve out of social pressure.** If you're not comfortable, say so.

## What to look for that juniors miss

These are the bugs that pass review by juniors but fail review by seniors. Memorize the shapes.

- **Implicit dependency on render order.** "This works because the parent always renders before me."
- **Stale closure** in an effect dep array.
- **N+1** in a list path.
- **Missing audit** on a mutation that has a `WHERE workspaceId`.
- **Side effect after** the transaction commits, with no compensation on failure.
- **Cross-tenant** check missing on a *referenced* ID, not just the primary one.
- **Optimistic update** without rollback on error.
- **Sentinel limits** that work in dev (small data) but fail in prod.
- **`as any` or `!`** on a value that can actually be null/undefined.
- **`useEffect` doing what `useMemo` should** (deriving state from props).

If you spot one, leave a *specific* comment with a *specific* fix path.

## What to look for that seniors miss (yes, you'll miss things)

- **Layered design that's hard to follow** if you don't know the conventions. The PR is correct; the codebase is hostile. Add a comment or docs link.
- **Performance regressions** invisible in dev. Did the author profile? Did you?
- **Migration risk**: data migrations that can't be rolled back. Always ask for the rollback plan.
- **Security surface area**: new public endpoints, new file uploads, new dependencies. Run the security checklist.

## Receive review with grace

The flip side: when *you're* the author, accept review the same way you'd give it.

- **Read every comment fully before responding.** Don't get defensive on a half-read line.
- **Distinguish opinion from fact.** If a reviewer is wrong, explain politely. If they're right, accept it.
- **Push back when you should.** "I considered that approach but chose this because X" is a totally valid response.
- **Don't take "nit:" comments personally.** They're suggestions; you can defer.
- **Thank the reviewer.** Reviews take real time. Acknowledge it.

## Drills

- Open a recent PR (yours or someone else's). Walk it through the 5-layer checklist. Write the comments you'd post — even if you don't post them.
- Find a PR comment that was given poorly (vague, demanding, moralizing). Rewrite it kindly. Notice how much harder kindness is than venting.
- Pretend a junior on your team submitted [`packages/services/src/status-report/create.ts`](../../packages/services/src/status-report/create.ts) as a brand-new PR. Write your review. Compare what you'd say with what the existing code already does well.
- Find one place in your own past code where you'd give yourself blocking review today.
- Practice reviewing without nitpicking. Find a PR you'd normally have 10 comments on; pick the 3 most important.
