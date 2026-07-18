# Practice pack

The first three fabledocs teach you to *read* this codebase: the curriculum (`../fabledocs/upskill/`) maps it, the onboarding pack (`../fabledocs2/`) narrates its decisions, the ADRs (`../fabledocs3/`) formalize them. This folder is where you stop reading. Everything here is active: you diagnose under time pressure, you build, you retrieve from memory, you follow a schedule.

| File | What it is | Time |
| --- | --- | --- |
| [01 Fire drills](01-fire-drills.md) | Six on-call incident simulations grounded in this repo's real seams. Symptom first, escalating hints, resolution criteria. | 45-90 min each |
| [02 Capstone: at-least-once alerting](02-capstone-at-least-once-alerting.md) | The repo's sharpest known flaw (ADR-009), fixed by you, in five staged milestones from design doc to fault-injection test. | 2-4 days |
| [03 Labs: SQL & Drizzle](03-labs-sql-and-drizzle.md) | Eight exercises against the real schema: tenancy queries, the partial index, the NOT NULL migration, audit forensics. | 30-60 min each |
| [04 Labs: pipeline & Go](04-labs-pipeline-and-go.md) | The workflows-testability refactor and Go checker exercises — the two least-practiced muscles in the earlier docs. | 1-3 hrs each |
| [05 Flashcards](05-flashcards.md) | ~60 retrieval cards: invariants, anchors, tradeoffs, numbers. For spaced repetition, not cramming. | 10 min/day |
| [06 Four-week plan](06-four-week-plan.md) | A daily schedule threading all four fabledocs into one arc: read → drill → build → interview-ready. | 4 weeks |

## Rules of engagement

1. **Attempt before hints.** Every drill and lab has staged hints; each one you open costs you a grade level. The point is calibrating what you can do cold.
2. **Write your answers down.** Spoken-in-your-head answers feel complete and aren't. Diagnosis notes, query text, design docs — artifacts or it didn't happen.
3. **Verify against code, not against these docs.** Some exercises deliberately contain an unresolved question (e.g. whether the cron query filters `deletedAt`). The docs won't tell you; the code will. That's the skill.
4. **This is a training clone.** Branch freely, break things, revert. Nothing here is production.
