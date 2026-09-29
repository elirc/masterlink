# Reference Artifacts

### Mission 25: Write the Docs That Don't Exist
**Tier:** Senior  
**Time Estimate:** 90 minutes  
**Goal:** Convert code-reading discoveries into workflow references an engineer can use while working.

**The Concept:** The best internal docs are not museum placards; they are checklists at the workbench. Linkwarden's workbench spans setup scripts, Next.js pages/API routes, React Query hooks, Prisma models, worker jobs, and tests [package.json:10-26](../../package.json#L10), [apps/web/pages/_app.tsx:48-114](../../apps/web/pages/_app.tsx#L48), [packages/router/links.tsx:394-548](../../packages/router/links.tsx#L394), [packages/prisma/schema.prisma:28-308](../../packages/prisma/schema.prisma#L28), [apps/worker/worker.ts:11-20](../../apps/worker/worker.ts#L11).

**Design Intent Before You Read the Code:** A workflow doc should tell a working engineer what to open, what to change, what to verify, and what failure modes to watch. It should cite the code that proves the workflow [apps/web/pages/api/v1/links/index.ts:9-71](../../apps/web/pages/api/v1/links/index.ts#L9), [apps/web/lib/api/controllers/links/postLink.ts:16-166](../../apps/web/lib/api/controllers/links/postLink.ts#L16).

**Find It In The Code:** Use these references while writing: setup [package.json:10-26](../../package.json#L10), env [`.env.sample:3-12`](../../.env.sample#L3), link flow [apps/web/components/ModalContent/NewLinkModal.tsx:88-100](../../apps/web/components/ModalContent/NewLinkModal.tsx#L88), API/auth [apps/web/lib/api/verifyUser.ts:14-72](../../apps/web/lib/api/verifyUser.ts#L14), tests [vitest.config.mts:4-12](../../vitest.config.mts#L4).

```text
Workflow reference pattern:
1. Start with the user-visible action.
2. Name the files in order.
3. Name the owner of each step.
4. Name one verification command or test.
5. Name one likely regression.
```

**The Aha Moment:** Documentation is strongest when it helps you take the next correct action.

**Socratic Checkpoint:**  
1. What workflow deserves a checklist first?  
2. Which exact files prove the workflow?  
3. What command verifies it?  
4. What bug should the doc help prevent?  
5. Who is the doc for: junior, reviewer, debugger, owner, or interviewer?

How to self-grade: strong answers cite setup scripts, link creation flow, auth checks, Prisma model, and test tooling [package.json:10-26](../../package.json#L10), [apps/web/components/ModalContent/NewLinkModal.tsx:88-100](../../apps/web/components/ModalContent/NewLinkModal.tsx#L88), [apps/web/lib/api/verifyUser.ts:14-72](../../apps/web/lib/api/verifyUser.ts#L14), [packages/prisma/schema.prisma:166-198](../../packages/prisma/schema.prisma#L166), [vitest.config.mts:4-12](../../vitest.config.mts#L4).

**Connects To:** Future feature work, because workflow docs become reusable operating procedures.

## Doc 1: Junior Onboarding Checklist

- Confirm Yarn 4 from `packageManager` [package.json:3](../../package.json#L3).
- Install dependencies with `yarn install`; root scripts expect Yarn workspaces [package.json:6-26](../../package.json#L6).
- Copy `.env.sample` to `.env`, fill `NEXTAUTH_URL`, `NEXTAUTH_SECRET`, and `DATABASE_URL` for manual setup [`.env.sample:3-12`](../../.env.sample#L3).
- Use Docker for Postgres and Meilisearch if not running them yourself [docker-compose.yml:1-28](../../docker-compose.yml#L1).
- Generate/migrate Prisma before feature work [package.json:19-22](../../package.json#L19).
- Start web and worker together with `yarn concurrently:dev` [package.json:17-18](../../package.json#L17).
- First PR checklist: cite the feature path, update schema/type/hook/API/UI together when needed, and run focused tests [packages/lib/schemaValidation.ts:125-148](../../packages/lib/schemaValidation.ts#L125), [packages/router/links.tsx:397-548](../../packages/router/links.tsx#L397), [vitest.config.mts:4-12](../../vitest.config.mts#L4).

## Doc 2: Architecture Guide For New Engineers

Navigate by flow:

1. App shell: `_app.tsx` providers and layout hook [apps/web/pages/_app.tsx:48-114](../../apps/web/pages/_app.tsx#L48).
2. Route access: `AuthRedirect` for client redirects [apps/web/layouts/AuthRedirect.tsx:23-91](../../apps/web/layouts/AuthRedirect.tsx#L23).
3. Server state: `packages/router` hooks [packages/router/links.tsx:26-60](../../packages/router/links.tsx#L26).
4. API entry: `apps/web/pages/api` [apps/web/pages/api/v1/links/index.ts:9-71](../../apps/web/pages/api/v1/links/index.ts#L9).
5. Controller/business logic: `apps/web/lib/api/controllers` [apps/web/lib/api/controllers/links/postLink.ts:12-166](../../apps/web/lib/api/controllers/links/postLink.ts#L12).
6. Data model: Prisma schema [packages/prisma/schema.prisma:28-308](../../packages/prisma/schema.prisma#L28).
7. Background work: worker startup [apps/worker/worker.ts:11-20](../../apps/worker/worker.ts#L11).

## Doc 3: Code Review Checklist

- Does a non-public API route call `verifyUser` before controller work? [apps/web/pages/api/v1/links/index.ts:9-12](../../apps/web/pages/api/v1/links/index.ts#L9)
- Does the controller validate with Zod? [apps/web/lib/api/controllers/links/postLink.ts:16-25](../../apps/web/lib/api/controllers/links/postLink.ts#L16)
- Does a write guard demo mode where existing write routes do? [apps/web/pages/api/v1/links/index.ts:32-38](../../apps/web/pages/api/v1/links/index.ts#L32)
- Are owner/member permissions respected? [apps/web/lib/api/setCollection.ts:24-38](../../apps/web/lib/api/setCollection.ts#L24)
- Does a mutation update/invalidate all affected query keys? [packages/router/links.tsx:543-546](../../packages/router/links.tsx#L543)
- Are unsupported methods handled explicitly? Existing routes need improvement here [apps/web/pages/api/v1/collections/index.ts:13-29](../../apps/web/pages/api/v1/collections/index.ts#L13).
- Are tests added for risky behavior using existing Vitest/Playwright patterns? [apps/web/pages/api/v1/preserved/token.test.ts:50-120](../../apps/web/pages/api/v1/preserved/token.test.ts#L50), [apps/web/e2e/tests/public/login.spec.ts:1-50](../../apps/web/e2e/tests/public/login.spec.ts#L1)

## Doc 4: Debugging Playbook

Link save fails:

1. Check client validation and form state [apps/web/components/ModalContent/NewLinkModal.tsx:23-100](../../apps/web/components/ModalContent/NewLinkModal.tsx#L23).
2. Check `useAddLink` fetch, error parsing, optimistic update, rollback [packages/router/links.tsx:397-548](../../packages/router/links.tsx#L397).
3. Check API auth/demo guard [apps/web/pages/api/v1/links/index.ts:9-42](../../apps/web/pages/api/v1/links/index.ts#L9).
4. Check `postLink` validation, SSRF, collection, duplicate, capacity, Prisma write [apps/web/lib/api/controllers/links/postLink.ts:16-166](../../apps/web/lib/api/controllers/links/postLink.ts#L16).

Search returns wrong links:

1. Check query string construction [packages/router/links.tsx:106-116](../../packages/router/links.tsx#L106).
2. Check API query conversion [apps/web/pages/api/v1/search/index.ts:13-35](../../apps/web/pages/api/v1/search/index.ts#L13).
3. Check Meilisearch branch and Prisma fallback [apps/web/lib/api/controllers/search/searchLinks.ts:54-255](../../apps/web/lib/api/controllers/search/searchLinks.ts#L54).

Auth/permission failure:

1. Check `verifyToken` [apps/web/lib/api/verifyToken.ts:9-35](../../apps/web/lib/api/verifyToken.ts#L9).
2. Check `verifyUser` [apps/web/lib/api/verifyUser.ts:14-72](../../apps/web/lib/api/verifyUser.ts#L14).
3. Check `getPermission` and `setCollection` [apps/web/lib/api/getPermission.ts:9-38](../../apps/web/lib/api/getPermission.ts#L9), [apps/web/lib/api/setCollection.ts:11-102](../../apps/web/lib/api/setCollection.ts#L11).

## Doc 5: Change Playbook

Branch -> implement -> test -> review -> merge:

1. Start from the user story and identify whether it is UI-only, API-only, or full-stack.
2. If persisted, update Prisma schema and migration path [packages/prisma/schema.prisma:1-8](../../packages/prisma/schema.prisma#L1), [package.json:19-22](../../package.json#L19).
3. Update Zod schema for accepted input [packages/lib/schemaValidation.ts:125-179](../../packages/lib/schemaValidation.ts#L125).
4. Update API handler/controller [apps/web/pages/api/v1/links/index.ts:9-71](../../apps/web/pages/api/v1/links/index.ts#L9), [apps/web/lib/api/controllers/links/postLink.ts:12-166](../../apps/web/lib/api/controllers/links/postLink.ts#L12).
5. Update router hook/cache behavior [packages/router/links.tsx:394-548](../../packages/router/links.tsx#L394).
6. Update UI component/page [apps/web/components/ModalContent/NewLinkModal.tsx:102-180](../../apps/web/components/ModalContent/NewLinkModal.tsx#L102).
7. Run focused Vitest or Playwright tests [vitest.config.mts:4-12](../../vitest.config.mts#L4), [apps/web/playwright.config.ts:9-24](../../apps/web/playwright.config.ts#L9).

## Doc 6: Senior Ownership Notes

Monitor:

- Link creation and preservation handoff [apps/web/lib/api/controllers/links/postLink.ts:104-166](../../apps/web/lib/api/controllers/links/postLink.ts#L104), [apps/worker/worker.ts:15-19](../../apps/worker/worker.ts#L15).
- Search parity with and without Meilisearch [apps/web/lib/api/controllers/search/searchLinks.ts:54-255](../../apps/web/lib/api/controllers/search/searchLinks.ts#L54).
- Auth/token/subscription gates [apps/web/lib/api/verifyUser.ts:18-72](../../apps/web/lib/api/verifyUser.ts#L18).
- Dashboard ordering and layout persistence [apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts:154-159](../../apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts#L154), [packages/prisma/schema.prisma:275-294](../../packages/prisma/schema.prisma#L275).

Improve:

- Central permission helpers, typed cache helpers, API method guards, and tests for collection permissions [apps/web/lib/api/getPermission.ts:9-38](../../apps/web/lib/api/getPermission.ts#L9), [packages/router/links.tsx:162-222](../../packages/router/links.tsx#L162), [apps/web/pages/api/v1/collections/index.ts:13-29](../../apps/web/pages/api/v1/collections/index.ts#L13).

Leave alone until needed:

- Working provider/layout structure, because it is easy to understand and used globally [apps/web/pages/_app.tsx:48-114](../../apps/web/pages/_app.tsx#L48).
- Worker/process separation, because it protects request responsiveness [apps/worker/worker.ts:11-20](../../apps/worker/worker.ts#L11).

## Doc 7: Interview Walkthrough

Practice answer:

"Linkwarden is a self-hosted collaborative bookmark and preservation app. Users save links into collections and tags, and background workers preserve or index the content" [README.md:30-66](../../README.md#L30), [apps/worker/worker.ts:15-19](../../apps/worker/worker.ts#L15).

"The web app is a Next.js Pages Router app. `_app.tsx` installs React Query and NextAuth providers; pages can use `getLayout`; `AuthRedirect` handles client redirects" [apps/web/pages/_app.tsx:26-39](../../apps/web/pages/_app.tsx#L26), [apps/web/pages/_app.tsx:48-114](../../apps/web/pages/_app.tsx#L48), [apps/web/layouts/AuthRedirect.tsx:23-91](../../apps/web/layouts/AuthRedirect.tsx#L23).

"The most important full-stack flow is link creation: form -> `useAddLink` -> `/api/v1/links` -> `verifyUser` -> `postLink` -> Prisma -> React Query cache replacement/invalidation" [apps/web/components/ModalContent/NewLinkModal.tsx:88-100](../../apps/web/components/ModalContent/NewLinkModal.tsx#L88), [packages/router/links.tsx:397-548](../../packages/router/links.tsx#L397), [apps/web/pages/api/v1/links/index.ts:9-42](../../apps/web/pages/api/v1/links/index.ts#L9), [apps/web/lib/api/controllers/links/postLink.ts:16-166](../../apps/web/lib/api/controllers/links/postLink.ts#L16).

