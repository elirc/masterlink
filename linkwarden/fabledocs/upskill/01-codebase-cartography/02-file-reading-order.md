# File Reading Order

28 files. For each: why it matters, what to look for, what to ignore. Junior path = 1–14. Mid path adds 15–22. Senior path adds 23–28 and reads everything adversarially ("what would I change?").

## Foundation (everyone)

| # | File | Why / look for | Ignore |
| --- | --- | --- | --- |
| 1 | `package.json` (root) | workspace list, every runnable script, `resolutions` pinning React 19 | `patch-package` details |
| 2 | `packages/prisma/schema.prisma` | all 13 models; especially `UsersAndCollections:151-164` (permissions) and `Link:166-198` (nullable format columns = state machine) | Stripe fields on first pass |
| 3 | `.env.sample` | the feature-flag surface: storage, Meili, AI, Stripe, SSO all optional | AI provider variants |
| 4 | `apps/web/pages/api/v1/links/index.ts` | canonical route: gate → method switch → controller | demo-mode guards |
| 5 | `apps/web/lib/api/verifyUser.ts` | the authn gate; note it writes responses itself | Stripe branch |
| 6 | `apps/web/lib/api/verifyToken.ts` | JWT decode + `jti` revocation lookup (`:24-33`) | — |
| 7 | `apps/web/lib/api/getPermission.ts` | 39 lines; spot the sharp edge: linkId branch ignores userId | — |
| 8 | `apps/web/lib/api/controllers/links/postLink.ts` | full controller anatomy: validate→authorize→rules→persist | www-normalization details |
| 9 | `packages/lib/schemaValidation.ts` | scan exports; read `PostLinkSchema:125-148` closely | commented-out blocks |
| 10 | `packages/router/links.tsx:26-116` | `useLinks`/`useFetchLinks`; note list view actually calls `/api/v1/search` | cache helpers (later) |
| 11 | `apps/web/components/ModalContent/NewLinkModal.tsx` | representative feature component; client-side zod at `:88-96` | styling |
| 12 | `apps/worker/worker.ts` | six loops in 23 lines — the whole worker at a glance | — |
| 13 | `apps/worker/workers/linkProcessing.ts` | poll loop, shared browser, `Promise.allSettled:71-72` | log formatting |
| 14 | `apps/worker/lib/archiveHandler.ts` | the repo's crown jewel; read twice; the `finally:203-230` is the exam | monolith file-content juggling |

## Mid additions

| # | File | Why / look for |
| --- | --- | --- |
| 15 | `apps/worker/lib/getLinkBatchFairly.ts` | fairness scheduling; count the queries per tick |
| 16 | `packages/lib/ssrf.ts` | full SSRF guard; `assertUrlIsSafeForServerSideFetch:306-329` |
| 17 | `packages/lib/safeFetch.ts` | socket pinning `:16-57`, redirect re-validation `:103-124` |
| 18 | `apps/web/lib/api/controllers/links/linkId/updateLinkById.ts` | the permission matrix in anger |
| 19 | `apps/web/lib/api/controllers/search/searchLinks.ts` | Meili find → Postgres decide |
| 20 | `apps/worker/workers/linkIndexing.ts` | `indexVersion` protocol; denormalized authz fields `:133-143` |
| 21 | `packages/router/links.tsx:383-560` | `useAddLink` optimistic mutation end to end |
| 22 | `apps/web/pages/api/v1/archives/[linkId].ts` | file serving + uploads; MIME allowlist; formidable limits |

## Senior additions

| # | File | Why / look for |
| --- | --- | --- |
| 23 | `apps/web/lib/api/archives/resolveAccessibleArchive.ts` + its `.test.ts` | authz resolver AND the house test style |
| 24 | `apps/web/lib/api/preserved/createPreservedFormatUrl.ts` | signed scoped URLs; the type-guard mindset `:26-44` |
| 25 | `apps/web/pages/api/v1/auth/[...nextauth].ts` | env-driven provider config; adapter + callbacks |
| 26 | `apps/web/lib/api/setCollection.ts` | resolver+authz fusion; "Unorganized" default behavior |
| 27 | `.github/workflows/playwright-tests.yml` | what CI actually enforces (only `@login`) — a gap you'd fix |
| 28 | `packages/filesystem/createFile.ts` + `s3Client.ts` | env-selected backends; error swallowing (`return false`) |

Pause-and-predict prompts (use while reading): before #8 — "where will authorization happen?"; before #14 — "how does a failed archive avoid infinite retries?"; before #19 — "why would search hit the DB after the index answered?"

Senior noticing, per pass: hidden coupling (route ↔ controller `{response,status}` convention), implicit invariants (`lastPreserved` null = pending; format string `"unavailable"` = tried-and-failed), ownership boundaries (tags belong to the collection **owner**, not the link creator — `postLink.ts:120-137`), missing tests (worker: none), shape transformations (Prisma row → Meili doc `linkIndexing.ts:133-143`; DB link → optimistic link `links.tsx:487-505`).
