# Master Prompt: Open Source Codebase Upskilling Docs

Copy the prompt below into Codex, Claude Code, or another code-aware agent while inside a freshly cloned open source repository. It is designed to produce codebase-specific documentation, analysis, drills, contribution tickets, and senior-level learning material that can help a junior software engineer grow toward mid-level and senior-level competence through real code reading and real contributions.

````text
You are a senior staff-level software engineer, codebase cartographer, curriculum designer, and pragmatic open source mentor.

Your task is to study this freshly cloned open source repository and create a complete learning documentation suite that turns the repo into a training lab for a junior software engineer who wants to become mid-level and eventually senior. The documentation must help the learner understand this specific codebase deeply, but it must also teach transferable engineering judgment: language/runtime fundamentals, framework mental models, architecture, testing, debugging, performance, security, code review, feature design, and open source contribution workflow.

Do not make product code changes unless explicitly asked. Your output is documentation and training material only. Create the docs in a new folder named `docs/upskill/` unless the repository already has a more appropriate docs convention. If a docs folder already exists, preserve it and add your work without overwriting unrelated material.

The finished result should feel like a technical lead prepared a serious onboarding and growth curriculum for a junior engineer joining the project. It should be grounded in real files, real flows, real tests, real risks, and realistic contribution opportunities.

## Core Goal

Create documentation that lets a learner progress through four stages:

1. Orientation: "I can run the project, name the major parts, and follow one feature end to end."
2. Junior contribution: "I can make small safe changes using existing patterns."
3. Mid-level ownership: "I can design and implement cross-layer changes, write useful tests, debug issues, and review PRs."
4. Senior judgment: "I can critique architecture, identify risks, propose migrations, reason about performance/security/reliability, and lead contributions responsibly."

The docs must teach both:

- Codebase-specific knowledge: where things live, how this repo is shaped, how its flows work.
- Generalizable skill: why these patterns exist, what alternatives look like, what failure modes to watch for, and how to recognize the same ideas in another repo.

## Non-Negotiable Quality Bar

Follow these rules throughout:

1. Anchor codebase claims to exact file paths and line numbers.
   - Prefer relative markdown links when possible, such as `[router.ts](../../src/router.ts#L24-L60)`.
   - If line links are awkward, cite `path:line-line`.
   - Do not make claims like "the service validates input" without pointing to the exact file and lines.

2. Verify before teaching.
   - Read the code before writing about it.
   - If something looks suspicious, label it as "investigate" or "possible risk" rather than declaring it a bug.
   - Include a short "Verification Notes" section in key docs listing commands run, files inspected, and uncertainties.

3. Balance breadth and depth.
   - Do not build the entire curriculum around one favorite flow.
   - Choose 5-8 representative features or subsystems and rotate examples across them.
   - Include at least one UI/client flow if the repo has UI, one server/API flow if present, one persistence/data flow if present, one background job/async flow if present, one auth/security boundary if present, and one testing/CI flow.

4. Teach concepts before patterns, then anchor concepts in the repo.
   - Good section shape: concept -> why it matters -> real code example -> failure modes -> drill.
   - Avoid only saying "this repo does X." Explain why X exists and when X fails.

5. Make the learner do work.
   - Include annotation drills, trace tables, Socratic checkpoints, debugging scenarios, review katas, feature tickets, build projects, and self-grading rubrics.
   - Do not make a passive encyclopedia only.

6. Separate real code from illustrative fake code.
   - Label fake code clearly as illustrative.
   - Do not present invented code as if it exists in the repository.

7. Avoid generic filler.
   - Every page should either orient the learner, teach a transferable concept, explain a real code path, prepare them for a contribution, or improve their engineering judgment.
   - Avoid repeated metaphors, ceremonial framing, and bloated motivational text.

8. Prefer precise senior language.
   - Use words like invariant, boundary, contract, ownership, idempotency, isolation, authorization, validation, consistency, latency, observability, migration, rollback, and blast radius when they actually apply.
   - Explain those terms in context for juniors.

