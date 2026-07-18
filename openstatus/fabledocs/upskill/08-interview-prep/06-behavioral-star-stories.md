# Behavioral STAR Story Worksheets

Nine worksheets. Sources: tickets/projects from [06-contribution-practice](../06-contribution-practice/README.md) (fill in results after actually doing them) plus two study-based stories available immediately. Rehearsal check applies to every story: **under 2 minutes? concrete? ends with impact?**

## Story 1: Learning a complex codebase fast *(available now)*
Prompts this answers: "Tell me about ramping up on unfamiliar code" / "How do you learn?"
Source: this curriculum's Phase-1 method.
Situation: large production monorepo (10 apps, 25 packages, three runtimes, Go + TS).
Task: become productive without a guide.
Action: manifests → entry points → schema → one flow end-to-end; wrote trace tables for the alerting pipeline; verified every claim against file:line before trusting it; kept a verification log separating confirmed findings from hypotheses.
Result: could navigate to any mutation's invariants (scope, tenancy, audit) in minutes; produced a prioritized risk register with anchored evidence.
Evidence to cite: [key flows](../01-codebase-cartography/05-key-flows.md), [risk register](../09-reference/risk-register.md).
Senior-signal details: hypothesis-labeling discipline ("confirmed vs investigate"); reading tests as documentation.
Resume bullet: *Reverse-engineered a multi-runtime monitoring platform (Next/tRPC/Hono/Go) into an anchored architecture and risk analysis.*

## Story 2: Finding a race condition by reading *(available now)*
Prompts: "A hard bug" / "Most interesting technical problem."
Situation: alerting pipeline flips monitor state on regional quorum.
Task: assess notification-delivery correctness.
Action: traced `/updateStatus` and found a read-check-write on `monitor.status` spanning awaits ([checker/index.ts:128-298](../../../apps/workflows/src/checker/index.ts#L128-L298)); identified the downstream unique-insert dedupe as the only backstop and its catch-all weakness ([alerting.ts:131-139](../../../apps/workflows/src/checker/alerting.ts#L131-L139)); designed the CAS fix + concurrent test (M10).
Result: a concrete, evidence-backed proposal distinguishing duplicate-cause from duplicate-effect.
Senior-signal: races found *by reading*, mitigations ranked (CAS vs lock vs tx), honest about unverified frequency.
Resume bullet: *Identified and designed the fix for a concurrency race in a production alerting pipeline.*

## Story 3: Legacy migration without breaking a public API *(do M1 first)*
Prompts: "Working with legacy code" / "Risky change safely shipped."
Skeleton: v1 route on inline SQL, migration direction documented in-code; you snapshot current behavior byte-level, migrate to the services verb, keep defense-in-depth middleware. Result slot: tests proving contract equality; audit rows now emitted. Senior-signal: behavior *addition* (audit) called out in changelog, not smuggled.

## Story 4: Extending an invariant with different failure semantics *(do M3 first)*
Prompts: "A design decision you're proud of" / "Disagreement resolved by reasoning."
Skeleton: user-path audit is fail-closed; you argue pipeline audit should be fail-open (availability of alerting > audit completeness) — same invariant, context-dependent enforcement. This story *is* a tradeoff discussion; rehearse the one-sentence version of each side.

## Story 5: Measured performance work *(do M2 first)*
Prompts: "Optimized something" — the trap question where juniors admit to guessing.
Skeleton: two mount queries flagged by an in-code "HMM" comment; you measured (payloads, waterfall), unified the pipe, verified identical rendering. Senior-signal: the measurement *preceded* the design; you can quote before/after numbers.

## Story 6: Reliability redesign *(do S1 or S3 first)*
Prompts: "Most complex thing you've built."
Skeleton: at-most-once alerting → outbox with staged rollout, or edge-triggered → reconciler. Emphasize the rollout (shadow → flag → enforce) and the kill switch — process maturity is the senior signal here.

## Story 7: Security investigation *(do S5 first)*
Prompts: "Security issue you found/handled."
Skeleton: hypothesis (data layer may bypass proxy gating), fixture-based experiment, evidence-first conclusion either way, regression test. Senior-signal: disclosure etiquette; "prove, don't assume" both directions.

## Story 8: Cross-language contract *(do M6 first)*
Prompts: "Worked across team/stack boundaries."
Skeleton: assertion semantics defined in TS, evaluated in Go, one variant silently unimplemented; you built shared test vectors + shadow evaluation. Senior-signal: the *contract artifact* (shared vectors) outliving your PR.

## Story 9: A mistake and what changed *(harvest honestly)*
Prompts: "Tell me about a failure."
Harvest it from doing the work above — the first time you forget `db: tx` in a test and pollute the shared DB (scenario 5's bug class) is a perfect, honest, low-stakes failure story: what happened, how you caught it, the checklist line you added. Interviewers want the *system change*, not the confession.

---

## Prompt → story map

| Prompt | Primary | Backup |
| --- | --- | --- |
| Ramp-up / learning | 1 | 8 |
| Hard bug | 2 | 9 |
| Risky change | 3 | 6 |
| Tradeoff / disagreement | 4 | 2 |
| Optimization | 5 | — |
| Complex build | 6 | 8 |
| Security | 7 | 2 |
| Failure | 9 | — |
| Ambiguity | 1 | 7 |
| Leadership/initiative | 6 | 3 |

Rehearsal protocol: write each as bullets (not prose), record the 2-minute telling, listen for: concrete nouns (file, number, name), a decision *you* made, and an ending with impact. Redo any story that ends with "…and it worked."
