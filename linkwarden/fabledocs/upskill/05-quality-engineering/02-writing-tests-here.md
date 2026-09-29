# Writing Tests Here

The house style, learned from the best existing suite (`apps/web/lib/api/archives/resolveAccessibleArchive.test.ts`): vitest, `vi.mock("@linkwarden/prisma")` with per-method `vi.fn()`, then assert **two things** — the returned `{status, response}` *and the exact where-clause* passed to Prisma (`resolveAccessibleArchive.test.ts:30-43`). Pinning the where-clause is what makes these *authorization* tests, not just unit tests: if someone deletes the `isPublic` OR-arm, a test fails even though every mocked call still "works."

Run commands: `yarn test` (all), `yarn test resolveAccessibleArchive` (filter by name), `yarn coverage`. __inferred__ (scripts verified in package.json; not executed here — no node_modules).

## Recipe 1: Happy path controller test
Template: mock prisma methods the controller calls → call controller with a valid `PostLinkSchemaType` body → assert status 200 and the create payload shape. Target: `postLink` (currently untested). Watch: it calls `setCollection`, `hasPassedLimit`, `fetchTitleAndHeaders`, `isUrlSafeForServerSideFetch` — mock at those module seams rather than reimplementing their behavior.

## Recipe 2: Validation failure
Call `postLink({url: "x".repeat(3000)}, 1)` → expect `{status: 400}` and message containing the zod path. No mocks needed until validation passes — a nice property: validation tests are nearly hermetic by construction (`postLink.ts:16-25` returns before any dependency runs).

## Recipe 3: Permission failure (the IDOR test)
The pattern from `resolveAccessibleArchive.test.ts` (stranger → mock `findFirst` resolves null → expect 401). Write the same for `updateLinkById`: member without `canUpdate` → `{status: 401, response: "Collection is not accessible."}` (`updateLinkById.ts:97-101`). Assert the *branch*, not just the code: also verify no `prisma.link.update` call happened (destructive-action-not-taken is the real assertion).

## Recipe 4: Cross-tenant rejection matrix
One `describe` per subject (owner / member+flag / member−flag / stranger / anonymous), table-driven with `it.each`. Model: the four cases in `resolveAccessibleArchive.test.ts` (owner, public+anonymous, denied, invalid params). Extend to a currently untested endpoint — `getLinkById` or `deleteLinkById`.

## Recipe 5: Async side-effect (worker logic)
Don't test `archiveHandler` whole (Playwright dependency). Extract-and-test: the archival-settings fallback (`archiveHandler.ts:85-107`) is a pure function of (tags, user) begging for extraction — cases: no archival tags → user defaults; one tag with `archiveAsPDF: true` → PDF on regardless of user; `aiTag` derived from `aiTaggingMethod !== DISABLED`.

## Recipe 6: Injected-seam test (the ssrf style)
`resolveHostnameForServerSideFetch(hostname, fakeLookup)` — pass a lookup returning private IPs, expect `UnsafeUrlError` (`ssrf.ts:259-304`; study `ssrf.test.ts` first). When you write new outbound-IO code, copy this signature style: dependency as trailing parameter with production default.

## Recipe 7: Route-level test with mocked auth
Model: `pages/api/v1/archives/[linkId].test.ts` (read it — it exercises the *route*, including method dispatch and the Bearer-only monolith rule `[linkId].ts:93-102`). Mock `verifyToken`/`verifyUser` modules; build fake `req`/`res` with vitest fns; assert status writes.

## Recipe 8: E2E with fixtures
Model: `e2e/tests/public/login.spec.ts` + fixtures (`e2e/fixtures/login-page.ts`). New specs: tag with `@yourtag` in the title so CI's grep matrix can target them (`playwright-tests.yml:45`); use the page-object pattern the fixtures establish; assert on visible outcomes (link card appears), never on network internals; do not wait for preservation (async — assert the row's *pre-archive* UI state).

## Flake prevention rules for this repo
No `waitForTimeout` — wait on locators/responses; keep worker OFF for e2e that don't need it (its writes mutate rows mid-test); fake timers for anything touching trial-window math (`getLinkBatchFairly.ts:60-68`); never assert on `createdAt` ordering across same-millisecond inserts (autoincrement `id` is the stable order — the code itself sorts by id for this reason, `getLinks.ts:15-16`).