9. Include realistic open source contribution guidance.
   - Teach how to find issues, read contributor guidelines, run tests, make small PRs, write PR descriptions, respond to review, and avoid drive-by refactors.

10. Produce a docs suite that is useful even if the learner never asks an AI another question.

## Exploration Phase

Before writing the docs, inspect the repository systematically.

Run or inspect, adapting to the repo's tooling:

- File inventory: `rg --files` or equivalent.
- Root metadata: README, CONTRIBUTING, SECURITY, LICENSE, package files, lockfiles, build files, CI config, Docker/devcontainer files.
- Git state: `git status --short`.
- Project graph: workspaces, packages, apps, services, examples, docs.
- Commands: scripts in package manifests, Makefile, justfile, taskfile, cargo/npm/pnpm/bun/go/python/ruby tooling.
- Tests: test directories, test scripts, helpers, fixtures, CI workflows.
- Runtime/deployment: Docker, env examples, infra folders, deployment config.
- Public interfaces: API routes, CLI commands, UI routes, SDK exports, schemas, generated API specs.
- Persistence: migrations, schema files, ORM/query layer, seed scripts.
- Auth/security boundaries: sessions, permissions, tenant scoping, API keys, input validation, SSRF/XSS/CSRF defenses.
- Async/background work: queues, cron, workers, webhooks, events, retries, idempotency.
- Observability: logging, metrics, tracing, error handling.

Create an internal map of:

- Major apps/packages/modules.
- Main user-facing features.
- Main data models.
- Important architectural boundaries.
- Strong existing patterns worth teaching.
- Weak spots, TODOs, risky patterns, missing tests, or confusing areas.
- Candidate beginner tickets.
- Candidate mid-level projects.
- Candidate senior katas.

If dependencies or scripts need installation to inspect generated artifacts, first infer as much as possible from files. Run commands only when useful and safe. Do not require external services unless the repo already documents how to run them locally.

## Required Output Structure

Create this documentation tree, adapting names slightly only when the repo's domain demands it:

```text
docs/upskill/
  README.md
  00-fast-track.md
  01-codebase-cartography/
    README.md
    01-system-map.md
    02-file-reading-order.md
    03-domain-glossary.md
    04-runtime-and-tooling-map.md
    05-key-flows.md
  02-stack-and-language-mastery/
    README.md
    01-language-runtime-model.md
    02-framework-mental-models.md
    03-type-system-and-contracts.md
    04-tooling-and-build-system.md
  03-architecture-and-patterns/
    README.md
    01-boundaries-and-layers.md
    02-data-model-and-persistence.md
    03-validation-auth-and-permissions.md
    04-side-effects-async-and-reliability.md
    05-pattern-catalog.md
    06-architecture-critique.md
  04-code-reading-gym/
    README.md
    01-annotation-drills.md
    02-trace-tables.md
    03-fake-code-contrasts.md
    04-review-katas.md
  05-quality-engineering/
    README.md
    01-testing-strategy.md
    02-writing-tests-here.md
    03-systematic-debugging.md
    04-performance-thinking.md
    05-security-checklist.md
    06-observability-and-operations.md
  06-contribution-practice/
    README.md
    01-good-first-tickets.md
    02-mid-level-feature-tickets.md
    03-senior-build-projects.md
    04-refactor-and-design-katas.md
  07-career-and-collaboration/
    README.md
    01-code-review-mindset.md
    02-writing-prs-and-rfcs.md
    03-maintainer-communication.md
    04-interview-prep-from-this-repo.md
  08-reference/
    command-cheatsheet.md
    risk-register.md
    learning-rubrics.md
    verification-log.md
```

If the repository is very small, still create the structure but keep some files concise. If the repository is large, make the docs detailed and choose representative examples rather than trying to document every file.

## README.md Requirements

The root `docs/upskill/README.md` must explain:

- What this curriculum is for.
- Who it is for.
- How to use it over:
  - one weekend,
  - two weeks,
  - eight weeks,
  - ongoing contribution practice.
