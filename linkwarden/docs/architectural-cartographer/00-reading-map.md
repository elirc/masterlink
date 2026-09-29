# Reading Map

## Mental Model

Linkwarden is a preservation-first bookmark system: the web app collects links into collections and tags, API handlers authenticate and authorize the request, controllers enforce domain rules, Prisma stores the catalog, and a worker later handles preservation/search background work [README.md:30-36](../../README.md#L30), [apps/web/pages/api/v1/links/index.ts:9-42](../../apps/web/pages/api/v1/links/index.ts#L9), [apps/web/lib/api/controllers/links/postLink.ts:104-166](../../apps/web/lib/api/controllers/links/postLink.ts#L104), [apps/worker/worker.ts:15-19](../../apps/worker/worker.ts#L15).

## Top 10 Files To Read In Order

1. `package.json`  
   Why now: it tells you the monorepo shape, package manager, scripts, and test commands [package.json:3-26](../../package.json#L3).  
   Understand first: a workspace script like `web:dev` delegates to an app package [package.json:11-18](../../package.json#L11).  
   Explain after: how to start web, worker, Prisma migrations, tests, and formatting [package.json:10-26](../../package.json#L10).

2. `apps/web/pages/_app.tsx`  
   Why now: it is the web app's global shell [apps/web/pages/_app.tsx:36-118](../../apps/web/pages/_app.tsx#L36).  
   Understand first: React Query and NextAuth are providers around all pages [apps/web/pages/_app.tsx:48-114](../../apps/web/pages/_app.tsx#L48).  
   Explain after: why every page can use `useSession`, router hooks, toasts, and query hooks.

3. `apps/web/layouts/AuthRedirect.tsx`  
   Why now: it shows client navigation policy [apps/web/layouts/AuthRedirect.tsx:23-91](../../apps/web/layouts/AuthRedirect.tsx#L23).  
   Understand first: public pages bypass protection, protected path prefixes redirect unauthenticated users [apps/web/layouts/AuthRedirect.tsx:41-80](../../apps/web/layouts/AuthRedirect.tsx#L41).  
   Explain after: why client route protection is helpful but not sufficient for API security.

4. `packages/prisma/schema.prisma`  
   Why now: it is the domain dictionary [packages/prisma/schema.prisma:28-75](../../packages/prisma/schema.prisma#L28), [packages/prisma/schema.prisma:126-246](../../packages/prisma/schema.prisma#L126).  
   Understand first: users own collections and tags; links belong to collections; membership permissions live in `UsersAndCollections` [packages/prisma/schema.prisma:126-164](../../packages/prisma/schema.prisma#L126).  
   Explain after: how a shared collection differs from a public collection.

5. `packages/types/global.ts`  
   Why now: it adapts Prisma models into client/API shapes [packages/types/global.ts:1-35](../../packages/types/global.ts#L1).  
   Understand first: `LinkIncludingShortenedCollectionAndTags` makes date/id fields optional/string-compatible for client data [packages/types/global.ts:14-34](../../packages/types/global.ts#L14).  
   Explain after: how `Sort`, `ViewMode`, `ArchivedFormat`, and request query types flow through UI and API [packages/types/global.ts:81-112](../../packages/types/global.ts#L81), [packages/types/global.ts:160-173](../../packages/types/global.ts#L160).

6. `packages/lib/schemaValidation.ts`  
   Why now: Zod schemas are runtime contracts for forms and API controllers [packages/lib/schemaValidation.ts:125-148](../../packages/lib/schemaValidation.ts#L125).  
   Understand first: link payloads accept URL/name/description/image/collection/tags [packages/lib/schemaValidation.ts:125-146](../../packages/lib/schemaValidation.ts#L125).  
   Explain after: why validation happens both in `NewLinkModal` and `postLink` [apps/web/components/ModalContent/NewLinkModal.tsx:88-100](../../apps/web/components/ModalContent/NewLinkModal.tsx#L88), [apps/web/lib/api/controllers/links/postLink.ts:16-25](../../apps/web/lib/api/controllers/links/postLink.ts#L16).

7. `packages/router/links.tsx`  
   Why now: it is the client data highway for links [packages/router/links.tsx:26-60](../../packages/router/links.tsx#L26).  
   Understand first: `useFetchLinks` uses `/api/v1/search`, while `useAddLink` writes to `/api/v1/links` [packages/router/links.tsx:72-103](../../packages/router/links.tsx#L72), [packages/router/links.tsx:394-424](../../packages/router/links.tsx#L394).  
   Explain after: how optimistic updates can make the UI feel fast while the server validates later [packages/router/links.tsx:426-548](../../packages/router/links.tsx#L426).

8. `apps/web/pages/api/v1/links/index.ts`  
   Why now: it is a compact API route anatomy [apps/web/pages/api/v1/links/index.ts:9-71](../../apps/web/pages/api/v1/links/index.ts#L9).  
   Understand first: it verifies the user before branching on method [apps/web/pages/api/v1/links/index.ts:9-12](../../apps/web/pages/api/v1/links/index.ts#L9).  
   Explain after: how GET/POST/PUT/DELETE map to controllers.

9. `apps/web/lib/api/controllers/links/postLink.ts`  
   Why now: it is a real backend workflow, not a thin wrapper [apps/web/lib/api/controllers/links/postLink.ts:12-166](../../apps/web/lib/api/controllers/links/postLink.ts#L12).  
   Understand first: validation, SSRF safety, collection resolution, duplicate checks, capacity checks, type inference, Prisma write, folder creation [apps/web/lib/api/controllers/links/postLink.ts:16-166](../../apps/web/lib/api/controllers/links/postLink.ts#L16).  
   Explain after: why saving a link is more like checking an artifact into an archive than inserting one row.

10. `apps/worker/worker.ts`  
    Why now: it shows what leaves the request/response path [apps/worker/worker.ts:1-20](../../apps/worker/worker.ts#L1).  
    Understand first: migration, RSS polling, preservation, AI tagging, indexing, and email work are separate background jobs [apps/worker/worker.ts:11-20](../../apps/worker/worker.ts#L11).  
    Explain after: why link creation can return before every preserved format exists.

## Three Most Important Data Flows

1. Create a link: `NewLinkModal.submit` validates `PostLinkSchema`, `useAddLink` POSTs and optimistically updates caches, `/api/v1/links` verifies the user, `postLink` writes `Link` plus `Tag` relations and archive folder, then React Query replaces/refetches cache data [apps/web/components/ModalContent/NewLinkModal.tsx:88-100](../../apps/web/components/ModalContent/NewLinkModal.tsx#L88), [packages/router/links.tsx:397-548](../../packages/router/links.tsx#L397), [apps/web/pages/api/v1/links/index.ts:32-42](../../apps/web/pages/api/v1/links/index.ts#L32), [apps/web/lib/api/controllers/links/postLink.ts:104-166](../../apps/web/lib/api/controllers/links/postLink.ts#L104).

2. Read links/search: `useLinks` builds a query string, `useFetchLinks` calls `/api/v1/search`, the API route verifies the user, and `searchLinks` chooses Meilisearch or Prisma fallback before returning paginated links [packages/router/links.tsx:26-60](../../packages/router/links.tsx#L26), [packages/router/links.tsx:72-103](../../packages/router/links.tsx#L72), [apps/web/pages/api/v1/search/index.ts:6-35](../../apps/web/pages/api/v1/search/index.ts#L6), [apps/web/lib/api/controllers/search/searchLinks.ts:54-255](../../apps/web/lib/api/controllers/search/searchLinks.ts#L54).

3. Dashboard data: dashboard page calls `useCollections`, `useDashboardData`, and `useUser`; `/api/v2/dashboard` verifies the user; `getDashboardDataV2` fetches section-specific recent, pinned, and collection links [apps/web/pages/dashboard.tsx:30-69](../../apps/web/pages/dashboard.tsx#L30), [packages/router/dashboardData.tsx:16-34](../../packages/router/dashboardData.tsx#L16), [apps/web/pages/api/v2/dashboard/index.ts:7-25](../../apps/web/pages/api/v2/dashboard/index.ts#L7), [apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts:23-171](../../apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts#L23).

## Pre-Reading Checklist

1. Can I name the package manager and root scripts? Evidence: `package.json` [package.json:3-26](../../package.json#L3).
2. Can I explain what `apps/web`, `apps/worker`, and `packages/*` own? Evidence: workspaces and worker startup [package.json:6-9](../../package.json#L6), [apps/worker/worker.ts:1-20](../../apps/worker/worker.ts#L1).
3. Can I point to the app-level providers? Evidence: `_app.tsx` [apps/web/pages/_app.tsx:48-114](../../apps/web/pages/_app.tsx#L48).
4. Can I distinguish client route gating from API auth? Evidence: `AuthRedirect`, `verifyUser`, `verifyToken` [apps/web/layouts/AuthRedirect.tsx:23-91](../../apps/web/layouts/AuthRedirect.tsx#L23), [apps/web/lib/api/verifyUser.ts:14-72](../../apps/web/lib/api/verifyUser.ts#L14).
5. Can I name the primary domain models? Evidence: Prisma schema [packages/prisma/schema.prisma:28-75](../../packages/prisma/schema.prisma#L28), [packages/prisma/schema.prisma:126-246](../../packages/prisma/schema.prisma#L126).
6. Can I find a link's form schema? Evidence: `PostLinkSchema` [packages/lib/schemaValidation.ts:125-148](../../packages/lib/schemaValidation.ts#L125).
7. Can I find a link's client mutation? Evidence: `useAddLink` [packages/router/links.tsx:394-548](../../packages/router/links.tsx#L394).
8. Can I find a link's API handler? Evidence: `/api/v1/links` [apps/web/pages/api/v1/links/index.ts:9-71](../../apps/web/pages/api/v1/links/index.ts#L9).
9. Can I find a link's database write? Evidence: `prisma.link.create` [apps/web/lib/api/controllers/links/postLink.ts:104-150](../../apps/web/lib/api/controllers/links/postLink.ts#L104).
10. Can I find existing tests? Evidence: Vitest config and examples [vitest.config.mts:4-12](../../vitest.config.mts#L4), [apps/web/pages/api/v1/preserved/token.test.ts:50-120](../../apps/web/pages/api/v1/preserved/token.test.ts#L50).

## Red Flags Checklist

- New page added under a protected route but not listed in `AuthRedirect` [apps/web/layouts/AuthRedirect.tsx:41-58](../../apps/web/layouts/AuthRedirect.tsx#L41).
- API route accepts a method but does not verify the user before controller work [apps/web/pages/api/v1/links/index.ts:9-12](../../apps/web/pages/api/v1/links/index.ts#L9).
- Controller trusts body shape without Zod validation [apps/web/lib/api/controllers/links/postLink.ts:16-25](../../apps/web/lib/api/controllers/links/postLink.ts#L16).
- Link-related cache mutation updates one query family but forgets dashboard/collections/tags/public links [packages/router/links.tsx:543-546](../../packages/router/links.tsx#L543).
- New fetch path duplicates owner/member permission logic incorrectly [apps/web/lib/api/controllers/search/searchLinks.ts:197-213](../../apps/web/lib/api/controllers/search/searchLinks.ts#L197).
- Code assumes `localStorage` during render where SSR/hydration may matter [apps/web/layouts/MainLayout.tsx:13-22](../../apps/web/layouts/MainLayout.tsx#L13).
- Search behavior diverges between Meilisearch and Prisma fallback [apps/web/lib/api/controllers/search/searchLinks.ts:54-153](../../apps/web/lib/api/controllers/search/searchLinks.ts#L54), [apps/web/lib/api/controllers/search/searchLinks.ts:155-255](../../apps/web/lib/api/controllers/search/searchLinks.ts#L155).
- Dashboard ordering logic treats numeric IDs as dates [apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts:154-159](../../apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts#L154).
- Unsupported API methods silently return no explicit 405 [apps/web/pages/api/v1/collections/index.ts:13-29](../../apps/web/pages/api/v1/collections/index.ts#L13).

