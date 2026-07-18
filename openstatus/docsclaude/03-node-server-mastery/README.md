# 03 — Node and Server Mastery

This module is about everything between "a request comes in" and "a row exists in the database." Most senior-level mistakes happen in this zone — wrong runtime, missed transaction, leaky auth boundary, accidental N+1.

## Files

| # | File | What it covers |
| --- | --- | --- |
| 01 | [`01-runtimes-explained.md`](01-runtimes-explained.md) | Node vs Edge vs Browser — what works where and why. |
| 02 | [`02-http-handlers.md`](02-http-handlers.md) | The request/response model in Next.js route handlers; middleware; context creation. |
| 03 | [`03-databases-with-drizzle.md`](03-databases-with-drizzle.md) | Drizzle queries, joins, transactions, the N+1 trap, libSQL/Turso specifics. |
| 04 | [`04-trpc-internals.md`](04-trpc-internals.md) | What tRPC actually does end-to-end — initialization, context, procedures, error formatter. |
| 05 | [`05-auth-and-multi-tenancy.md`](05-auth-and-multi-tenancy.md) | NextAuth model, sessions, cookies, workspace resolution, IDOR prevention. |
| ⨯ | [`exercises.md`](exercises.md) | Drills for each topic, grounded in the repo. |

## How to work this module

Have the codebase open. Almost every file maps to live code you can read and modify. Start the dashboard locally and test your understanding against the network panel: every request you trace here should map to a function you can name.
