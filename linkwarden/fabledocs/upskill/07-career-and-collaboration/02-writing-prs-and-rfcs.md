# Writing PRs and RFCs

## PR descriptions — the five sections

**What** (one sentence, imperative) · **Why** (link issue; the user/maintainer pain) · **How** (design choices a reviewer can't infer from the diff — *why this way*, not a diff narration) · **How tested** (commands run + what you did NOT test, honestly) · **Risks & follow-ups** (blast radius, rollback, filed follow-ups).

Worked example for [Ticket 4](../06-contribution-practice/01-good-first-tickets.md):

> **What:** Extract URL normalization from postLink into `packages/lib/normalizeUrl.ts` with unit tests.
> **Why:** The www/trailing-slash dedupe logic (postLink.ts:47-67) is untestable inline and is a candidate for reuse at query time (#issue).
> **How:** Pure extraction — behavior pinned by characterization tests written against the *old* code first. Kept the first-occurrence `replace` semantics even though `replaceAll` looks more correct; changing that is behavior, not refactoring (noted as follow-up).
> **Tested:** `yarn test normalizeUrl` (12 cases incl. no-www, multi-slash, non-http). Not tested: postLink integration (no existing harness — see follow-up #2).
> **Risks:** None expected — byte-identical logic; revert = single commit.

That "kept the weird semantics on purpose" line is what separates trusted contributors: it tells the reviewer you saw the landmine and stepped around it deliberately.

## Commit messages

Imperative subject ≤72 chars; body = why + notable non-obvious choice; one logical change per commit (extraction ≠ behavior change ≠ formatting — three commits). This repo has no visible convention to match (fresh clone, no history — [verification log](../09-reference/verification-log.md)); default to Conventional-ish (`test: add characterization tests for URL normalization`) and stay consistent.

## When to RFC (vs just PR)

RFC when any is true: schema change beyond additive; public API contract change; new dependency/service; touching authz semantics; work >1 week; two credible designs exist. Otherwise a good PR description suffices — RFC-ing trivia is its own anti-pattern.

## RFC template (tuned to this repo)

```
# RFC: <title>            Status: Draft | Review | Accepted | Rejected
## Context
Problem + evidence (anchors: file:line, issue links, measurements).
## Goals / Non-goals
Non-goals stop scope creep in review; list at least two.
## Design
Data model changes (schema.prisma diff sketch, migration phases).
API changes (routes, request/response, error codes; v1 compat statement).
Worker impact (new loops? eligibility predicates? deploy-order with web?).
Client impact (packages/router keys touched; mobile parity; cache invalidation).
Env/config surface (new vars → .env.sample entry).
## Alternatives considered
≥2, each with the reason NOT chosen stated fairly (steel-man).
## Migration & rollout
Expand/contract phases; flag name; rollback per phase; self-host upgrade notes.
## Test & observability plan
What proves it works; how we'd know it broke in prod.
## Open questions
```

The repo-specific sections (worker deploy-order, mobile parity, self-host notes) exist because this codebase has two deployables sharing a schema and an audience that upgrades unsupervised — your RFC template should always encode *your* system's standing risks.

Drill: Kata 8 in [06/04](../06-contribution-practice/04-refactor-and-design-katas.md) — write the M3 retries RFC with this template.
