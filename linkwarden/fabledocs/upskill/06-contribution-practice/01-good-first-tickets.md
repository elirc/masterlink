# Good First Tickets

15 junior tickets spread across the repo. All are **docs/tests/small-scope** by design — the point is practicing the *shape* of a good contribution: read anchors → small diff → tests → honest PR description. Every ticket ends with its interview-story potential.

> These are self-assigned training tickets. If you intend to upstream any, first check Linkwarden's real issue tracker and CONTRIBUTING conventions — maintainers reject unsolicited changes that don't match project direction.

---

## Ticket 1: Add a CI job that runs unit tests
Difficulty: Easy — 2-4h. Skills: GitHub Actions, CI reasoning.
Story: As a maintainer, I want `yarn test` to run on every PR so unit-tested code can't silently break.
Why good: CI currently runs only Playwright `@login` (`.github/workflows/playwright-tests.yml:45`); 8 vitest files exist but nothing enforces them.
Acceptance: - [ ] new workflow (or job) runs `yarn install && yarn test` on PR - [ ] green on current main - [ ] no Postgres needed (current unit tests are hermetic — verify).
Read first: `.github/workflows/playwright-tests.yml` (setup steps to copy), `vitest.config.mts`.
Files touched: `.github/workflows/unit-tests.yml` (new).
Plan: copy checkout/corepack/install steps; add `yarn test -- --run`.
Could go wrong: postinstall (playwright download) slows job — skip via env or `--mode=skip-build` research.
Checks: run `act` locally or push to a fork branch.
Review questions: should it also run `tsc --noEmit` for the worker? (separate ticket).
Interview story potential: "I noticed a production OSS repo enforced almost nothing in CI and closed that gap" — quality-mindset story.

## Ticket 2: Unit-test `searchQueryBuilder`'s token parser
Difficulty: Easy — 2-4h. Skills: vitest, parsing edge cases.
Story: As a maintainer, I want the search token parser covered so query syntax can evolve safely.
Why good: pure function, zero infra, real user-facing behavior (search operators).
Read first: `apps/web/lib/api/searchQueryBuilder.ts` (whole file), `packages/lib/ssrf.test.ts` (house test style).
Files touched: `apps/web/lib/api/searchQueryBuilder.test.ts` (new).
Plan: table-driven cases — plain terms, quoted phrases, tag:/collection: filters, empty string, weird unicode.
Could go wrong: assumptions about behavior — write tests that document *current* behavior; flag surprises in the PR instead of "fixing" silently.
Checks: `yarn test searchQueryBuilder`.
Interview story potential: "characterization tests before refactoring" — classic mid-level judgment story.

## Ticket 3: Unit-test `isArchivalTag` and archival-settings fallback
Difficulty: Easy — 1-2h. Skills: vitest, boolean-logic edge cases.
Why good: `archiveHandler.ts:85-107` picks per-tag settings over user defaults; the helper `packages/lib/isArchivalTag.ts` gates that. No coverage exists.
Acceptance: - [ ] cases: tag with all-null flags, mixed flags, aiTag only - [ ] documents the "any tag true wins" semantics.
Files touched: `packages/lib/isArchivalTag.test.ts` (new).
Interview story potential: small, but feeds the "how do per-item settings override defaults" design conversation.

## Ticket 4: Extract and test URL normalization from `postLink`
Difficulty: Medium — 3-5h. Skills: refactor-for-testability, TS.
Story: As a developer, I want the www/trailing-slash normalization (`postLink.ts:47-67`) as a named pure function so duplicate detection is testable.
Why small blast radius: pure extraction, same call site, tests pin behavior.
Read first: `postLink.ts:47-67`.
Files: `packages/lib/normalizeUrl.ts` + test (new), `postLink.ts` (call it).
Could go wrong: subtle behavior change (e.g. `replace` first-occurrence semantics) — tests first, extraction second.
Review questions: should normalization also apply at *query* time elsewhere?
Interview story potential: "made an untestable business rule testable without changing behavior."

## Ticket 5: Boot-time env summary log for the worker
Difficulty: Easy — 2-3h. Skills: DX/observability.
Story: As a self-hoster, I want the worker to log which features are active (Meili? S3? Stripe? AI provider? preservation on?) at startup so misconfiguration is visible.
Why good: debugging rounds 1 (../08-interview-prep/05) showed silent env-dependent behavior; this is the cheapest observability win.
Read first: `apps/worker/worker.ts:11-20`, conditional clients (`meilisearchClient.ts:1-10`, `s3Client.ts`).
Files: `apps/worker/worker.ts` (+ small helper).
Could go wrong: logging secret values — log presence booleans only.
Interview story potential: "how would you know it broke?" — you have a concrete answer you shipped.

## Ticket 6: Document the Link format-columns state machine
Difficulty: Easy — 2h. Skills: technical writing, invariant extraction.
Story: As a contributor, I want `null` vs `"unavailable"` vs path semantics of `Link.image/pdf/readable/monolith/preview` + `lastPreserved` documented next to the schema.
Anchors to encode: `postLink.ts:138-148`, `archiveHandler.ts:203-224`, `getLinkBatchFairly.ts:35-38`, `updateLinkById.ts:157-163`.
Files: comment block in `schema.prisma:166-198` or `packages/prisma/README.md`.
Interview story potential: "found an undocumented invariant and wrote it down" — reads as senior-in-training.

