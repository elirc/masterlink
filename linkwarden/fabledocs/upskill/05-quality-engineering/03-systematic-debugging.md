# Systematic Debugging

The method: **reproduce → narrow (bisect the system, not the code) → hypothesize → test the cheapest probe → fix the root cause → add regression coverage.** The skill interviewers watch for is asking *bisection questions* ("is the bug above or below this boundary?") instead of opening files hopefully.

Tools that exist in this stack: browser devtools + React Query Devtools (dep in `apps/web/package.json`), worker console logs (every loop logs batches — `linkProcessing.ts:76-81`), Prisma query logging (this repo: set `DEBUG=true` — `packages/prisma/index.ts` switches the client to `["query","info","warn","error"]`), `yarn prisma:studio` to inspect rows, Playwright traces for e2e, `node --inspect` for the worker.

Timed interview versions of scenarios 1–4: [08-interview-prep/05](../08-interview-prep/05-debugging-and-code-review-rounds.md).

---

## Scenario 1: New links never finish archiving
Reproduction: add a link; spinner forever; worker container running.
First question: is the worker **selecting** the row (scheduler), **failing** on it (job), or **succeeding without the UI seeing it** (read path)?
Narrowing: 1. worker logs — does "Processing N links" print? If never → scheduler: check `getLinkBatchFairly.ts:45-77` filters (Stripe/trial/email-verified env coupling). 2. If prints then errors → job: the per-link catch logs reason (`linkProcessing.ts:58-68`). 3. If succeeds → check the row in prisma:studio; if columns updated, it's the client cache (does the UI ever refetch link status? — *investigate*, there's no polling hook obviously wired).
Useful probes: `SELECT count(*) FROM "Link" WHERE "lastPreserved" IS NULL;` — queue depth in one query.
Likely root causes: stray `STRIPE_SECRET_KEY`; unverified email with provider on; `DISABLE_PRESERVATION=true`; worker not running at all.
Regression: unit test the where-clause construction per env combination.
Senior lesson: env-conditional queries are invisible failure surfaces — a boot-time "active features" log (Ticket 5) converts this whole scenario into a 10-second diagnosis.
Interview narration: lead with the three-way bisection, not with "I'd check the logs."

## Scenario 2: Uploads fail only in production (works locally)
Reproduction: `POST /api/v1/archives/[linkId]` returns 400 on a 8MB PDF in prod, fine locally.
First question: which limit tripped — formidable's (`maxFileSize`, `[linkId].ts:193-197`), the manual buffer check (`:71-78`), Next's `responseLimit` (`:17-22`), or an *infrastructure* limit (reverse-proxy body cap) that never reaches Node?
Narrowing: 1. error message text identifies the app-level checks (each has distinct copy). 2. No app log at all + instant 413/400 → proxy (nginx `client_max_body_size`). 3. Env diff: `NEXT_PUBLIC_MAX_FILE_BUFFER` set differently in prod (build-time inlining! changing it requires rebuild — classic `NEXT_PUBLIC_` trap).
Probes: curl with a 1KB file (isolates size from format); check response headers for proxy signatures.
Likely root causes: proxy cap; stale `NEXT_PUBLIC_` value baked into the image.
Regression: document limits in one place; add an integration test at the boundary size.
Senior lesson: "works locally" bugs are usually *between* your code and the user — enumerate the middleboxes before re-reading your code.

## Scenario 3: A member sees "Collection is not accessible" moving a link they can edit
Reproduction: user is a member with `canUpdate: true`, drags link to another collection they're also a member of; 401.
First question: which of `updateLinkById.ts`'s *four* distinct rejections fired? (`:82-86` target mismatch, `:92-96` non-owner move, `:97-101` no update rights — plus zod).
Narrowing: match the response string to the branch — then read `:88-96`: **non-owners cannot move links between collections at all**, by design.
Probes: none needed beyond string-matching; the "bug" is a product rule.
Likely root cause: intended behavior, unclearly surfaced (UI let them try; error says "not accessible" instead of "only the owner can move links").
Regression: not a code fix first — a UX/copy ticket + a test documenting the rule.
Senior lesson: some bug reports are **specification discoveries**. The fix is making the invariant visible (disable the drag affordance for non-owners), not weakening the check. Debugging ends in a product conversation — say that in interviews; it's a strong signal.

## Scenario 4: Search misses links that definitely exist
Reproduction: link visible in collection view, absent from search.
First question: is the link **in the index** (write side) or **filtered out** (read side)?
Narrowing: 1. `indexVersion` column on the row — null/stale means never indexed → write side: is Meili configured? (`meiliClient` null ⇒ *different code path entirely* — Prisma `contains` fallback, so also test: does the miss reproduce with exact substring?). 2. Indexed but missing → read side: filters (`buildMeiliFilters` — tags/collection tokens), or the Prisma re-check dropping it (`searchLinks.ts:95-120` — membership correct?). 3. Indexing loop healthy? Its log prints "N left" (`linkIndexing.ts:170-175`).
Probes: query Meili directly (`curl $MEILI_HOST/indexes/links/search`) with the raw term — splits index-content from filter bugs in one step.
Likely root causes: indexing loop crashed/behind; `MEILI_INDEX_VERSION` bumped without letting reindex finish; tokenization (searching "link's" style terms).
Regression: reconciliation check (sampled count comparison DB vs index) as an admin/stats surface.
Senior lesson: for any dual-store bug, *first* establish which store is wrong; every minute before that is wasted.

## Scenario 5: Worker memory climbs until OOM (the capacity-math scenario)
Reproduction: import 5k links; watch RSS grow across hours.
First question: leak (unreleased per-job resources) or **legitimate peak** (batch × per-job memory exceeds container)?
Narrowing: 1. Do the math first: `ARCHIVE_TAKE_COUNT` (default 5) parallel jobs × (page + screenshot + pdf + monolith buffers, each capped by `*_MAX_BUFFER` env) — is the ceiling above the container limit? 2. If math says fine → leak-hunt: heap snapshots between batches (`node --inspect`), suspects: contexts not closed on the timeout path (check `finally` reaches `:229` in all paths), listeners accumulating on the shared browser. 3. Note the designers already suspect leakage: 30-min browser rotation (`linkProcessing.ts:29-33`) and a supervisor restart (`apps/worker/index.ts`) are mitigations, not fixes.
Probes: `process.memoryUsage()` log per batch — slope tells leak vs sawtooth in an hour.
Likely root causes: peak-memory under-provisioning (most common in the wild); genuine context leak on a rarely-hit error path.
Regression: capacity doc (max memory formula) + memory metric with alert threshold.
Senior lesson: distinguish *leak* from *peak* before profiling; and recognize rotation/restart patterns in a codebase as the authors telling you where the bodies are buried.
