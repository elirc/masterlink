# Fast Track — one weekend

Goal: by Sunday night you can run Linkwarden locally, trace two end-to-end flows aloud, and have made one safe change with a passing test.

## 1. Install and run (Saturday morning)

All commands from the repo root. This repo uses **Yarn 4 via corepack** (`packageManager` field in [package.json](../../package.json)) and **Postgres**.

| Step | Command | Status |
| --- | --- | --- |
| Enable yarn 4 | `corepack enable && corepack prepare yarn@4.12.0 --activate` | __inferred__ (from `.github/workflows/playwright-tests.yml:71-75`) |
| Install deps | `yarn install` | __inferred__ |
| Configure env | copy `.env.sample` → `.env`; set `NEXTAUTH_URL=http://localhost:3000/api/v1/auth`, `NEXTAUTH_SECRET`, `DATABASE_URL=postgresql://user:pass@localhost:5432/linkwarden` | __inferred__ (var names from [.env.sample](../../.env.sample)) |
| Generate Prisma client | `yarn prisma:generate` | __inferred__ |
| Run migrations | `yarn prisma:deploy` | __inferred__ |
| Start web + worker | `yarn concurrently:dev` (or `yarn web:dev` alone) | __inferred__ (scripts in [package.json](../../package.json)) |
| Unit tests | `yarn test` (vitest, excludes e2e — see [vitest.config.mts](../../vitest.config.mts)) | __inferred__ |

No Postgres handy? `docker compose up` uses [docker-compose.yml](../../docker-compose.yml). The CI workflow is the authoritative "known-good" install recipe — read `.github/workflows/playwright-tests.yml` when stuck.

> The worker (`yarn worker:dev`) needs a Playwright Chromium; `yarn install` triggers `playwright install --with-deps chromium` via the web app's postinstall. The web app runs fine without the worker — links just stay unpreserved.

## 2. First 10 files to open, in order

1. [package.json](../../package.json) — workspace layout + every script you'll run.
2. [packages/prisma/schema.prisma](../../packages/prisma/schema.prisma) — the whole domain in 300 lines. Read `User`, `Collection`, `UsersAndCollections`, `Link`, `Tag`.
3. [apps/web/pages/api/v1/links/index.ts](../../apps/web/pages/api/v1/links/index.ts) — the canonical API route: auth first, then method switch into controllers.
4. [apps/web/lib/api/verifyUser.ts](../../apps/web/lib/api/verifyUser.ts) — the authentication gate every private route calls.
5. [apps/web/lib/api/controllers/links/postLink.ts](../../apps/web/lib/api/controllers/links/postLink.ts) — a full controller: validate → authorize → business rules → persist.
6. [apps/web/lib/api/getPermission.ts](../../apps/web/lib/api/getPermission.ts) — the authorization primitive.
7. [packages/router/links.tsx](../../packages/router/links.tsx) — shared React Query hooks; the client's whole data layer.
8. [apps/web/components/ModalContent/NewLinkModal.tsx](../../apps/web/components/ModalContent/NewLinkModal.tsx) — a representative UI feature using those hooks.
9. [apps/worker/worker.ts](../../apps/worker/worker.ts) — the whole worker entry point in 22 lines: one awaited `migrationWorker()`, then five `while (true)` workers (RSS polling, link processing, auto-tagging, indexing, trial-end emails).
10. [apps/worker/lib/archiveHandler.ts](../../apps/worker/lib/archiveHandler.ts) — the archiving engine; the most interesting file in the repo.

## 3. Trace two flows (Saturday afternoon)

Do these with the code open. Full trace tables live in [01-codebase-cartography/05-key-flows.md](01-codebase-cartography/05-key-flows.md).

**Flow A — adding a link:** UI modal (`NewLinkModal.tsx:88-100`, client-side zod parse) → `useAddLink` mutation with an optimistic cache insert using a negative temp id (`packages/router/links.tsx:426-520`) → `POST /api/v1/links` (`apps/web/pages/api/v1/links/index.ts:32-42`) → `postLink` controller: zod re-validation, SSRF check, collection permission, duplicate check, plan-capacity check, Prisma create (`postLink.ts:16-151`).

**Flow B — preserving that link:** worker loop picks a fair batch across users (`apps/worker/lib/getLinkBatchFairly.ts`) → `archiveHandler` re-checks SSRF, opens a Playwright page, races the work against a timeout (`archiveHandler.ts:110-198`) → finally-block marks leftover formats `"unavailable"` and stamps `lastPreserved` (`archiveHandler.ts:203-230`) → the indexing loop pushes it into Meilisearch (`apps/worker/workers/linkIndexing.ts:133-159`).

Pause-and-predict: before opening `archiveHandler.ts`, guess — what stops the same link from being archived twice? (Answer: `lastPreserved: null` in the batch query's where-clause, `getLinkBatchFairly.ts:35-38`. That's the idempotency key of the whole pipeline.)

## 4. One small safe change (Sunday)

Add a unit test for an existing pure function — zero product risk, real contribution shape. Candidates: `isArchivalTag` ([packages/lib/isArchivalTag.ts](../../packages/lib/isArchivalTag.ts)) or the www-normalization logic you saw in `postLink.ts:47-67` (extract-and-test is a stretch goal; testing `packages/lib/utils.ts` helpers is the safe version). Mimic the style of [packages/lib/ssrf.test.ts](../../packages/lib/ssrf.test.ts). Run `yarn test` and watch it pass.

## 5. Teach-back (Sunday night)

Explain Flow A aloud in 3 minutes as if an interviewer asked *"Walk me through how a write goes through a system you know well."* You must name: where validation happens (twice — client and server), where authorization happens (`setCollection` → `getPermission`), what's optimistic vs confirmed in the UI, and what happens asynchronously afterwards. Record yourself; if you said "and then it just saves it," do it again.

## What the fast track skips

Auth internals (NextAuth's 60-provider config), Stripe/billing, the mobile app, Meilisearch query building, i18n, migrations discipline, and all of the interview-prep material. That's the other 7 modules.