## Ticket 7: Test `getLinkBatchFairly` round-robin math
Difficulty: Medium — 4-6h. Skills: DB test setup or query-builder injection.
Why good: the worker's most intricate logic (`getLinkBatchFairly.ts:98-145`) has zero tests; the pure round-robin allocation can be tested by extracting the picking loop or via a test DB.
Plan: prefer extraction of the in-memory allocation into a pure function (ids in → picks out), test that; DB integration test is stretch.
Could go wrong: over-large refactor — keep the Prisma calls where they are, extract only the loop.
Interview story potential: strong — "added first tests to a scheduler, chose extraction over mocking" (see Story 4).

## Ticket 8: 401 vs 403 audit (report only)
Difficulty: Easy — 3h. Skills: HTTP semantics, audit writing.
Story: As a maintainer, I want a list of endpoints returning 401 for authorization failures (e.g. `updateLinkById.ts:92-101`, `resolveAccessibleArchive.ts:55-67` invalid-params-as-401) so we can decide on a convention.
Deliverable: a table (endpoint, current, correct, client impact) — **no code change**; status-code changes are breaking for API consumers, which is the lesson.
Interview story potential: "I found the bug class, then argued *against* immediately fixing it" — contract-awareness story.

## Ticket 9: Add `Retry-After`-style guidance to demo-mode responses
Difficulty: Easy — 1-2h. Skills: API consistency.
Story: demo-mode guards repeat a string literal in 4+ places in `links/index.ts:33-65` alone; centralize into a helper (`isDemoMode` exists — `apps/web/lib/api/isDemoMode.ts` is already used elsewhere) returning a consistent response.
Files: `links/index.ts` + grep for the literal.
Could go wrong: subtle copy differences between routes — diff them first.
Interview story potential: DRY-with-judgment (dedupe strings, not concepts).

## Ticket 10: Unit-test `resolveAccessibleArchive` public-collection path
Difficulty: Easy — 2h. Skills: reading existing tests, coverage thinking.
Why good: `resolveAccessibleArchive.test.ts` exists — read it, find the untested branch (e.g. anonymous user + `isPublic` + preview path `resolveAccessibleArchive.ts:70-72`), add cases.
Interview story potential: "extended a test suite by reading for uncovered branches" — small but demonstrates coverage literacy.

## Ticket 11: Type the `{response, status}` controller contract
Difficulty: Medium — 4h. Skills: TS generics, incremental typing.
Story: As a developer, I want a shared `ControllerResult<T>` type so controllers can't forget `status` and routes get typed responses.
Read first: `postLink.ts` return shapes, `getLinks.ts:140`, two routes.
Files: `packages/types/global.ts` (+ adopt in links controllers only — keep the diff reviewable).
Could go wrong: union explosions; keep `T` loose initially.
Interview story potential: "introduced a contract type incrementally instead of a big-bang refactor."

## Ticket 12: Document self-hosting storage layout
Difficulty: Easy — 2h. Skills: ops writing.
Story: As a self-hoster, I want to know what lives under `STORAGE_FOLDER` (`archives/{collectionId}/{linkId}{suffix}`, previews under `archives/preview/`) and what's safe to back up/delete.
Anchors: `packages/filesystem/createFile.ts`, path construction `resolveAccessibleArchive.ts:70-72`.
Interview story potential: minor; feeds ops-awareness talking points.

## Ticket 13: Add `p-limit`-style concurrency cap to RSS polling (behind discussion)
Difficulty: Medium — 3-5h. Skills: async control, restraint.
Story: As an operator of a large instance, I don't want every RSS feed fetched simultaneously (`rssPolling.ts:19-36` unbounded `Promise.all`).
Why this needs a design note first: single-user self-hosts don't care; is the dep worth it? Write the 5-line tradeoff before code — that's the exercise.
Could go wrong: changing polling semantics (per-feed error isolation must survive).
Interview story potential: "identified a thundering-herd risk and scoped the fix to actual impact."

## Ticket 14: E2E test for link creation
Difficulty: Medium — 4-6h. Skills: Playwright, fixtures.
Story: As a maintainer, I want a `@links` e2e (create link → appears in list) alongside `@login`.
Read first: `apps/web/e2e/fixtures/*`, `tests/public/login.spec.ts`, CI matrix (`playwright-tests.yml:45` — add the tag).
Could go wrong: waiting on archive completion (don't — assert the row, not preservation).
Interview story potential: "extended a minimal e2e suite; learned fixture design" — testing-strategy story.

## Ticket 15: Fix-or-document the `sort` NaN fallthrough
Difficulty: Easy — 2h. Skills: input validation, tiny-diff discipline.
Story: `Number(req.query.sort)` (`search/index.ts:16`, `links/index.ts:16`) yields NaN for garbage input, silently falling to default sort (`searchLinks.ts:26-30` if-chain). Either validate with zod (consistent with body validation) or document intended behavior.
Why good: teaches "silent coercion at the query-param boundary" in a 5-line diff.
Interview story potential: feeds the "where do you validate" answer with a war story.

---

Common review-rejection reasons to pre-empt in every PR: no linked issue/motivation; drive-by refactors bundled in; tests asserting implementation not behavior; breaking the API contract (status codes/response shapes) without calling it out; style drift from surrounding code.