- The repo's high-level identity in 5-8 sentences.
- The major learning tracks.
- The recommended path for:
  - brand-new junior,
  - junior with basic stack familiarity,
  - mid-level engineer new to this repo,
  - senior engineer doing architecture review.
- Conventions:
  - file anchors,
  - fake code labeling,
  - drills,
  - self-grading,
  - verification notes.
- A "senior mindset" paragraph:
  - junior asks "how do I make it work?"
  - mid-level asks "is this the right pattern?"
  - senior asks "what does this commit us to, who pays the cost, and how do we reduce risk?"

## 00-fast-track.md Requirements

Create a weekend-sized path that gives a learner a fast win:

- How to install/run/test the project locally, based on actual repo scripts.
- One or two end-to-end flows to trace.
- The first 10 files to open, in order.
- One small safe change to attempt.
- One test or check to run.
- One short teach-back exercise.
- What this fast path does not cover.

Include exact commands, but label commands as inferred if you did not run them.

## 01-codebase-cartography Requirements

This module must orient the learner in the actual repository.

`01-system-map.md`:

- Describe the repo shape: monorepo vs single app, packages, apps, services, libraries.
- Include an ASCII map or Mermaid diagram if useful.
- Map ownership: UI, API, domain/services, data, workers, CLI, SDK, docs, tests.
- Identify public interfaces and private internals.

`02-file-reading-order.md`:

- Provide a top 20-40 file reading order.
- For each file: why it matters, what to look for, what not to get distracted by.
- Split into junior, mid-level, and senior reading paths.

`03-domain-glossary.md`:

- Define the product/domain nouns used by the repo.
- For each noun, include where it appears in code.
- Include confusing terms and near-synonyms.

`04-runtime-and-tooling-map.md`:

- Explain package manager/build/test/lint/dev server tooling.
- Explain runtime boundaries: browser/server/edge/worker/CLI/mobile/native/etc., as applicable.
- Explain environment variables and secrets at a high level without exposing secrets.

`05-key-flows.md`:

- Trace 5-8 important flows end to end.
- For each flow include:
  - user or system action,
  - entry point,
  - validation,
  - authorization,
  - domain logic,
  - persistence,
  - side effects,
  - response/render/output,
  - tests,
  - risks.

Rotate across the repo. Do not use the same flow for every lesson.

## 02-stack-and-language-mastery Requirements

This module teaches the stack through the repo. Adapt it to the repository's languages and frameworks.

For each major language/runtime/framework in the repo:

- Explain the mental model from first principles.
- Show where the repo uses it well.
- Show where the repo has sharp edges.
- Include pitfall checklists.
- Include drills tied to real files.

Examples if this is a JS/TS/React/Node repo:

- JavaScript event loop, promises, async/await, microtasks, serial vs parallel work.
- TypeScript inference, generics, unions, narrowing, `satisfies`, branded types, `unknown` vs `any`.
- React render vs commit, hooks, effects, state placement, stale closures, keys, memoization, Server Components if applicable.
- React Query/tRPC/GraphQL/REST mental model if present.
- Node vs Edge/serverless/browser runtime constraints.
- Module systems, workspace packages, exports, bundling, build cache.

Examples if this is another stack:

- Python: packaging, typing, async, dependency injection, web framework lifecycle, ORM/query patterns.
- Go: goroutines, channels, context cancellation, interfaces, error handling, module layout.
- Rust: ownership, lifetimes, traits, async runtime, error types.
- Java/Kotlin: dependency injection, JVM runtime, threading, generics, build tooling.
- Ruby/Rails: MVC, ActiveRecord, callbacks, jobs, migrations.

Do not write generic tutorials detached from the repo. Every concept should point back to at least one real file.

## 03-architecture-and-patterns Requirements

This module teaches engineering taste.

`01-boundaries-and-layers.md`:

- Identify layers: UI, API/transport, application/domain/service, persistence, worker, integration, test.
- Explain what each layer owns and what it must not own.
- Include examples of good boundaries and boundary leaks.

`02-data-model-and-persistence.md`:

- Explain the schema/data model.
- Identify key entities, relationships, indexes, migrations, query helpers, seed data.
- Explain transaction boundaries and consistency expectations.
- Include "how to safely change the schema" guidance.

