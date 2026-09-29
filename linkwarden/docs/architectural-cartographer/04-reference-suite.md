# Reference Suite

## Doc 1: Junior Onboarding Guide

Start with `package.json` to learn scripts, workspaces, and test commands [package.json:3-26](../../package.json#L3). Then read `_app.tsx` for providers, `AuthRedirect` for page access rules, `schema.prisma` for the domain, `packages/router/links.tsx` for frontend data fetching, and `/api/v1/links` plus `postLink` for backend flow [apps/web/pages/_app.tsx:48-114](../../apps/web/pages/_app.tsx#L48), [apps/web/layouts/AuthRedirect.tsx:23-91](../../apps/web/layouts/AuthRedirect.tsx#L23), [packages/prisma/schema.prisma:28-308](../../packages/prisma/schema.prisma#L28), [packages/router/links.tsx:26-60](../../packages/router/links.tsx#L26), [apps/web/pages/api/v1/links/index.ts:9-71](../../apps/web/pages/api/v1/links/index.ts#L9), [apps/web/lib/api/controllers/links/postLink.ts:12-166](../../apps/web/lib/api/controllers/links/postLink.ts#L12).

Day-one commands are reconstructed from repo scripts and env samples: `yarn install`, copy `.env.sample` to `.env`, fill `NEXTAUTH_URL`, `NEXTAUTH_SECRET`, and `DATABASE_URL`, run `yarn prisma:generate`, `yarn prisma:dev`, and `yarn concurrently:dev` [package.json:10-26](../../package.json#L10), [.env.sample:3-12](../../.env.sample#L3).

## Doc 2: Mid-Level Architecture Guide

The primary boundary is browser UI -> React Query hooks -> Next API routes -> API controllers/helpers -> Prisma [packages/router/links.tsx:72-103](../../packages/router/links.tsx#L72), [apps/web/pages/api/v1/search/index.ts:6-35](../../apps/web/pages/api/v1/search/index.ts#L6), [apps/web/lib/api/controllers/search/searchLinks.ts:16-255](../../apps/web/lib/api/controllers/search/searchLinks.ts#L16), [packages/prisma/schema.prisma:1-8](../../packages/prisma/schema.prisma#L1). Server state belongs in `packages/router` hooks; ephemeral UI preferences and selection live in Zustand stores [packages/router/dashboardData.tsx:6-35](../../packages/router/dashboardData.tsx#L6), [apps/web/store/localSettings.ts:28-123](../../apps/web/store/localSettings.ts#L28), [apps/web/store/links.ts:12-42](../../apps/web/store/links.ts#L12).

The app's most important domain invariant is collection access: users can act on owned collections or member collections with the required permission flags [packages/prisma/schema.prisma:151-164](../../packages/prisma/schema.prisma#L151), [apps/web/lib/api/setCollection.ts:24-38](../../apps/web/lib/api/setCollection.ts#L24).

## Doc 3: Senior Ownership Guide

Own these risks first: route protection is manual path-prefix matching [apps/web/layouts/AuthRedirect.tsx:41-58](../../apps/web/layouts/AuthRedirect.tsx#L41), unsupported API methods lack explicit 405s [apps/web/pages/api/v1/collections/index.ts:13-29](../../apps/web/pages/api/v1/collections/index.ts#L13), dashboard sorting likely compares IDs incorrectly [apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts:154-159](../../apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts#L154), and search behavior diverges between Meilisearch and Prisma fallback [apps/web/lib/api/controllers/search/searchLinks.ts:54-153](../../apps/web/lib/api/controllers/search/searchLinks.ts#L54), [apps/web/lib/api/controllers/search/searchLinks.ts:155-255](../../apps/web/lib/api/controllers/search/searchLinks.ts#L155).

Do not over-refactor the whole app. First extract reusable permission predicates, type cache helpers, add route method guards, and add tests around link creation/search/collection permission [apps/web/lib/api/getPermission.ts:9-38](../../apps/web/lib/api/getPermission.ts#L9), [packages/router/links.tsx:162-222](../../packages/router/links.tsx#L162), [apps/web/lib/api/controllers/links/postLink.ts:32-39](../../apps/web/lib/api/controllers/links/postLink.ts#L32).

## Doc 4: Code Review Guide

Review in this order:

1. Route/auth: does every non-public API call `verifyUser` or a deliberate public helper? Compare `/api/v1/links` [apps/web/pages/api/v1/links/index.ts:9-12](../../apps/web/pages/api/v1/links/index.ts#L9).
2. Validation: does the controller use the relevant Zod schema? Compare `postLink` [apps/web/lib/api/controllers/links/postLink.ts:16-25](../../apps/web/lib/api/controllers/links/postLink.ts#L16).
3. Permissions: does collection/link access check owner and member paths? Compare `setCollection` [apps/web/lib/api/setCollection.ts:24-38](../../apps/web/lib/api/setCollection.ts#L24).
4. Persistence: does the Prisma write connect relations explicitly? Compare `prisma.link.create` [apps/web/lib/api/controllers/links/postLink.ts:104-150](../../apps/web/lib/api/controllers/links/postLink.ts#L104).
5. Cache: does a mutation update or invalidate all affected query keys? Compare `useAddLink` [packages/router/links.tsx:534-548](../../packages/router/links.tsx#L534).
6. Tests: is there a Vitest or Playwright path for this behavior? Existing patterns live in preserved API tests and login E2E tests [apps/web/pages/api/v1/preserved/token.test.ts:50-120](../../apps/web/pages/api/v1/preserved/token.test.ts#L50), [apps/web/e2e/tests/public/login.spec.ts:1-50](../../apps/web/e2e/tests/public/login.spec.ts#L1).

## Doc 5: Debugging Guide

For "I added a link and it disappeared": inspect `NewLinkModal.submit`, `useAddLink.onMutate`, API route, `postLink`, and `onError/onSuccess` rollback/invalidation [apps/web/components/ModalContent/NewLinkModal.tsx:88-100](../../apps/web/components/ModalContent/NewLinkModal.tsx#L88), [packages/router/links.tsx:426-548](../../packages/router/links.tsx#L426), [apps/web/pages/api/v1/links/index.ts:32-42](../../apps/web/pages/api/v1/links/index.ts#L32), [apps/web/lib/api/controllers/links/postLink.ts:16-166](../../apps/web/lib/api/controllers/links/postLink.ts#L16).

For "search results are wrong": inspect `buildQueryString`, `/api/v1/search`, and both Meilisearch/Prisma branches [packages/router/links.tsx:106-116](../../packages/router/links.tsx#L106), [apps/web/pages/api/v1/search/index.ts:13-35](../../apps/web/pages/api/v1/search/index.ts#L13), [apps/web/lib/api/controllers/search/searchLinks.ts:54-255](../../apps/web/lib/api/controllers/search/searchLinks.ts#L54).

For "user can/cannot access a collection": inspect Prisma `UsersAndCollections`, `getPermission`, and `setCollection` [packages/prisma/schema.prisma:151-164](../../packages/prisma/schema.prisma#L151), [apps/web/lib/api/getPermission.ts:27-36](../../apps/web/lib/api/getPermission.ts#L27), [apps/web/lib/api/setCollection.ts:24-38](../../apps/web/lib/api/setCollection.ts#L24).

## Doc 6: Change Playbook

To add a feature end-to-end:

1. Find the model field or add one in Prisma, then run/generate migrations [packages/prisma/schema.prisma:1-8](../../packages/prisma/schema.prisma#L1), [package.json:19-22](../../package.json#L19).
2. Add/extend Zod schema in `packages/lib/schemaValidation.ts` [packages/lib/schemaValidation.ts:125-179](../../packages/lib/schemaValidation.ts#L125).
3. Add/extend API route/controller under `apps/web/pages/api` and `apps/web/lib/api/controllers` [apps/web/pages/api/v1/links/index.ts:1-71](../../apps/web/pages/api/v1/links/index.ts#L1).
4. Add/extend React Query hook in `packages/router` [packages/router/links.tsx:394-548](../../packages/router/links.tsx#L394).
5. Update components under `apps/web/components` or pages under `apps/web/pages` [apps/web/components/ModalContent/NewLinkModal.tsx:102-180](../../apps/web/components/ModalContent/NewLinkModal.tsx#L102).
6. Add tests using Vitest or Playwright patterns [vitest.config.mts:4-12](../../vitest.config.mts#L4), [apps/web/playwright.config.ts:9-24](../../apps/web/playwright.config.ts#L9).

## Doc 7: Interview Walkthrough

"Linkwarden is a self-hosted collaborative bookmark manager that stores links in collections, preserves content formats, supports tags/highlights/public sharing, and runs background jobs for preservation and indexing" [README.md:30-66](../../README.md#L30), [apps/worker/worker.ts:15-19](../../apps/worker/worker.ts#L15).

"The web app uses Next.js Pages Router. `_app.tsx` installs React Query and NextAuth providers, then `AuthRedirect` gates routes and pages choose layouts with `getLayout`" [apps/web/pages/_app.tsx:26-34](../../apps/web/pages/_app.tsx#L26), [apps/web/pages/_app.tsx:48-114](../../apps/web/pages/_app.tsx#L48), [apps/web/layouts/AuthRedirect.tsx:23-91](../../apps/web/layouts/AuthRedirect.tsx#L23).

"For a link creation request, the modal validates a Zod-shaped payload, a React Query mutation posts to `/api/v1/links`, the API verifies the user, the controller validates and resolves permissions, Prisma creates the link and tag relations, and the client replaces optimistic cache data" [apps/web/components/ModalContent/NewLinkModal.tsx:88-100](../../apps/web/components/ModalContent/NewLinkModal.tsx#L88), [packages/router/links.tsx:397-548](../../packages/router/links.tsx#L397), [apps/web/pages/api/v1/links/index.ts:9-42](../../apps/web/pages/api/v1/links/index.ts#L9), [apps/web/lib/api/controllers/links/postLink.ts:16-166](../../apps/web/lib/api/controllers/links/postLink.ts#L16).

