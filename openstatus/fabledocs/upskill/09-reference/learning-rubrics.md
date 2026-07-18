# Learning Rubrics

Observable behaviors, not vibes. Grade yourself per skill: the level whose behaviors you *demonstrably* did this month (artifacts required), plus the interview-ready check.

## Codebase navigation

| Level | Observable behavior | Interview-ready when… |
| --- | --- | --- |
| Junior | Finds a named symbol with search; follows one file's logic | you can narrate any one key flow with files named |
| Mid | Locates the right layer for a change without search (schema vs service vs router vs UI); predicts a file's contents from conventions before opening it | you can answer "where would X live and why" for arbitrary X in < 30s |
| Senior | Maps an unfamiliar subsystem (e.g. `apps/server/src/routes/slack/`) in under an hour with an ownership/risk summary | you can teach the navigation *method* itself |

## Fullstack change execution

| Level | Observable | Interview-ready when… |
| --- | --- | --- |
| Junior | Ships a one-layer change with an existing-pattern test (tickets J1–J16) | you can walk through the diff and say what each hunk does |
| Mid | Ships a cross-layer change with design note, risk + rollback sections (M-tickets); both standard negative tests included without being told | your PR description alone convinces a stranger the change is safe |
| Senior | Ships an invariant-touching change behind a staged rollout with a kill switch (S-projects) | you can defend the *rejected* alternatives |

## Correctness & concurrency

| Level | Observable | Interview-ready when… |
| --- | --- | --- |
| Junior | Handles the error paths zod/types surface | you enumerate loading/empty/error states unprompted |
| Mid | Spots read-modify-write across awaits; names the idempotency key for any retried effect | you can explain R1 and R8 from memory with fixes |
| Senior | Chooses failure semantics deliberately (at-most vs at-least-once; fail-open vs fail-closed) and writes them down | you volunteer the crash-window analysis in design discussions |

## Testing

| Level | Observable | Interview-ready when… |
| --- | --- | --- |
| Junior | Copies the house test shape correctly (`db: tx` threaded) | you can explain *why* the rollback wrapper works |
| Mid | Designs the test *plan* per change: which layer, which negatives, what's deliberately untested | you can critique a flaky test's design (Kata 8) |
| Senior | Builds test infrastructure (contract vectors, fault injection) and tests it | you argue coverage tradeoffs in cost terms |

## Security

| Level | Observable | Interview-ready when… |
| --- | --- | --- |
| Junior | Runs the [pre-merge checklist](../05-quality-engineering/05-security-checklist.md) mechanically | you can define IDOR with this repo's defense |
| Mid | Catches missing tenancy scoping / secret leakage in review without the checklist | you can answer Q5/Q6 of [08/03](../08-interview-prep/03-api-and-data-modeling-questions.md) cold |
| Senior | Designs the investigation (S5): hypothesis → fixture → evidence → regression pin | you can discuss the SSRF surface of a monitoring product unprompted |

## Communication

| Level | Observable | Interview-ready when… |
| --- | --- | --- |
| Junior | PR descriptions have all five sections | your PRs get first-pass approvals on clarity |
| Mid | Reviews rank severity and anchor every finding; questions carry done homework | your Kata-6 written review reads kind *and* firm |
| Senior | RFCs state constraints before designs; disagreement resolved via cheap experiments | you can produce Story 4's both-sides summary in one minute each |

## Monthly self-assessment protocol

1. For each rubric, write the artifact that proves your level (PR link, recorded mock, written review). No artifact = the level below.
2. Pick the *lowest* row — that's next month's focus (weakest-link model; interviews probe for floors, not ceilings).
3. Re-run one timed round from [08/05](../08-interview-prep/05-debugging-and-code-review-rounds.md) and compare notes to last month's.

Honest calibration note: "mid-level" at most companies = consistently Mid on every rubric here with occasional Senior rows — not Senior everywhere. The interview goal is demonstrating the Mid column *reliably under pressure* plus one or two genuinely senior artifacts (Stories 2, 4, or 6).
