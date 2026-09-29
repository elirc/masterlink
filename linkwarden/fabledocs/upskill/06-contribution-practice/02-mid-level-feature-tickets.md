# Mid-Level Feature Tickets

10 cross-layer tickets. **Each requires a one-page design note before implementation** (template in [07/02](../07-career-and-collaboration/02-writing-prs-and-rfcs.md)) covering: options considered, chosen tradeoff, risk, rollback. That habit *is* the mid-level skill.

## M1: Explicit archive status field
Difficulty: Medium — 2-3 days. Skills: API design, derived state, client updates.
Story: as an API consumer I want `link.archiveStatus: "pending"|"processing"|"done"|"failed"` instead of divining it from nullable columns.
Cross-layer: types (`packages/types/global.ts`) → derive server-side in link serializers → UI badges (`LinkViews/`) → mobile unaffected (additive).
Read first: the state machine — `postLink.ts:138-148`, `archiveHandler.ts:203-224`, [Trace 5](../04-code-reading-gym/02-trace-tables.md).
Risk: "processing" is not currently distinguishable from "pending" (no in-flight marker) — your design note must either add one (column write when batch picks) or merge the states honestly.
Rollback: additive field; remove on revert.
Interview story potential: "made an implicit state machine explicit for API consumers" — API-design story.

## M2: Move title-fetch off the write path
Difficulty: Medium-Hard — 2-4 days. Skills: async design, UX tradeoffs.
Story: `POST /links` p95 depends on the target site (`postLink.ts:78-88`); create instantly with url-as-name, let the worker set the title.
Design note must cover: how the UI learns the new name (refetch on preservation? optimistic "fetching title…" state), duplicate-name UX, and the *regression*: users who expect the title immediately.
Files: `postLink.ts` (remove fetch), `archiveHandler.ts` (title write already possible — meta extraction exists `:151-164`), `links.tsx` (optimistic name copy).
Risk/rollback: behavior change visible to users — feature-flag it (`DEFER_TITLE_FETCH`) so rollback = env flip.
Interview story potential: latency-budget story with a measured before/after — one of the strongest possible.

## M3: Bounded retries for failed archives
Difficulty: Hard — 3-5 days. Skills: schema migration, worker semantics, backfill.
The design from [critique #3-months plan step 3](../03-architecture-and-patterns/06-architecture-critique.md): `archiveAttempts` + `lastArchiveError` columns; eligibility `attempts < 3`; stamp-done after final attempt.
Design note: interaction with manual re-archive (which resets what?); poison-pill guarantee preserved; migration default (0) semantics for existing `"unavailable"` links (do NOT resurrect them — write why).
Tests: eligibility where-clause unit tests per attempt count.
Interview story potential: "added retry semantics to a queue without a queue" — reliability story, senior-tier.

## M4: Collection-level archive settings
Difficulty: Medium — 2-3 days. Skills: settings cascade design, null semantics.
The Kata 7 scenario ([04/04](../04-code-reading-gym/04-review-katas.md)) done right: nullable booleans on Collection (null = inherit), precedence tag > collection > user, resolution extracted into one tested helper (replacing `archiveHandler.ts:85-107` inline logic).
Rejection risk: product opinion — does the maintainer *want* this cascade? (Issue first.)
Interview story potential: "designed a 3-level settings cascade and its null semantics."

## M5: Error codes alongside messages
Difficulty: Medium — 2-3 days. Skills: API evolution, i18n decoupling.
Additive `code` field on error responses (start with links + collections controllers); client switches on code with message fallback (`links.tsx:521-524`).
Design note: code taxonomy (resource_not_accessible vs validation_failed…), stability promise, docs.
Interview story potential: "evolved an API error contract without breaking clients."

## M6: Rate limiting on auth + link creation
Difficulty: Medium-Hard — 3-4 days. Skills: middleware design, abuse thinking.
Per [08/03 Q11](../08-interview-prep/03-api-and-data-modeling-questions.md): token bucket keyed by IP (pre-auth) / userId (post-auth), interception at the `verifyUser` funnel, in-memory store with a Redis interface for later, observe-only mode first.
Design note: self-host defaults (generous), env knobs, 429 contract.
Interview story potential: "added rate limiting with an observe-first rollout" — production-maturity story.

## M7: Search index deletion propagation
Difficulty: Medium — 2 days. Skills: dual-store consistency.
Investigate then fix: do `deleteLinksById` / collection deletes remove Meili docs? (Curriculum flagged as unverified — your first task is establishing ground truth and writing it up.) If missing: delete-by-id on the index post-commit, plus a reconciliation count on the stats route.
Interview story potential: "found and closed a dual-store consistency gap" — the search-consistency interview answer with *you* in it.

## M8: Session/API-token last-used tracking surfaced in UI
Difficulty: Medium — 2 days. Skills: full-stack feature, privacy thinking.
`AccessToken.lastUsedAt` exists (`schema.prisma:243`) — *investigate* whether it's written on use (`verifyToken.ts` doesn't touch it); wire it (throttled write — not every request) and show in settings token list.
Design note: write amplification (update per request?) → throttle to 1/hour per token.
Interview story potential: "shipped security-visibility feature; handled write-amplification."

## M9: Dashboard data staleness fix for membership changes
Difficulty: Medium — 2-3 days. Skills: cache invalidation design.
When a user is added/removed from a collection, which client caches and search-index docs go stale? Map it (Flow 4 drill), then: server bumps affected links' `indexVersion` on membership change; client invalidates `["links"]`, `["collections"]`, `["dashboardData"]` on the membership mutations in `packages/router/collections.tsx`.
Interview story potential: the cache-invalidation war story every interviewer asks for.

## M10: Worker health endpoint v2
Difficulty: Medium — 2 days. Skills: observability implementation.
Extend `getWorkerStats.ts` with: queue depth, oldest-pending age, links marked unavailable in last 24h, index lag, RSS staleness (the five SQL statements from [05/06's drill](../05-quality-engineering/06-observability-and-operations.md)). Admin-only (check how `pages/admin` gates — `NEXT_PUBLIC_ADMIN` env).
Interview story potential: "built the health surface I wished I had while debugging" — pairs with any debugging story.

---

Rejection-risk review (all tickets): product-opinion changes (M2, M4) need maintainer buy-in first; schema changes (M1, M3, M8) must be additive with stated deploy-order; anything touching authz (M6, M9) needs the two-user test matrix in the PR itself.
