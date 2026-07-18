# 05 — Quality Engineering

Code that works *now* is junior code. Code that keeps working *after* the next change, *under load*, *for users you didn't design for*, and *when someone else maintains it* is senior code. Quality engineering is the practice of thinking about all four.

## Files

| # | File | What it covers |
| --- | --- | --- |
| 01 | [`01-testing-strategy.md`](01-testing-strategy.md) | The test pyramid, picking what to test where, the OpenStatus test stack. |
| 02 | [`02-writing-tests-here.md`](02-writing-tests-here.md) | Concrete recipes — service tests, router tests, the transaction-rollback pattern. |
| 03 | [`03-systematic-debugging.md`](03-systematic-debugging.md) | Debugging by deduction. Browser tools, Node tools, narrowing. |
| 04 | [`04-performance-thinking.md`](04-performance-thinking.md) | Where time actually goes. Profiling React, query plans, bundle analysis. |
| 05 | [`05-security-checklist.md`](05-security-checklist.md) | The 20 things to check before merging a PR. SSRF, IDOR, secrets, etc. |
| ⨯ | [`exercises.md`](exercises.md) | Bug-hunting and "what would you write" exercises. |

## How to work this module

Quality engineering is a skill you build by *practicing it on real code*. Reading is necessary but insufficient. Pick a real PR (yours or someone else's) and apply the security checklist before merging. Profile a real page and compare to your prediction. Write one test that you wouldn't have thought to write before this module.