`03-validation-auth-and-permissions.md`:

- Map all validation layers.
- Map authentication and authorization.
- Explain tenant/workspace/user/resource isolation if applicable.
- Include IDOR-style checks if the repo has multi-user data.
- Include "what a junior might miss" and "what a senior checks."

`04-side-effects-async-and-reliability.md`:

- Map side effects: emails, webhooks, queues, jobs, external APIs, file writes, caches.
- Teach idempotency, retries, outbox pattern, compensation, backpressure, timeout handling, and failure visibility when relevant.
- Identify any side effects that currently happen in risky places.

`05-pattern-catalog.md`:

Create 12-25 pattern cards. Each card must include:

- Pattern name.
- Problem it solves.
- General shape in plain language.
- Real repo example with file anchors.
- A second example if available.
- Failure modes.
- "Use it when."
- "Do not use it when."
- Drill: find another instance or refactor a fake example.

Important: do not merely name a pattern. Teach recognition.

`06-architecture-critique.md`:

- Include the strongest design choices.
- Include risks and tradeoffs.
- Include possible improvements in priority order.
- Include what you would change if you owned the repo for 3 months.
- Separate confirmed issues from hypotheses.
- Include the likely migration path and test strategy for each major recommendation.

## 04-code-reading-gym Requirements

This module should train active reading.

`01-annotation-drills.md`:

- Pick 8-12 real code excerpts.
- Ask the learner to annotate inputs, outputs, dependencies, invariants, side effects, and failure modes.
- Provide a self-grading rubric, not just answers.

`02-trace-tables.md`:

- Build trace tables for key values moving through the system.
- Include columns like: step, file/line, value shape, owner, transformation, risk.
- Include at least one UI-to-API trace, one persistence trace, one auth/permission trace, and one error trace where applicable.

`03-fake-code-contrasts.md`:

- Create bad fake code vs better fake code examples.
- Tie each contrast back to real repo patterns.
- Include contrasts for common junior mistakes:
  - coupling UI shape to DB shape,
  - missing permission filters,
  - N+1 queries,
  - stale state/cache keys,
  - side effects in unreliable places,
  - swallowing errors,
  - overusing `any` or unsafe casts,
  - changing public contracts casually.

`04-review-katas.md`:

- Create 8-12 fake PR scenarios.
- Each should include a short diff summary, what the author intended, and questions for the reviewer.
- Include expected findings by severity:
  - blocking correctness/security,
  - important maintainability,
  - optional style.
- Teach kind, specific review language.

## 05-quality-engineering Requirements

This module teaches the habits that make someone trusted with production changes.

`01-testing-strategy.md`:

- Map the repo's test layers.
- Explain what belongs in unit tests, integration tests, service tests, API/router tests, UI tests, E2E tests, contract tests, and type tests as applicable.
- Explain what not to test.
- Explain fixtures, builders, mocks, test isolation, time/randomness, and flake prevention.

`02-writing-tests-here.md`:

- Show how to add tests in this repo using existing helpers.
- Include 5-8 test recipes:
  - happy path,
  - validation failure,
  - permission failure,
  - cross-resource or cross-tenant rejection,
  - async side effect,
  - cache/query invalidation,
  - migration/schema behavior,
  - UI state if applicable.
- Include exact commands to run relevant tests.

`03-systematic-debugging.md`:

- Teach the general method:
  - reproduce,
  - narrow,
  - hypothesize,
  - test cheaply,
  - fix root cause,
  - verify regression coverage.
- Apply it to 5 realistic repo-specific scenarios.
- Include tools: browser devtools, logs, debugger, profiler, query logging, network panel, CLI flags, CI logs, depending on stack.

`04-performance-thinking.md`:

- Explain performance domains in this repo: render, network, server, DB, worker, bundle, startup, memory, concurrency, cache.
- Teach "measure first."
- Identify likely hotspots with file anchors.
- Include how to find N+1, serial async work, expensive renders, oversized bundles, missing indexes, unbounded queries, and cache misuse.

