# 05 — Exercises

## Testing

1. Pick one untested service verb. Write the three baseline tests (happy + audit, scope rejection, cross-workspace rejection).
2. Find a test that depends on order. Fix it. Demonstrate the fix by running the suite in random order.
3. Add `expectAuditRow` assertions to a test that exercises a mutation but doesn't currently verify the audit row.
4. Sketch a test that demonstrates `withTransaction`'s tx-reuse behavior. Show that an inner service called inside an outer tx rolls back along with the outer.
5. Convert one test from "happy path" to "happy path + edge case + failure case." Make the test names self-documenting.

## Debugging

6. Use React DevTools' Profiler to identify the most-rendered component on the dashboard. Hypothesize why; verify.
7. Enable Drizzle query logging. Run a single tRPC call. Count queries. Predict how many should have happened; explain any difference.
8. Introduce a bug intentionally: comment out a `WHERE workspaceId = ...`. Predict the failure mode (test? prod symptom?). Verify against the test suite.
9. Use the Network panel to inspect a tRPC batch. Identify each procedure in the batch. Predict the slowest.
10. Pick a recent bug report (real or fabricated). Walk through the diagnostic loop: reproduce → narrow → hypothesize → test → fix. Document each step.

## Performance

11. Run the bundle analyzer on the dashboard. Identify the three largest dependencies. Are any reducible (tree-shake, lazy-import, replace)?
12. Take a list endpoint. Count its DB round-trips for a request returning 100 rows. Compare with what's theoretically optimal (1-3 batched queries).
13. Find an `await` chain that could be `Promise.all`. Refactor; measure with `console.time` before and after.
14. Profile a page render. Identify a re-render that could be eliminated. Eliminate it (move state down, memoize correctly, split component). Verify with Profiler.
15. Run `EXPLAIN QUERY PLAN` on the largest query in [`packages/api/src/router/statusPage.ts`](../../packages/api/src/router/statusPage.ts). Identify any `SCAN` operations. Sketch what indexes would convert them to `SEARCH`.

## Security

16. Apply the pre-merge security checklist to one of your recent PRs (or pick a public PR from the repo). Document how many items pass and which fail.
17. Find every `dangerouslySetInnerHTML` usage. For each, decide if it's justified.
18. Find every URL fetch in service code that takes user input. Verify SSRF protection.
19. Pick one service mutation. Verify it has the three security guarantees: scope check, workspace ownership, audit emit.
20. Write a "negative test" — a test that confirms an attack is blocked. Example: a test that confirms a read-only API key can't write.

## Synthesis

21. Pretend you've been asked to harden the security of `statusReport.create`. List five concrete improvements (each independent). Defend ordering by impact.
22. Pretend you've been asked to make the public status page 2x faster. List your top three changes. Reference the actual code.
23. Pretend you're investigating a production incident: "users from workspace A see data from workspace B." Walk through your first hour: what do you check, in what order?
24. Pretend you're reviewing a PR that adds a new tRPC procedure. Write the review comments you'd post.

## Self-grading

You're senior-shaped on quality engineering when:
- You can write a security-relevant test before writing the code.
- You can identify the bottleneck in any slow page within 10 minutes.
- You can debug a "works in dev, fails in prod" bug without panicking.
- You can run the pre-merge security checklist in your head.
- You can explain to a junior *why* an audit row inside the transaction matters — and tell a story of a time someone got it wrong.
