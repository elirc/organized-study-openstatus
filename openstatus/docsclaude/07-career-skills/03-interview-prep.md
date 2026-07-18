# 03 — Interview Prep (Senior, Grounded in This Codebase)

Interviews at senior level are pattern recognition and judgement, not algorithms. The best prep is being able to *talk* about real code you've worked on. OpenStatus is open source — you can talk about it in interviews even if you didn't write it, framed as "I studied this codebase to learn the patterns."

This file is a bank of questions you should be able to answer. If you can answer all of them comfortably, you'll do well in most senior interviews.

## Language and runtime

1. Explain the JavaScript event loop. Why does `Promise.resolve().then(...)` fire before `setTimeout(0)`?
2. What does `await` actually do? Write the desugared `.then()` form.
3. Why is `Object.is(NaN, NaN)` true but `NaN === NaN` false?
4. What's a closure? Show one with a real bug it can cause in React.
5. ESM vs CJS — three differences. Which one is OpenStatus, and why does it matter for Edge runtime?
6. What's structural typing? Show a case where it causes a real bug and how branded types prevent it.
7. Why does `useEffect(() => { setX(x+1) }, [])` go infinite if you remove the deps? Why is the warning a help, not a nag?
8. Walk through what happens when you call `await fetch(url)` from a server component vs a client component.

## React

9. Render vs commit — what's the difference?
10. Why do hooks have rules? What goes wrong if you call them conditionally?
11. When is `useMemo` worth using? When is it not?
12. Explain stale closures with an example, then three ways to fix them.
13. What does `key` do in a list? When does the wrong key cause a real bug?
14. Server components vs client components — what can each do that the other can't?
15. What does `"use client"` actually do at build time and at runtime?
16. Hydration — what is it, and how can it fail?
17. React Query: what is `staleTime` for? What does `invalidateQueries` do?
18. tRPC — how do client types stay in sync with the server without code generation?

## State management

19. Local, lifted, URL, server, derived — for each, when do you put state there?
20. Why don't we store `hasUnresolvedIssue` as a column? When *would* we?
21. When should state live in `nuqs` (URL) vs React Query (cache) vs local?
22. Optimistic updates: when worth it, when not? What's the rollback contract?

## Backend

23. Walk through the layers: UI → router → service → DB. What's each layer's job?
24. Why do services take a `ctx` instead of importing `db` directly?
25. What does `withTransaction` do, and why does it reuse an outer tx?
26. Why is the audit emit *inside* the transaction?
27. IDOR — what is it, how does OpenStatus prevent it across workspaces? Cite the file.
28. Why does `requireScope` go first in a service verb, before parse and transaction open?
29. Why does `LIMIT_EXCEEDED` map to `TOO_MANY_REQUESTS` (429), not `FORBIDDEN` (403)?
30. Drizzle is not an ORM — what is it? When would you prefer raw SQL?
31. N+1 — show me one in fake code, then show me how OpenStatus avoids one in real code (`enrichReportsBatch`).
32. Edge vs Lambda — what runs where in this codebase? Why?
33. Why does the public statusPage.get TODO call out performance? What would you do about it?

## Architecture and design

34. Pretend you're adding a new entity (e.g., "incident postmortem"). Walk through the layers you'd touch.
35. Side effects (notification email) — in-band vs outbox? Defend.
36. Member roles for dashboard users — sketch the design.
37. Why are wire schemas and service schemas different? Why is that a feature, not a bug?
38. The Drizzle row type *is* the domain type in this codebase. When would you separate them?
39. The dashboard's `endingLink` hard-codes the lambda router list. What's the cost of this? How would you improve it?
40. Why does this codebase prefer `as const` literal unions over TypeScript `enum`?

## Quality

41. What's the test pyramid for an app like this? What do you skip and why?
42. Why does `withTestTransaction` rely on production code using `withTransaction`?
43. Mocking the DB — why not? What do you mock instead?
44. Run me through a debugging session: "users report stale data after creating a report."
45. Performance audit a slow page — what tools, what order?
46. Security checklist — name 10 things you check before merging.
47. SSRF — explain it, then explain how this codebase prevents it.

## Code review and communication

48. What makes a good PR description?
49. What makes a good RFC?
50. How do you give review that teaches, not just catches?
51. Disagreement in code review — when do you stand firm, when do you defer?
52. A junior says "let's rewrite this in Convex/Remix." What's your response?

## "Tell me about a time" questions, framed for this codebase

You can talk about studying OpenStatus as if it were a project you observed (even if you didn't ship). Frame as "I read this code to understand the patterns, and what struck me was…":

53. "Tell me about a time you found a subtle bug in unfamiliar code." → IDOR test, the cross-workspace rejection scenario.
54. "Tell me about a time you improved a performance issue." → The N+1 in status report enrichment, batched query pattern.
55. "Tell me about a time you designed for a security concern." → The audit log fail-closed pattern, `requireScope` ordering.
56. "Tell me about a tradeoff you had to make." → Edge vs Lambda splitting; in-band vs outbox notifications.

## System design rounds

Senior interviews often ask "design X." Common shapes:

- **Design a status page service** (this is OpenStatus). You have an unfair advantage — study the design and rebuild it.
- **Design a notification system** with email/SMS/Slack/PagerDuty, multi-tenant.
- **Design a uptime checker** that runs N checks/second across regions.
- **Design a typed RPC system** (this is tRPC). Walk through how the types travel.

For each, the structure of a strong answer:

1. **Clarify requirements.** Scale (1K? 1M?), latency targets, multi-tenant, regions, compliance.
2. **Data model.** Tables, relations, indexes.
3. **Request path.** What runs where? What's the hot path?
4. **Failure modes.** What breaks under load? Under partial outage?
5. **Tradeoffs.** What did you optimize, what did you sacrifice?
6. **Iteration.** What would you do next, given more time?

## What interviewers are actually checking

Beyond the answers: how you *think*.

- Do you ask clarifying questions before diving in?
- Do you acknowledge tradeoffs explicitly?
- Do you say "I don't know" without flinching, then reason from first principles?
- Do you push back gracefully when the interviewer disagrees?
- Do you finish what you started, or leave things hanging?

Practice all five. Algorithm questions test pattern matching; senior interviews test taste and behavior.

## Drills

- Pick 10 questions from this list. Time yourself: answer each in 3 minutes. Record. Listen back. What did you ramble on? What did you skip?
- For each "tell me about a time" question, write a one-paragraph story. Use the STAR format (Situation, Task, Action, Result).
- Pair with a friend. Do a 45-minute mock system design round. They pick from the list above.
- Take a recent PR (yours or someone else's) and prepare to talk about it for 10 minutes. What did you change? Why? What would you do differently?