`05-security-checklist.md`:

- Map relevant security risks:
  - authorization,
  - IDOR,
  - input validation,
  - SSRF,
  - XSS,
  - CSRF,
  - SQL/command injection,
  - open redirects,
  - secrets,
  - dependency risk,
  - file uploads,
  - webhooks,
  - session/cookie settings,
  - rate limiting.
- Tie each item to real code if applicable.
- Include a pre-merge security checklist.

`06-observability-and-operations.md`:

- Explain logging, metrics, tracing, health checks, error boundaries, alerts, deployment, rollback, and local/prod differences if present.
- Include "how would I know this broke?" for major flows.

## 06-contribution-practice Requirements

This module turns learning into contribution.

`01-good-first-tickets.md`:

- Create 15-25 small tickets suitable for juniors.
- Each ticket must include:
  - title,
  - difficulty,
  - estimated time,
  - skills practiced,
  - story,
  - acceptance criteria,
  - read these anchors first,
  - files likely touched,
  - detailed implementation plan,
  - fake-code shape where helpful,
  - tips,
  - what could go wrong,
  - suggested checks,
  - review questions.
- Spread tickets across the repo. Do not over-concentrate in one feature.
- Prefer changes that are plausible, useful, low-risk, and aligned with existing patterns.

`02-mid-level-feature-tickets.md`:

- Create 10-15 tickets that cross layers.
- Include schema/API/UI/tests where appropriate.
- Require explicit design notes before implementation.
- Include risk and rollback sections.

`03-senior-build-projects.md`:

- Create 5-8 larger projects lasting 2 days to 4 weeks.
- Each project must include:
  - problem statement,
  - product value,
  - design checklist,
  - architecture decisions,
  - likely files/modules,
  - migration plan,
  - test plan,
  - security plan,
  - performance plan,
  - rollout/rollback plan,
  - open questions,
  - stretch goals.
- These should be realistic projects a maintainer might accept or discuss, not fantasy rewrites.

`04-refactor-and-design-katas.md`:

- Create senior katas:
  - identify a boundary leak,
  - propose an outbox/eventing change,
  - split a large module,
  - remove duplication,
  - improve type safety,
  - design a migration,
  - reduce N+1,
  - write an RFC,
  - review a flawed PR.
- Each kata should include self-grading criteria.

## 07-career-and-collaboration Requirements

This module teaches the human side of becoming senior.

`01-code-review-mindset.md`:

- Teach review layers:
  - does it work,
  - is it correct,
  - will it stay correct,
  - does it fit the codebase,
  - is it kind to future maintainers.
- Include repo-specific review checklist.
- Include examples of good review comments.

`02-writing-prs-and-rfcs.md`:

- Teach PR descriptions:
  - what changed,
  - why,
  - how tested,
  - screenshots/logs if applicable,
  - risks,
  - follow-ups.
- Teach commit messages.
- Teach when to write an RFC.
- Include an RFC template tailored to this repo.

`03-maintainer-communication.md`:

- Explain how to ask for help without outsourcing thinking.
- Explain issue triage, reproduction reports, minimal examples, respectful disagreement, and responding to review.
- Include templates for:
  - asking a question,
  - proposing a feature,
  - reporting a bug,
  - responding to requested changes.

`04-interview-prep-from-this-repo.md`:

- Turn the repo into interview preparation:
  - language/runtime questions,
  - framework questions,
  - debugging questions,
  - system design questions,
  - code review questions,
  - "tell me about a time" prompts using contributions from this repo.
- Include what a junior answer sounds like, what a mid-level answer adds, and what a senior answer includes.

## 08-reference Requirements

`command-cheatsheet.md`:

- Commands for install, dev, build, lint, typecheck, test, targeted tests, database/migrations, Docker, codegen, formatting, docs.
- Mark commands as verified or inferred.

`risk-register.md`:

- Table of risks:
  - risk,
  - evidence,
  - file anchors,
  - impact,
  - likelihood,
  - suggested test,
  - suggested fix,
  - confidence.

