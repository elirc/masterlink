# Refactor and Design Katas

Senior exercises. No production PRs required — the deliverable is a branch + write-up you could defend. Self-grading criteria per kata; "Strong" always includes stating what you deliberately did NOT do.

## Kata 1: Heal the boundary leak — unify the auth gates
Task: design (then optionally implement) the merge of `verifyUser.ts` and `isAuthenticatedRequest.ts` into one core + two adapters, eliminating the subscription-check drift (`verifyUser.ts:60-70` vs `isAuthenticatedRequest.ts:44-46`).
Self-grade — Solid: core returns a discriminated union; adapters preserve exact current behaviors *including the drift documented as a decision*. Strong: you wrote the behavioral diff table first and chose which semantics win, with a migration note for the loser's callers.

## Kata 2: Design an outbox — "index writes must not be lost"
Task: paper-design replacing the `indexVersion` polling protocol with an outbox table + dispatcher; then argue whether it's worth it.
Self-grade — Solid: correct outbox mechanics (same-transaction write, at-least-once dispatch, idempotent consumer). Strong: you conclude it's probably NOT worth it here (polling already gives at-least-once with less machinery) and can say precisely what requirement would flip the answer (multiple consumers / sub-second freshness).

## Kata 3: Split a module — `links.tsx` (1100+ lines)
Task: propose the file split for `packages/router/links.tsx` (queries / mutations / cache-editors / optimistic-assembly), with import-graph before/after.
Self-grade — Solid: cohesive modules, no cycles, public surface unchanged. Strong: you sequence it as 3 reviewable PRs and identify the one function whose move is riskiest (the shared `upsert*` helpers, used by multiple mutations).

## Kata 4: Remove duplication with judgment — the demo-mode guard
Task: extract the repeated demo-mode block (`links/index.ts:33-65` ×3, and its siblings across routes) into one wrapper; decide between higher-order handler vs early-return helper.
Self-grade — Solid: one implementation, all routes covered, responses byte-identical. Strong: you enumerated the routes where demo-mode should apply but currently *doesn't* (grep-driven) and filed that as the actually-valuable finding.

## Kata 5: Improve type safety where it pays — typed cache editors
Task: generic `InfiniteData<LinksPage>` types for the helpers at `links.tsx:162-264`, killing `oldData: any`.
Self-grade — Solid: compiles, behavior-identical (the Kata-4-from-review-katas trap avoided). Strong: a type test (`expectTypeOf`) pinning the page shape, and an honest note on what the types still can't catch (server/client drift — runtime shape unchanged).

## Kata 6: Design a migration — normalized URLs
Task: expand-migrate-contract plan for a `normalizedUrl` column backing the duplicate check (`postLink.ts:47-67`) with a unique partial index per owner.
Self-grade — Solid: 3-phase plan, backfill via AppMigration worker, old code tolerant throughout. Strong: you handled the collision question (two existing links that normalize identically — the backfill must *report*, not resolve) and the index's interaction with member-created links.

## Kata 7: Reduce N+1 — the scheduler rewrite
Task: replace `getLinkBatchFairly`'s per-user loops (`:87-96,122-128`) with one window-function query preserving fairness ordering.
Self-grade — Solid: correct SQL (`ROW_NUMBER() OVER (PARTITION BY ...)`) via `$queryRaw`, same picks on a worked example. Strong: characterization tests written against the *old* implementation first, then run against both; plus the honest "is it worth it" (measured tick cost) verdict.

## Kata 8: Write the RFC — retries (M3) as a formal document
Task: take mid-ticket M3 and write the full RFC using the template in [07/02](../07-career-and-collaboration/02-writing-prs-and-rfcs.md): context, goals/non-goals, design, alternatives (jobs table / external queue / status-quo), migration, rollout, open questions.
Self-grade — Solid: a maintainer could approve or reject it without asking clarifying questions. Strong: the alternatives section is strong enough that a reader could reasonably choose differently — that's what "steel-manned" means.

## Kata 9 (review): Grade a flawed PR under time pressure
Task: 15 minutes on Review Kata 8 (next-auth upgrade, [04/04](../04-code-reading-gym/04-review-katas.md)) writing the actual review comment thread.
Self-grade — Solid: caught the shared-secret/token coupling. Strong: your comments sequence the safe path (pin, smoke matrix, staged rollout) instead of just objecting.
