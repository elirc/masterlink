# Verification Log

Running record of what was inspected while authoring this curriculum. Date: **2026-07-09**. Environment: Windows 11, repo at `masterlink/linkwarden`, git branch `main` with **no commits** (fresh clone/init — `git log` returns "does not have any commits yet"), no `node_modules` installed.

## Commands run

| Command | Result |
| --- | --- |
| `ls` repo root, `apps/*`, `packages/*`, `docs/` | verified workspace layout (web/worker/mobile apps; filesystem/lib/prisma/router/types packages) |
| `cat package.json` (root + each workspace) | verified scripts and dependency lists quoted in docs |
| `git log --oneline` / `git remote -v` | no commits; no remote configured on the linkwarden repo itself |
| `grep -n "^model\|^enum" packages/prisma/schema.prisma` | 13 models / 6 enums; schema read in full (309 lines) |
| `ls packages/prisma/migrations \| wc -l` | 93 migration directories |
| `find . -name "*.test.ts"` (excl. node_modules) | 8 unit-test files, listed in "test inventory" below |
| `cat .github/workflows/playwright-tests.yml` | CI runs Playwright grep `@login` only; Postgres 16 service; yarn 4.12.0 via corepack |
| `cat vitest.config.mts` | vitest excludes `**/e2e/**`; alias `@` → `apps/web` |
| `grep -n` on `.env.sample` | env var inventory used in runtime/tooling map |

**Not run** (no dependencies installed; DB not provisioned): `yarn install`, `yarn test`, `yarn web:dev`, `yarn prisma:*`, Playwright e2e. All build/run commands in the curriculum are therefore marked __inferred__ from `package.json` scripts and the CI workflow.

## Files read in full (line anchors written from these reads)

- `apps/web/pages/api/v1/links/index.ts` (73 lines)
- `apps/web/pages/api/v1/archives/[linkId].ts` (314)
- `apps/web/pages/api/v1/search/index.ts` (partial, 1-50)
- `apps/web/lib/api/verifyUser.ts` (74), `verifyToken.ts` (37), `isAuthenticatedRequest.ts` (50), `getPermission.ts` (39), `setCollection.ts` (~105)
- `apps/web/lib/api/controllers/links/postLink.ts` (168), `getLinks.ts` (142), `linkId/updateLinkById.ts` (202)
- `apps/web/lib/api/controllers/search/searchLinks.ts` (1-120 of ~200)
- `apps/web/lib/api/archives/resolveAccessibleArchive.ts` (85)
- `apps/web/lib/api/preserved/createPreservedFormatUrl.ts` (98)
- `apps/web/components/ModalContent/NewLinkModal.tsx` (1-120 of ~200)
- `apps/web/hooks/useCollectivePermissions.ts` (33)
- `packages/router/links.tsx` (1-530 of ~1100; useLinks, useFetchLinks, cache helpers, useAddLink incl. onMutate/onError)
- `packages/prisma/schema.prisma` (309)
- `packages/lib/ssrf.ts` (349), `safeFetch.ts` (head + grep anchors), `schemaValidation.ts` (head 120 + export index + PostLinkSchema body), `meilisearchClient.ts` (10)
- `packages/filesystem/s3Client.ts`, `createFile.ts` (via head, no line-number capture — anchors to these files avoid tight ranges)
- `apps/worker/index.ts` (17), `worker.ts` (23), `workers/linkProcessing.ts` (86), `workers/linkIndexing.ts` (203), `workers/rssPolling.ts` (~40), `lib/getLinkBatchFairly.ts` (180), `lib/archiveHandler.ts` (264), `lib/autoTagLink.ts` (head 50)
- `apps/web/pages/api/v1/auth/[...nextauth].ts` (head 80 — provider imports + adapter setup only)
- `apps/web/lib/api/archives/resolveAccessibleArchive.test.ts` (1-80 — house test style: `vi.mock("@linkwarden/prisma")`, where-clause assertions)
- `packages/prisma/index.ts` (full — global-cached PrismaClient singleton; `DEBUG=true` query logging)
- `.yarnrc.yml` (full — `nodeLinker: node-modules`, `nmMode: hardlinks-local`)