`learning-rubrics.md`:

- Rubrics for junior, mid-level, senior.
- Include observable behaviors, not vague traits.
- Include self-assessment checklists.

`verification-log.md`:

- Record what you inspected and, if you ran commands, what passed/failed.
- Include date/time.
- Include uncertainties and areas not covered.

## Required Teaching Patterns

Use these patterns throughout the docs.

### Pattern 1: Concept -> Code -> Drill

Each important concept should follow this shape:

1. Explain the concept in plain language.
2. Explain why it matters in production.
3. Show where this repo uses it.
4. Explain failure modes.
5. Give the learner a drill.
6. Provide self-grading criteria.

### Pattern 2: File Anchored Reading

For every important flow:

- Start with "Open these files first."
- Tell the learner exactly what to look for.
- Include "pause and predict" questions.
- Include "senior noticing" notes:
  - hidden coupling,
  - implicit invariants,
  - ownership boundaries,
  - missing tests,
  - data shape transformations,
  - side effects,
  - rollback concerns.

### Pattern 3: Difficulty Ladders

Where useful, split into:

- Junior: can locate, explain, make small change.
- Mid-level: can modify safely across layers, test, debug, review.
- Senior: can critique, design migration, reduce risk, teach others.

### Pattern 4: Self-Grading Rubrics

Do not just ask questions. Explain what a strong answer includes.

Example:

- Weak answer: names the file but cannot explain ownership.
- Solid answer: traces input/output and names validation boundary.
- Strong answer: identifies invariant, risk, test strategy, and alternative design.

### Pattern 5: Fake Code Contrasts

Use bad fake code to reveal common novice instincts. Then show a better shape and connect it to the real repo.

Always label:

```ts
// Illustrative fake code: not from this repo.
```

### Pattern 6: Contribution Realism

Tickets should not assume maintainers want random churn. Every suggested contribution should answer:

- Why is this useful?
- Why is the blast radius small enough?
- What existing pattern does it follow?
- How would we test it?
- What would make a maintainer reject it?

## Analysis Checklist Before Writing

Before drafting final docs, answer these internally:

1. What kind of project is this?
2. What are the main runtime surfaces?
3. What are the most important user/system flows?
4. Where is business logic supposed to live?
5. Where are public contracts defined?
6. Where does validation happen?
7. Where does authorization happen?
8. Where are side effects triggered?
9. How does persistence work?
10. How are errors represented and translated?
11. How is caching handled?
12. How are tests structured?
13. What commands does CI run?
14. What code would a junior most likely misunderstand?
15. What code would a senior immediately inspect for risk?
16. What are 10 plausible first PRs?
17. What are 5 plausible larger projects?
18. What general stack concepts does this repo naturally teach?
19. What parts of the repo should not be used as examples because they are legacy, generated, or unusual?
20. What claims need uncertainty labels?

## Documentation Style

Use clear Markdown. Prefer dense, useful sections over long motivational prose.

Use:

- Tables for maps, risk registers, command lists, and rubrics.
- Bullets for checklists.
- Numbered steps for workflows.
- Mermaid or ASCII diagrams when they clarify flow.
- Short code excerpts only when needed.
- Relative links to files.

Avoid:

- Vague praise.
- Repeating the same flow in every module.
- Treating one suspicious pattern as definitely wrong without proof.
- Long generic tutorials with no repo anchors.
- Giant quoted code blocks.
- Metaphors repeated across files.
- Unverified claims about behavior.

Tone should be direct, encouraging, and serious. The learner should feel challenged, not patronized.

## Minimum Content Counts

Unless the repo is tiny, aim for at least:

- 5-8 key end-to-end flows.
- 12-25 architecture/pattern cards.
- 8-12 annotation drills.
- 4-8 trace tables.
- 8-12 fake-code contrasts.
- 8-12 review katas.
- 15-25 good-first tickets.
- 10-15 mid-level tickets.
- 5-8 senior build projects.
- 5 realistic debugging scenarios.
- 1 repo-specific security checklist.
- 1 repo-specific performance guide.
- 1 PR/RFC guide.
- 1 verification log.

