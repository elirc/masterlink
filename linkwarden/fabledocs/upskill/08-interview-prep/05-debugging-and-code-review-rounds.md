# Debugging & Code-Review Round Simulations

Timed simulations converted from [05-quality-engineering/03-systematic-debugging.md](../05-quality-engineering/03-systematic-debugging.md) and [04-code-reading-gym/04-review-katas.md](../04-code-reading-gym/04-review-katas.md). Run each with a timer; narrate aloud — interviewers grade the *process* (reproduce → narrow → hypothesize → cheap test → root cause → regression test), not the speed.

Grading rubric (all simulations): **Fail** = random file-opening, silent thinking, "it's probably X" without a test. **Pass (mid)** = states a bisection question, names the cheapest probe, finds root cause, proposes a regression test. **Strong** = also states the invariant that was violated and how to prevent the class of bug.

---

## Debugging Round 1 (25 min): "Links never finish archiving"

Setup you're given: self-hosted user reports all new links stay on a spinner; old links fine; no errors in the web UI.
Your opening move (say it): "The pipeline is web → DB row with `lastPreserved: null` → worker poll → files → format columns update. The UI spinner reads those columns, so first question: is the worker *running*, *picking up rows*, or *failing per-link*?"
Narrowing path: worker logs (it logs every batch — `linkProcessing.ts:76-81`) → if "0 left" never prints, check eligibility query filters (`getLinkBatchFairly.ts:45-77`: **subscription/trial filter when STRIPE_SECRET_KEY set**, email-verified filter when `NEXT_PUBLIC_EMAIL_PROVIDER=true`) → likely root cause seed: user set a stray `STRIPE_SECRET_KEY` in a self-hosted env, so no user has an active subscription and the batch is always empty.
Interviewer follow-ups: "How would you make this failure visible?" (queue-depth gauge + startup config summary log). "How do you prevent it?" (boot-time config validation: warn if Stripe key set but no price ids).
Regression test: unit test on the where-clause builder with/without `STRIPE_SECRET_KEY`.

## Debugging Round 2 (25 min): "Search returns a link the user can't open"

