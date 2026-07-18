# 04 — Exercises

These are design exercises, not coding exercises. Write your answers in prose; argue both sides where there's tension.

## Layering

1. Sketch the layer diagram for OpenStatus from memory. Compare with the actual structure. What did you misremember?
2. Find a router procedure that has more than ~30 lines of resolver code. Decide if some of it should move down to a service. Justify.
3. Pick a service file. List every layer it interacts with (DB, audit, errors, scope). Note any layer it skips.
4. Imagine adding a CLI tool that creates a status report from a script. Walk through what it would need to construct: actor, workspace, ctx, input. Where does each piece come from?
5. A junior on your team adds a tRPC procedure that runs a Drizzle query directly without going through a service. Write the code review comment.

## Validation

6. Pick one field (e.g., status report title) and document its validation at every layer: form, router, service, DB. Confirm they're consistent.
7. Find a `z.coerce.date()`. Trace the date value across the request boundary: what shape does it have on the wire, what shape after parsing?
8. Add a new field "severity" (low/medium/high/critical) to status reports. List every file you'd touch. Order them in the right migration sequence.
9. Could you reduce Zod schemas by deriving them from the Drizzle table definition? Pros and cons.
10. Why doesn't the service import the *router's* Zod schema? (Hint: think about transport-independence.)

## Transactions and audit

11. Find one mutation. List every row it writes, in order. Confirm they're all in one transaction.
12. Find a service that calls another service. Trace the transaction reuse: who opens it, who joins, what's the inner ctx.db?
13. Sketch an outbox table for notifications. What columns? What worker drains it? What retry behavior?
14. Find a delete service that doesn't audit (if any). Decide if that's correct or a gap.
15. Pretend you have a "merge two workspaces" feature. List every consideration: atomicity, audit, side effects, rollback, audit-of-the-merge itself.

## Derived data and caching

16. Pick an `invalidateQueries` call. Argue that it's too broad. Argue that it's too narrow. Decide.
17. Sketch a Redis-backed cache for `statusPage.get`. What's the key? When do you invalidate? What's the TTL?
18. Find a place where data is computed in a React render (e.g., `hasUnresolvedIssue`). Argue for memoizing it. Then argue against. Decide.
19. The public status page query loads a lot. Suggest a split that lets embed-only pages skip the heavy parts. Reference the actual code.
20. What would you cache at the CDN level for a public status page? What would you intentionally *not* cache?

## Synthesis

21. Pick one OpenStatus feature (status reports, monitors, notifications). Re-design it from scratch given the requirements. Compare against the actual code. What's different? Why is the real version the way it is?
22. Pick one *missing* feature you'd want to build (e.g., "incident postmortems"). Design the layers: DB schema, service verbs, router shape, UI flow, tests. Don't write code yet.
23. Imagine OpenStatus needs to support 10x more public status page traffic. What changes? (Hint: caching, read replicas, query splits, CDN, edge function strategy.)
24. The team is considering moving notifications from in-band to an outbox. Write a 1-page RFC: motivation, design, migration plan, risks.

## Self-grading

You're senior-shaped on architecture when:
- You can argue both sides of a layering question without picking until you have facts.
- You can spot a missing transaction, a missing audit row, or a missing scope check in 10 seconds.
- You can name three caching strategies and pick the right one per scenario.
- You can sketch the layers of a new feature before writing any code.
- You can read a PR and notice what's *missing*, not just what's wrong.
