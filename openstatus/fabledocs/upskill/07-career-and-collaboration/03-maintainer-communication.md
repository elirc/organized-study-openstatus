# Maintainer Communication

Open-source (and every job) runs on written asks. The rule for all four templates: **do the thinking, then ask for the decision** — never outsource the investigation.

## Asking a question

```md
**Context:** Migrating GET /v1/monitor/:id to the services layer (per the comment
at apps/server/src/routes/v1/index.ts:143-147).
**What I found:** The service verb returns the zod-parsed monitor (regions as array);
the v1 response returns regions as array too, but `assertions` differs: raw text vs parsed.
**My read:** Preserving the wire contract means serializing assertions back at the route.
**Question:** Is byte-identical v1 output a hard requirement, or is parsed-assertions
an acceptable (documented) change?
```

Shows: you read the code, formed a hypothesis, and reduced their work to one decision. Contrast with "how should I migrate this route?" — which asks them to do your job.

## Proposing a feature

Problem first, solution second, scope third: what hurts (with evidence), the smallest version that helps, what you're explicitly *not* doing. Link the RFC template ([02-writing-prs-and-rfcs.md](02-writing-prs-and-rfcs.md)) for anything invariant-touching. Ask "is this direction welcome?" **before** writing three weeks of code — the highest-ROI sentence in open source.

## Reporting a bug

```md
**Expected:** One alert notification per monitor transition.
**Actual:** Two PagerDuty pages, same monitor, 40s apart (screenshots attached).
**Repro:** Not deterministic; occurs when ≥2 regions report error within the same tick.
**Evidence:** two notification_trigger rows with distinct cronTimestamps;
wide-event pairs attached (request_ids X, Y overlapping in time).
**Suspicion (unverified):** read-modify-write on monitor.status across awaits in
apps/workflows/src/checker/index.ts:216-348 — both requests may pass the
`monitor.status === "error"` guard before either writes.
**Happy to:** write the concurrent-request test if the theory sounds plausible.
```

Minimal, evidence-led, suspicion labeled as suspicion, and it ends with an offer, not a demand.

## Responding to review / disagreeing respectfully

- Concede fast when they're right; fix > defend.
- When you disagree, disagree with *evidence class*: "I kept the per-request client because of the comment at checker.go:76 — connection reuse skews latency measurement. Happy to change if that constraint no longer holds."
- Convert stalemates into cheap experiments: "Want me to benchmark both and post numbers?"
- Never litigate taste; litigate invariants and data.

## Triage etiquette (when you become a regular)

Reproduce before labeling; ask for the minimal missing info in one message (not four rounds); close-with-kindness ("this is by design because X — see Y") beats silence.

## Drill

Write the bug report for [debugging scenario 4](../05-quality-engineering/03-systematic-debugging.md) (fail-open proxy on schema-invalid page rows) as if filing it upstream. Grade: does it separate expected/actual/evidence/suspicion, and does it offer a next step you can own?
