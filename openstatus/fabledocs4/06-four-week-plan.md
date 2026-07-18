# Four-week plan

One arc through all four fabledocs: **week 1 read, week 2 drill, week 3 build, week 4 perform.** Budget ~2 hrs/weekday, ~4 hrs one weekend day; flashcards (`05-flashcards.md`) are 10 min every single day from Day 3 onward, non-negotiable — they're the thread that keeps weeks from evaporating.

Every day ends with two written sentences in a running log: *what I now believe about this codebase that I didn't this morning* and *what I claimed today that I haven't verified.* The second list is Friday's homework, every week.

## Week 1 — Map (fabledocs/upskill + fabledocs2)

| Day | Work | Output |
| --- | --- | --- |
| Mon | `fabledocs/upskill/00-fast-track.md` + `01-codebase-cartography/01-system-map.md`; then fabledocs2 README + Part 1 (architecture). Get the repo running (`pnpm dx`) or document precisely where it fails. | System diagram drawn from memory, diffed against the real one |
| Tue | fabledocs2 Parts 2-3 (API, data) with the anchor files open in a second pane — read every cited region, not the docs alone | Part 2 & 3 gate questions answered in writing |
| Wed | fabledocs2 Parts 4-6 (testing, errors, observability). Start flashcards section A-C | Parts 4-6 gate questions; the alert-path crash table drawn by hand |
| Thu | `upskill/03-architecture-and-patterns/` 01-04: layers, data model, validation/auth, side effects — the deep versions of what fabledocs2 narrated | The side-effect map reproduced from memory with durability column |
| Fri | `03/06-architecture-critique.md` + `09-reference/risk-register.md`. **Before reading**: write your own top-3 risk list, then diff | Your risk list vs the repo's, disagreements argued in writing |
| Sat | Verification day: the week's "claimed but unverified" list — run the code, read the lines, resolve each item | Every claim marked ✓/✗/still-open with evidence |

## Week 2 — Drill (fabledocs4 fire drills + SQL labs)

| Day | Work | Output |
| --- | --- | --- |
| Mon | Fire drill 1 (alert never sent) + drill 4 (flaky quota tests) — one distributed, one testing | Two incident logs, self-graded |
| Tue | SQL labs 1, 2, 5 (tenancy, IDOR probe, audit forensics) against a seeded local DB | Queries + EXPLAIN output saved |
| Wed | Fire drill 2 (silent pipeline) + drill 3 (quorum flap). These are the hard pair — allow the full 2×90 min | Differential diagnosis table; interleaving trace |
| Thu | SQL labs 3, 6, 7 (quorum count, serialized-column tax, timestamp hunt — lab 7's trace feeds Week 3) | The `getScheduledTime` numeric trace, verdict either way |
| Fri | Fire drills 5-6 (status-page leak, ghost probes) — both end in a *definitive code answer* to a question the docs left open | Two resolved bug-suspicions with file:line evidence |
| Sat | SQL labs 4 & 8 (NOT NULL migration on a branch; two-plane UI state machine) + flashcard catch-up (should be through section F) | Migration branch with green tests |

## Week 3 — Build (Lab 04 + capstone)

The week that separates "studied it" from "worked on it." If time forces a choice, cut everything else before cutting Milestone 5 — the failing-on-main test *is* the portfolio piece.

| Day | Work | Output |
| --- | --- | --- |
| Mon | Lab 04-A steps 1-2: seam inventory + clock injection + first-ever workflows test (pins the Lab 7 trace) | Green test file in apps/workflows |
| Tue | Lab 04-A steps 3-4: provider seam + the concurrency test; measure the R1 flake rate over 50 runs | The number. Write it down — it's an interview sentence |
| Wed | Capstone M1-M2: the RFC (both designs compared honestly) + additive migration with zod sibling | RFC a stranger could implement from |
| Thu | Capstone M3-M4: write path rework + the sweep endpoint | Provider failure now leaves a queryable `failed` row |
| Fri | Capstone M5: the three fault-injection tests, especially crash-after-insert-before-send failing on main | All green on branch; the on-main failure screenshot/output saved |
| Sat | Buffer + Lab 04-B (Go checker: trace, assertion mapping, one Go test). If the week slipped, capstone wins and Go moves to Week 4 | `go test ./...` green with your new case |

## Week 4 — Perform (fabledocs3 + interview prep)

| Day | Work | Output |
| --- | --- | --- |
| Mon | fabledocs3: read all 10 ADRs cold and — before opening `interview-translation.md` — record yourself giving 90-second answers for ADRs 002, 004, 009 | Recordings; then diff against the written versions |
| Tue | Remaining 7 ADR spoken answers, each timed. Cut every one that runs past 100 seconds. Practice each follow-up answer | 10 answers ≤ 90s |
| Wed | Rewrite the ADR-009 answer to feature **your capstone**: "…and I fixed it: here's the design, the tradeoff, and the test that fails on main." Do the same for ADR-010 with Lab 04-A and your measured flake rate | The two upgraded stories — these are your best material |
| Thu | Mock system-design round: `upskill/08-interview-prep/04-system-design-from-this-repo.md` — design "a monitoring product" out loud in 40 min without notes, then diff against the doc | Gap list |
| Fri | Mock debugging round: rerun fire drill 1 or 3 cold (no hints, 45 min timer) + `08-interview-prep/05-debugging-and-code-review-rounds.md` | S-grade or schedule a redo |
| Sat | Full flashcard sweep (all 60, ⚡ cards must be 100%); write the one-page "what I'd do owning this repo for 3 months" from memory, diff against `03/06`'s version | The one-pager — your closing-question ammunition |

## Exit criteria

You're done with this repo when all five are true: (1) all ⚡ flashcards cold; (2) fire drills 1 and 3 at S-grade without hints; (3) the capstone's crash test demonstrably fails on main and passes on your branch; (4) all 10 ADR answers spoken under 90 seconds with the wart included; (5) at least two of the docs' open questions (ghost probes, status-page data gating, the `getScheduledTime` unit bug, HTTP-driver isolation) resolved by *you*, with evidence, where the answer disagrees-or-agrees with the docs and you know which. That last one is the real graduation: the docs stopped being the authority and the code became it.
