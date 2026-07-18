# 04 — Architecture and Patterns

This module is about *taste*. Once you can read code, the next jump is reading it and seeing the architecture — the layering, the boundaries, the side effects, the tradeoffs. That's what a senior brings to a code review that a mid doesn't.

We'll use OpenStatus as the lab, but every pattern here generalizes. The goal isn't to memorize this repo's structure; it's to internalize the *reasoning* so you can apply it on a different codebase next month.

## Files

| # | File | What it covers |
| --- | --- | --- |
| 01 | [`01-layering-and-services.md`](01-layering-and-services.md) | Why a service layer exists, what belongs in it, how to keep routers thin. |
| 02 | [`02-validation-and-boundaries.md`](02-validation-and-boundaries.md) | Zod at every layer, error mapping, why duplication is the point. |
| 03 | [`03-transactions-and-audit.md`](03-transactions-and-audit.md) | Atomicity, the audit log invariant, fail-closed design. |
| 04 | [`04-derived-data-and-caching.md`](04-derived-data-and-caching.md) | Read models, computed views, cache invalidation strategies. |
| 05 | [`05-real-patterns-in-this-repo.md`](05-real-patterns-in-this-repo.md) | A catalog of the recurring shapes you'll see — and how to use them yourself. |
| ⨯ | [`exercises.md`](exercises.md) | Design exercises with answer rubrics. |

## How to work this module

Architectural taste develops slowly. Don't rush through. Read each file, then re-read code you thought you understood and notice what you now see differently. Sketch UML-ish diagrams in a notebook. Argue out loud (rubber duck or human) about what you'd change.

The single most useful exercise: take a feature in this repo and try to re-design it from scratch given the requirements. Then diff your design against the actual code. The gaps are the lessons.
