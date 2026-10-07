# Linkwarden Upskill Curriculum

A training lab built on this repository for one learner profile: a **junior fullstack JS engineer** (React/Node/TS CRUD experience) who wants to reach mid-level fast, build senior judgment, and pass interviews for mid-level fullstack roles.

Every page teaches two things at once:

1. **This codebase** — where things live, how its real flows work, with exact file/line anchors.
2. **Transferable skill** — why the pattern exists, its failure modes, and how to articulate it in an interview.

## What this repo is

Linkwarden is a self-hostable, collaborative bookmark manager. Users save links into **collections** (shareable with per-member `canCreate/canUpdate/canDelete` permissions), and a background **worker** preserves each link as a screenshot, PDF, single-file HTML (monolith), and readable article view using a headless Playwright browser. Links are tagged (optionally by an LLM), indexed into **Meilisearch** for full-text search, and served back through permission-checked API routes. It is a Yarn 4 workspaces monorepo: a Next.js (pages router) web app, a long-running Node worker, a React Native mobile app, and shared packages for the Prisma schema, validation/utility logic, React Query hooks, and an S3-or-local filesystem abstraction. Payments (Stripe) and email are optional — most behavior toggles on env vars, which makes the codebase a good study in *conditional infrastructure*.

Because the same features exist at every layer (UI modal → API route → controller → Prisma → background worker → search index), it is unusually good territory for practicing full-stack reasoning: nearly every interview topic — authz, SSRF, pagination, optimistic UI, idempotent background jobs, cache invalidation — has a real implementation here you can point at.

## How to use it

| Time budget | Path |
| --- | --- |
| One weekend | [00-fast-track.md](00-fast-track.md) — run it, trace two flows, make one safe change |
| Two weeks | Fast track → [01-cartography](01-codebase-cartography/README.md) → [05-key-flows](01-codebase-cartography/05-key-flows.md) → [03-architecture](03-architecture-and-patterns/README.md) → 2–3 drills from [04-code-reading-gym](04-code-reading-gym/README.md) |
| Eight weeks | All modules in order, one contribution ticket per week from [06-contribution-practice](06-contribution-practice/README.md) |
| Ongoing contributor | [06-contribution-practice](06-contribution-practice/README.md) + [07-career-and-collaboration](07-career-and-collaboration/README.md); use [09-reference/risk-register.md](09-reference/risk-register.md) as a source of test-writing work |
| **Interview in two weeks** | Go directly to [08-interview-prep/07-two-week-cram-plan.md](08-interview-prep/07-two-week-cram-plan.md) |

### Recommended paths by profile

- **Brand-new junior:** 00 → 01 (all) → 02 (all) → 04 drills → 06 good-first-tickets.
- **Junior who knows React/Node:** 00 → 01/05-key-flows → 03 (all) → 04 → 06 mid-level tickets → 08.
- **Mid-level, new to this repo:** 01/01-system-map + 01/05-key-flows → 03/06-architecture-critique → 06 senior projects.
- **Senior doing architecture review:** 03/06-architecture-critique → 09/risk-register → 05/05-security-checklist.
- **Candidate, interview in 2 weeks:** [08-interview-prep/07-two-week-cram-plan.md](08-interview-prep/07-two-week-cram-plan.md), which pulls from everything else.

## Other guides in this repo

`docs/` holds three other AI-written suites on the same codebase. They do not link to this curriculum, so expect overlap:

- `docs/architectural-cartographer/` — a top-down map in five parts (reading map, junior, mid-level, senior, reference). Overlaps most with `01-codebase-cartography` and `03-architecture-and-patterns`.
- `docs/mission-learning-path/` — 25 Socratic missions (8 junior, 9 mid-level, 7 senior, and a capstone in `04-reference-artifacts.md`). A good second pass after `04-code-reading-gym`.
- `docs/user-story-build-path/01-stories.md` — 10 feature stories from easy to expert, overlapping with `06-contribution-practice`.

Their code blocks are condensed and annotated rather than verbatim (see the note in each suite's README); the snippets in this curriculum are either real or labeled as fake.

## Conventions

- **File anchors** look like `apps/web/lib/api/verifyUser.ts:18-23` or as relative links. Line ranges were confirmed against the working tree when written (see [09-reference/verification-log.md](09-reference/verification-log.md)); if the code moves, search for the named symbol.
- **Fake code is labeled.** Every illustrative snippet that is not from this repo starts with `// Illustrative fake code: not from this repo`. Everything else is real.
- **Drills** are tasks *you* do — annotating, predicting, tracing — each with a self-grading rubric (Basic / Solid / Strong). Don't read the answers first.
- **Verification labels:** commands are marked __verified__ (actually run during authoring) or __inferred__ (read from scripts/CI but not executed here). Behavior claims that couldn't be fully verified are labeled *investigate* or *possible risk*, never asserted as bugs.
- **Interview angle** sections flag where a topic maps to a common interview question and cross-link into [08-interview-prep](08-interview-prep/README.md).

## Senior vocabulary

These words appear throughout, defined in context the first time each module uses them, because they're both engineering vocabulary *and* interview vocabulary: **invariant** (a condition that must always hold), **boundary** (where one component's responsibility ends), **contract** (the shape/behavior a caller may rely on), **ownership** (which code/team is responsible for a piece of data), **idempotency** (safe to run twice), **isolation** (one tenant/user can't see another's data), **authorization** (is this user *allowed*), **consistency** (do two stores agree), **latency**, **observability** (can you tell what it's doing in production), **migration**, **rollback**, **blast radius** (how much breaks if this goes wrong).

## The mindset ladder

- Junior asks: *"How do I make it work?"*
- Mid-level asks: *"Is this the right pattern? What are the tradeoffs?"*
- Senior asks: *"What does this commit us to, who pays the cost, and how do we reduce the risk?"*

Interviews for mid-level roles test exactly the second and third questions. A definition proves you read a book; a tradeoff anchored to real code proves you shipped software. This curriculum exists to give you that real code.