## Post-authoring corrections

Two claims were corrected during writing after re-verification: (1) the Prisma generator uses the default `prisma-client-js` output, not a custom path — the facade is `packages/prisma/index.ts`'s singleton, and the tooling doc was fixed to say so; (2) Prisma query logging in this repo is enabled via `DEBUG=true` (custom client config), not the generic `DEBUG=prisma:query` — the debugging doc was fixed. All 53 curriculum files verified present on disk at completion.

## Test inventory (verified by find)

`apps/web/lib/api/archives/resolveAccessibleArchive.test.ts`, `apps/web/lib/api/controllers/migration/importFromHTMLFile.test.ts`, `apps/web/lib/api/preserved/createPreservedFormatUrl.test.ts`, `apps/web/pages/api/v1/archives/[linkId].test.ts`, `apps/web/pages/api/v1/config/index.test.ts`, `apps/web/pages/api/v1/preserved/token.test.ts`, `apps/web/pages/api/v1/preserved/view.test.ts`, `packages/lib/ssrf.test.ts`. E2E: `apps/web/e2e/` with fixtures; CI greps `@login` only.

## 2026-10-06 — accuracy pass against the committed snapshot

Re-checked statically against `elirc/masterlink` (commit `d9d387c`); nothing was installed or run. The environment line above is stale: the tree is now committed in its own repository.

- All 970 relative Markdown links under `fabledocs/` and `docs/` resolve, and every backticked `path:line` citation falls inside its file. The backticked paths that do not exist (`packages/lib/normalizeUrl.ts`, `.github/workflows/unit-tests.yml`, `pages/api/v1/public/collections/rss.ts`, `apps/web/pages/api/v1/links/defaults.ts` and similar) are all files a ticket or review exercise asks you to create.
- `apps/worker/worker.ts` is 22 lines and starts one awaited migration plus five `while (true)` workers; the "6 loops in 23 lines" wording in the fast track and reading order was corrected.
- Spot-read and still accurate: the yarn 4.12.0 corepack step at `.github/workflows/playwright-tests.yml:71-75`, the root `package.json` scripts (`concurrently:dev`, `prisma:deploy`, `test` = vitest), `schema.prisma` at 308 lines, and `PostLinkSchema` in `packages/lib/schemaValidation.ts`.

## Uncertainties / not covered

- **apps/mobile** was not explored beyond its existence; curriculum claims about it are limited to "React Native app sharing `packages/router` hooks" (supported by resolutions in root package.json and router's design).
- `[...nextauth].ts` read only to line 80; claims limited to provider breadth + adapter + env toggles.
- `searchLinks.ts` beyond line 120 (non-Meili branch) inferred to mirror `getLinks.ts`; labeled as such where cited.
- `rssHandler.ts`, preservation scheme handlers (`handleMonolith`, `handleReadability`, etc.), Stripe libs, migration importers, dashboard controllers, admin pages: skimmed or unread; cited only by path/purpose, no line anchors.
- Runtime behavior (actual archive of a page, Meili round-trip, login flow) was **not executed**; all behavioral claims derive from code reading.
- The three pre-existing `docs/` folders (`architectural-cartographer`, `mission-learning-path`, `user-story-build-path`) were left untouched and unexamined beyond names.

## Suspicions intentionally labeled in the curriculum (not confirmed bugs)

- Divergence between `verifyToken.ts` and `isAuthenticatedRequest.ts` subscription handling.
- `cursor` param means offset in Meili branch vs Prisma cursor in fallback (`searchLinks.ts:65` vs `getLinks.ts:95-96`).
- Unbounded `Promise.all` in `rssPolling.ts:19-36` on large installs.
- `archiveHandler.ts` finally-block stamping `lastPreserved` on failure = no retry semantics.
- Widespread 401-where-403 status codes in controllers.