Setup: user searches, clicks a result, gets "You don't have access to this collection."
Opening move: "Search path is Meili-ids → Prisma re-check (`searchLinks.ts:67-120`); the open path is `resolveAccessibleArchive`/link fetch. If search shows it but open denies it, the two authz checks disagree — diff them."
Narrowing: compare search's where (`collection owner OR member` `searchLinks.ts:99-114`) vs archive's (`owner OR member OR isPublic`, `resolveAccessibleArchive.ts:38-42`) vs link-detail fetch → hypothesize: membership was just revoked; Meili's denormalized `collectionMemberIds` (`linkIndexing.ts:136`) is stale, and *this particular request path* trusted the index prefilter… but the Prisma re-check should have caught it. So next cheapest probe: reproduce with a fresh revocation and log both queries.
Root-cause candidates to voice: stale index + a path that skips the DB re-check (public tags/links routes), or React Query cache showing pre-revocation data client-side (`["links"]` cache never invalidated on membership change).
Interviewer follow-ups: "which store is the source of truth?"; "design the fix" (invalidate affected caches on membership mutation; optionally reindex collection's links on membership change).
Regression test: integration test — revoke membership, assert search results exclude within one poll cycle *or* document the accepted staleness.

## Debugging Round 3 (20 min): "Duplicate rows render in the list while scrolling"

Setup: intermittent; users see the same link twice mid-infinite-scroll; disappears on refresh.
Opening move: "Pagination consistency. Two suspects: offset-mode pagination under concurrent inserts, or the optimistic temp-id row not being reconciled."
Narrowing: does it correlate with Meili enabled? (`cursor` = offset there, `searchLinks.ts:64-65` — an insert during scroll shifts pages → duplicates). Or with adding a link mid-scroll (temp `-Date.now()` id + server row both present if `upsertLinkInInfiniteData` missed the swap — `links.tsx:162-191` replaces by `optimisticId`, but only in pages it finds it in).
Cheap probe: reproduce with Meili off → if gone, it's offset drift.
Interviewer follow-ups: "fix without changing Meili?" (dedupe by id at render/flatten time — `links.tsx:52-54` flatMap is the seam); "long-term?" (search-after cursors).
Regression test: unit test on the flatten helper with overlapping pages.

## Debugging Round 4 (20 min): "Worker memory climbs until the container OOMs"

Setup: self-hosted, heavy import (5k links); worker RSS grows for hours, then killed.
Opening move: "Long-lived process + per-job resources. Inventory what's created per link: browser context + page (`archiveHandler.ts:77-80`), buffers for screenshot/pdf/monolith, DB client. What's guaranteed released?"
Narrowing: context closed in `finally` (`:229` — with `.catch(()=>{})`); browser rotated every 30 min (`linkProcessing.ts:29-33` — an admission that leaks happen); suspects: pages that never resolve holding contexts for the full `BROWSER_TIMEOUT` × parallel batch, large file buffers held simultaneously (`Promise.allSettled` over batch = N pages' worth of buffers), monolith buffers.
Cheap probes: `--inspect` heap snapshot between batches; log `process.memoryUsage()` per batch; reduce `ARCHIVE_TAKE_COUNT` and observe slope.
Interviewer follow-ups: "why does rotating the browser 'fix' it and what does that tell you?"; "what limit prevents OOM regardless?" (container memory limit + smaller batch — capacity planning beats leak-whack-a-mole).
Regression: not a unit test — an ops guardrail: memory metric + alert; document max batch × max buffer math.

---

## Review Round 1 (20 min): "Add public collection RSS endpoint" (fake PR)

You review a diff adding `pages/api/v1/public/collections/rss.ts` that fetches a collection by id from `req.query` and returns its links as RSS — **no `isPublic` check**.
Expected findings — Blocking: missing `isPublic: true` in the where (IDOR; compare `resolveAccessibleArchive.ts:38-42`); unvalidated `Number(req.query.id)` NaN path. Important: no pagination (unbounded query); XML built by string concat (injection via link names). Optional: cache headers.
Say it kindly and concretely: "This exposes private collections — the public routes elsewhere gate on `isPublic` (see `resolveAccessibleArchive.ts:41`); can we add that to the where-clause and a test like `[linkId].test.ts`'s stranger case?"

## Review Round 2 (20 min): "Speed up duplicate check" (fake PR)

Diff replaces the two-variant URL match in `postLink.ts:47-67` with `findFirst({ where: { url } })`.
Expected findings — Blocking: drops the ownership scope (`collection: { ownerId: userId }`) — now matches *other users'* links: cross-tenant information leak via 409 timing, and wrong feature behavior. Important: loses www/trailing-slash normalization (regression for users). Optional: suggest DB-level normalized-url column + index as the real fix.
Lesson to voice: performance PRs that touch a where-clause are **authorization** PRs.

## Review Round 3 (15 min): "Retry failed archives" (fake PR)

Diff removes the `finally`-block stamping in `archiveHandler.ts:203-224` so failures stay `lastPreserved: null` and get retried.
Expected findings — Blocking: permanently-failing URLs now retry forever (poison pill) — worker spends its whole budget on them; need attempt counts/backoff before this is safe. Important: `"unavailable"` semantics break (UI treats null as "in progress" forever). Optional: propose the `archive_attempts` column design.
Lesson: deleting code that looks like a bug ("it marks failures as done!") without understanding *why* it's load-bearing — always ask what invariant the weird code protects.

## Review Round 4 (15 min): "Type-safe cache helpers" (fake PR)

Diff changes `upsertLinkInInfiniteData(oldData: any, …)` to a generic typed version but, mid-refactor, drops the `if (!replaced)` prepend branch (`links.tsx:182-188`).
Expected findings — Blocking: behavior change hidden in a "types-only" PR — new links no longer appear until refetch when absent from page 1. Important: types assert page shape that dashboardData variant doesn't share. Optional: praise the direction; request the refactor split types-only vs behavior.
Lesson: "refactor" PRs deserve *harder* behavioral scrutiny than feature PRs, because nobody's looking.