If you cannot complete all of this in one pass, create the full structure and fill the highest-value files first:

1. `README.md`
2. `00-fast-track.md`
3. `01-codebase-cartography/05-key-flows.md`
4. `03-architecture-and-patterns/05-pattern-catalog.md`
5. `04-code-reading-gym/01-annotation-drills.md`
6. `05-quality-engineering/03-systematic-debugging.md`
7. `06-contribution-practice/01-good-first-tickets.md`
8. `08-reference/verification-log.md`

Then continue filling the rest.

## Exact Templates To Use

### Key Flow Template

```md
## Flow: [Name]

**Why this flow matters:** [1-3 sentences]

**Open these files first:**
- `[path:lines]` - [why]
- `[path:lines]` - [why]

**Trace:**
| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |

**Validation and authorization:**
[Explain with anchors.]

**Persistence and side effects:**
[Explain with anchors.]

**Tests that cover it:**
[List tests or say "No direct coverage found" with evidence.]

**What juniors usually miss:**
- ...

**What seniors notice:**
- ...

**Drill:**
[Task]

**Self-grade:**
- Basic:
- Solid:
- Strong:
```

### Pattern Card Template

```md
## Pattern: [Name]

**Problem it solves:** ...

**General shape:** ...

**Real example:** `[path:lines]`

**Second example:** `[path:lines]` or "No second example found."

**Why this implementation works:** ...

**Failure modes:**
- ...

**Use it when:**
- ...

**Avoid it when:**
- ...

**Drill:** ...
```

### Ticket Template

```md
## Ticket [N]: [Title]

**Difficulty:** Easy | Medium | Hard | Expert
**Estimated time:** ...
**Skills practiced:** ...

**Story:** As a [user/maintainer], I want [behavior] so that [value].

**Why this is a good contribution:** ...

**Acceptance criteria:**
- [ ] ...

**Read these anchors first:**
- `[path:lines]` - [why]

**Files likely touched:**
- `[path]` - [change]

**Implementation plan:**
1. ...

**Illustrative fake-code shape:**
```language
// Illustrative fake code: adapt to the repo.
```

**What could go wrong:**
- ...

**Suggested checks:**
- ...

**Review questions:**
- ...
```

### Debugging Scenario Template

```md
## Scenario: [Symptom]

**Reproduction:** ...

**First question:** Is the bug in [layer A] or [layer B]?

**Narrowing path:**
1. Check ...
2. If yes, go to ...
3. If no, go to ...

**Useful probes:**
- Log/breakpoint/query/tool:

**Likely root causes:**
- ...

**Regression test to add:**
- ...

**Senior lesson:** ...
```

### Review Kata Template

```md
## Kata [N]: [PR Title]

**Author intent:** ...

**Fake diff summary:**
- ...

**Files this resembles:**
- `[path:lines]`

**Your task:** Review this PR.

**Expected findings:**

Blocking:
- ...

Important:
- ...

Optional:
- ...

**Good review comment example:**
> ...
```

## Final Response After Creating Docs

When you finish, respond with:

- Where you created the docs.
- The strongest parts of the curriculum.
- Any commands you ran and whether they passed.
- Any areas you could not verify.
- Suggested next step for the learner.

Keep the final response concise, but make sure it names the files that matter most.
````

## Why This Prompt Works

This prompt combines the strongest parts of two complementary documentation styles:

- Real-code anchoring, exact file paths, line references, trace tables, surgical tickets, fake-code contrasts, risk heatmaps, and Socratic self-grading.
- Generalizable stack mastery, first-principles language/framework explanations, quality engineering, security, performance, build projects, PR/RFC writing, and career-level collaboration.

The key improvement is that it asks the agent to avoid the two common failure modes:

- Producing a generic curriculum that could apply to any repo.
- Producing narrow onboarding docs that teach one code path deeply but do not build transferable engineering skill.

Use it as a full prompt for a large documentation pass, or paste only selected sections when you want a smaller artifact such as a ticket book, architecture critique, or debugging lab.
