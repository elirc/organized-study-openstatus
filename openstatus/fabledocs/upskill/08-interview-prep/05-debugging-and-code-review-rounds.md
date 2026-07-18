# Debugging & Code Review Rounds (Timed Simulations)

Seven timed simulations converted from earlier modules. Run each with a timer and, ideally, a person (or a rubber duck you answer *aloud* to). Interviewer follow-ups included — have someone spring them, or spring them on yourself at the timestamps given.

## Format expectations (what interviewers grade)

Debugging rounds: structured narrowing (not shotgun guessing), cheap probes first, stating assumptions, and finishing with root cause + regression test. Review rounds: severity-ranked findings, invariants over taste, kind specific language, and *one* good question to the author.

---

## Debug Round A (25 min): the double page
Setup: hand yourself [scenario 1](../05-quality-engineering/03-systematic-debugging.md) *without re-reading it* — "customer paged twice for one outage; here's the alerting code" (open [checker/index.ts](../../../apps/workflows/src/checker/index.ts#L86-L356) and [alerting.ts](../../../apps/workflows/src/checker/alerting.ts#L17-L255) cold).
Interviewer follow-ups: at 10 min — "you can run one SQL query, what is it?"; at 18 min — "the trigger rows have *different* cronTimestamps, now what?"; at 23 min — "what test prevents regression?"
Grading rubric: Basic = finds the dedupe insert. Solid = distinguishes same-tick vs cross-tick duplication and reads the guard race. Strong = proposes CAS/serialization + concurrent test, and mentions what telemetry would confirm frequency in prod.

## Debug Round B (25 min): stuck in error
Setup: scenario 2 cold — "endpoint healthy, dashboard says error for 2 hours."
Follow-ups: 8 min — "Tinybird shows all-green probes for the last hour"; 15 min — "the monitor's region list was edited yesterday"; 20 min — "design the permanent fix."
Rubric: Strong = articulates edge-triggered vs level-triggered and proposes the reconciler (Project S3) unprompted.

## Debug Round C (20 min): works via API, fails via UI
Setup: scenario 3 cold. This round tests *layer knowledge* more than debugging.
Follow-ups: 7 min — "the browser shows BAD_REQUEST with a zodError payload"; 14 min — "why does the REST path not hit this?"
Rubric: Strong = explains the two-schema design and the legacy/services split from memory, with the migration comment cited.

## Debug Round D (20 min): the flaky CI suite
Setup: scenario 5 cold — "this suite fails ~1 in 5 on main, passes on rerun."
Follow-ups: 8 min — "failures always mention quota"; 15 min — "you can change one helper call — which?"
Rubric: Strong = names the shared-fixture taxonomy and the existing `cleanQuotaGatedTables` remedy, then argues for per-test workspaces as the systemic fix.

---

## Review Round A (20 min): async audit
PR: [Kata 6](../04-code-reading-gym/04-review-katas.md) (fire-and-forget audit).
Deliverable: written review, severity-ranked, ≤ 6 comments.
Follow-ups: "the author replies 'audit inserts add 15ms p99' — respond" (demand the measurement, offer the outbox-in-tx alternative).
Rubric: Strong = names the invariant deletion in the first sentence, both inconsistency directions (lost audit *and* orphan audit), and keeps the tone collegial.

## Review Round B (20 min): quorum removal
PR: [Kata 2](../04-code-reading-gym/04-review-katas.md).
Follow-ups: "author shows a trace where the count query took 800ms — now what?" (fix the query/index, don't delete the semantics; ask for the wide-event distribution, not one trace).
Rubric: Strong = separates the performance claim from the behavior change and proposes how to get the perf *without* the semantic loss.

## Review Round C (15 min): the Go client refactor
PR: [Kata 4](../04-code-reading-gym/04-review-katas.md) (shared HTTP client).
Rubric: Strong = cites the in-code comment as the contract ("new client per request to avoid connection reuse", [checker.go:76](../../../apps/checker/handlers/checker.go#L76)), explains *why* it exists (latency measurement fidelity), and suggests the measured alternative.

---

## Self-assessment after each round

Log: minutes to first correct hypothesis; probes wasted; whether you finished with a test; one sentence you'd say differently. Two clean Strong runs per round type = interview-ready for that round. Cross-check against [learning-rubrics.md](../09-reference/learning-rubrics.md).
