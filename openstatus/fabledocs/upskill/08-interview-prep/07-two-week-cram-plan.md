# Two-Week Cram Plan

Assumes ~2.5 focused hours/day (more on weekends). Everything references materials already in this curriculum. If you must cut, cut breadth of question cards, never the mocks or the teach-backs — production under time pressure is what's actually tested.

## Week 1 — load the evidence base

**Day 1.** [00-fast-track](../00-fast-track.md) Saturday-morning + afternoon sections: run the repo if feasible (Docker ok), read the first 10 files. Evening: Story 1 worksheet ([08/06](06-behavioral-star-stories.md)) in bullets.
**Day 2.** [Key flows](../01-codebase-cartography/05-key-flows.md) 1–3, tracing each in the code as you read. Teach-back: narrate Flow 1 aloud in 3 minutes.
**Day 3.** Flows 4–7. Cards [01/Q1–Q5](01-js-ts-node-deep-dive.md) aloud with anchors open.
**Day 4.** [Pattern catalog](../03-architecture-and-patterns/05-pattern-catalog.md) 1–8. Cards 01/Q6–Q10. Teach-back: explain the audit invariant + its failure semantics in 90 seconds.
**Day 5.** Patterns 9–16. Cards [03/Q1–Q6](03-api-and-data-modeling-questions.md). Write Story 2 in bullets.
**Day 6 (weekend, 4h).** Full system-design run: [08/04](04-system-design-from-this-repo.md) steps 1–6 aloud, recorded, repo open (40 min). Review recording against the rubric. Then [annotation drills](../04-code-reading-gym/01-annotation-drills.md) 1–4.
**Day 7 (checkpoint, 2h).** Self-assessment: (a) narrate Flows 1 and 3 from memory — no notes; (b) answer 01/Q2, 01/Q7, 03/Q5 cold; (c) tell Story 1 and 2 in under 2 min each. **Gate:** anything shaky gets Day 8 morning; if all clean, Day 8 as written.

## Week 2 — production under pressure

**Day 8.** [Frontend cards](02-frontend-framework-questions.md) Q1–Q8 aloud. One timed debug: Round A from [08/05](05-debugging-and-code-review-rounds.md) (25 min, honest timer).
**Day 9.** [API cards](03-api-and-data-modeling-questions.md) Q6–Q12. Timed debug Round B. Story 9 harvested/written.
**Day 10.** Review rounds A + C ([08/05](05-debugging-and-code-review-rounds.md)), written reviews, graded against [07/01](../07-career-and-collaboration/01-code-review-mindset.md).
**Day 11.** Second full system-design run — **repo closed** this time, recorded (35 min). Compare to Day 6's run: the delta is your remaining prep. Remaining 01-cards.
**Day 12.** Weakness day: rerun your two worst artifacts (a round, a set of cards, a story). Practical-coding warmup: implement `mapWithConcurrency` and a debounce, typed, from scratch (classics from 01/Q11).
**Day 13 (weekend, 4h).** Full mock loop: 40-min system design (a *variant*: "add real-time" or "10× monitors", [08/04 step 6](04-system-design-from-this-repo.md)) + 25-min debug Round D + behavioral: 4 stories from the [prompt map](06-behavioral-star-stories.md), timed. Recorded, reviewed.
**Day 14 (checkpoint, light).** Final self-assessment against [learning-rubrics.md](../09-reference/learning-rubrics.md) interview-ready column. Prepare *your* questions for interviewers (pull two from the risk register — "how do you handle X?" grounded in real analysis reads as senior). Then stop. Sleep is prep.

## Daily invariants

- Every answer spoken **aloud** — silent rehearsal doesn't transfer.
- Every claim anchored: "in the codebase I studied, …" + concrete mechanism.
- End each day writing one sentence you'd say in an interview that you couldn't have said yesterday.

## If the interview is for a specific stack variant

Skew days 8–9: heavy-frontend role → double the 02-cards, add Q9–Q12; heavy-backend → double 03-cards + re-run [03/04 side-effects](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md) drill; Go-adjacent → add checker files ([checker.go](../../../apps/checker/handlers/checker.go), [update.go](../../../apps/checker/checker/update.go)) to Day 8 reading.
