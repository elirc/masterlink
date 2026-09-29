# Mission Learning Path Journal

## Mission Design Rationale

The missions are ordered from system heartbeat to feature ownership. The early missions teach where runtime begins: package scripts, `_app`, layouts, and folder boundaries [package.json:10-26](../../package.json#L10), [apps/web/pages/_app.tsx:48-114](../../apps/web/pages/_app.tsx#L48). The middle missions trace data movement: React Query hooks, Zustand stores, API routes, middleware/auth helpers, controllers, and Prisma [packages/router/links.tsx:26-60](../../packages/router/links.tsx#L26), [apps/web/store/links.ts:12-42](../../apps/web/store/links.ts#L12), [apps/web/lib/api/verifyUser.ts:14-72](../../apps/web/lib/api/verifyUser.ts#L14), [apps/web/lib/api/controllers/links/postLink.ts:16-166](../../apps/web/lib/api/controllers/links/postLink.ts#L16). The senior missions train suspicion: performance, security, testing, diff review, and history interpretation [apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts:154-159](../../apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts#L154), [apps/web/lib/api/controllers/search/searchLinks.ts:54-255](../../apps/web/lib/api/controllers/search/searchLinks.ts#L54), [vitest.config.mts:4-12](../../vitest.config.mts#L4).

## Why These Code Paths

Link creation was chosen as the main end-to-end path because it exercises UI form state, schema validation, React Query mutation, optimistic cache, API auth, permission checks, Prisma relations, archive folder creation, and UI cache refresh [apps/web/components/ModalContent/NewLinkModal.tsx:23-100](../../apps/web/components/ModalContent/NewLinkModal.tsx#L23), [packages/router/links.tsx:394-548](../../packages/router/links.tsx#L394), [apps/web/pages/api/v1/links/index.ts:9-42](../../apps/web/pages/api/v1/links/index.ts#L9), [apps/web/lib/api/controllers/links/postLink.ts:16-166](../../apps/web/lib/api/controllers/links/postLink.ts#L16).

Search and dashboard were chosen as secondary paths because they teach read-side complexity, pagination, query keys, optional Meilisearch, and personalized sections [packages/router/links.tsx:72-103](../../packages/router/links.tsx#L72), [apps/web/pages/api/v1/search/index.ts:13-35](../../apps/web/pages/api/v1/search/index.ts#L13), [apps/web/lib/api/controllers/search/searchLinks.ts:54-255](../../apps/web/lib/api/controllers/search/searchLinks.ts#L54), [packages/router/dashboardData.tsx:16-34](../../packages/router/dashboardData.tsx#L16), [apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts:23-171](../../apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts#L23).

## Skill By Tier

- Junior tier: locate entry points, read folders, recognize typed props, follow navigation, and explain a component without changing it.
- Mid-level tier: connect state to API, read side effects, inspect middleware/auth chains, review diffs, and explain composition.
- Senior tier: infer architectural tradeoffs, find likely bugs, audit security/performance, write missing tests, and reason from migrations/history.

## What A Senior Does Differently

A senior engineer does not stop at "this works." They ask whether route protection is duplicated or brittle [apps/web/layouts/AuthRedirect.tsx:41-58](../../apps/web/layouts/AuthRedirect.tsx#L41), whether cache updates include every affected query family [packages/router/links.tsx:543-546](../../packages/router/links.tsx#L543), whether permission logic is centralized enough [apps/web/lib/api/getPermission.ts:9-38](../../apps/web/lib/api/getPermission.ts#L9), whether search behaves the same with and without Meilisearch [apps/web/lib/api/controllers/search/searchLinks.ts:54-255](../../apps/web/lib/api/controllers/search/searchLinks.ts#L54), and whether tests cover user-visible risk [apps/web/pages/api/v1/preserved/token.test.ts:50-120](../../apps/web/pages/api/v1/preserved/token.test.ts#L50).

