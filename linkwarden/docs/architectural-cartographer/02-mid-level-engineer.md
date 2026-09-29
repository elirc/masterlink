# Mid-Level Engineer Guide

## Full-Stack Architecture Diagram

```text
Browser
  |
  | Next.js page + layout
  | apps/web/pages/dashboard.tsx, apps/web/layouts/MainLayout.tsx
  v
React state layer
  |-- React Query server state: packages/router/*
  |-- Zustand UI state: apps/web/store/*
  v
Next.js API routes
  | apps/web/pages/api/v1/* and api/v2/*
  v
API helpers/controllers
  | verifyUser -> controller -> Prisma
  v
Postgres through Prisma
  | packages/prisma/schema.prisma
  v
Background worker
  | apps/worker/worker.ts for preservation/index/RSS/AI/email jobs
```

Evidence: dashboard imports router hooks and layout [apps/web/pages/dashboard.tsx:1-29](../../apps/web/pages/dashboard.tsx#L1), `_app` installs providers [apps/web/pages/_app.tsx:48-114](../../apps/web/pages/_app.tsx#L48), API routes call `verifyUser` and controllers [apps/web/pages/api/v1/links/index.ts:9-71](../../apps/web/pages/api/v1/links/index.ts#L9), Prisma models define the persisted domain [packages/prisma/schema.prisma:28-308](../../packages/prisma/schema.prisma#L28), and the worker starts background jobs [apps/worker/worker.ts:11-20](../../apps/worker/worker.ts#L11).

## Type System Deep Dive

The codebase uses three layers of type discipline:

- Runtime validation with Zod, such as `PostLinkSchema`, `UpdateLinkSchema`, `PostCollectionSchema`, and `UpdateDashboardLayoutSchema` [packages/lib/schemaValidation.ts:125-179](../../packages/lib/schemaValidation.ts#L125), [packages/lib/schemaValidation.ts:210-241](../../packages/lib/schemaValidation.ts#L210), [packages/lib/schemaValidation.ts:312-322](../../packages/lib/schemaValidation.ts#L312).
- Shared domain types that adapt Prisma models for frontend payloads, such as `LinkIncludingShortenedCollectionAndTags` [packages/types/global.ts:14-34](../../packages/types/global.ts#L14).
- Prisma-generated types/models from `schema.prisma`, including relation and enum definitions [packages/prisma/schema.prisma:28-97](../../packages/prisma/schema.prisma#L28), [packages/prisma/schema.prisma:126-198](../../packages/prisma/schema.prisma#L126).

Annotated contract:

```ts
export const PostLinkSchema = z.object({
  type: z.enum(["url", "pdf", "image"]).nullish(), // only three create-time link types
  url: z.string().trim().max(2048).url().optional(), // URL is optional for uploads/manual type
  name: z.string().trim().max(2048).optional(),
  description: z.string().trim().max(2048).optional(),
  image: z.enum(["jpeg", "png"]).optional(), // upload-derived image extension
  collection: z.object({
    id: z.number().optional(), // existing collection path
    name: z.string().trim().max(2048).optional(), // create/find-by-name path
  }).optional(),
  tags: z.array(z.object({
    id: z.number().optional(), // existing tag
    name: z.string().trim().max(50), // new or existing tag name
  })).optional() || [],
});
```

Source: `PostLinkSchema` [packages/lib/schemaValidation.ts:125-148](../../packages/lib/schemaValidation.ts#L125).

## State Management Deep Dive

Server state belongs to React Query. `useLinks` returns flattened pages plus the original infinite query object [packages/router/links.tsx:26-60](../../packages/router/links.tsx#L26). `useFetchLinks` uses query key `["links", { params }]`, calls `/api/v1/search`, supports mobile auth headers, and only runs when authenticated [packages/router/links.tsx:62-103](../../packages/router/links.tsx#L62).

Local UI state belongs to Zustand or local component state. Selected link IDs live in a simple `Record<number, true>` with `toggleSelected`, `clearSelected`, and `setSelected` [apps/web/store/links.ts:3-42](../../apps/web/store/links.ts#L3). Display preferences live in `localSettings`, are mirrored to `localStorage`, and affect view mode/color/columns/show fields [apps/web/store/localSettings.ts:28-123](../../apps/web/store/localSettings.ts#L28).

Annotated selected-state store:

```ts
const useLinkStore = create<LinkStore>()((set, get) => ({
  selectedIds: {}, // key lookup is faster and simpler than scanning arrays
  isSelected: (id) => !!get().selectedIds[id],
  toggleSelected: (id) =>
    set((state) => {
      const next = { ...state.selectedIds }; // immutable copy for Zustand update
      if (next[id]) {
        delete next[id];
        return { selectedIds: next, selectionCount: state.selectionCount - 1 };
      } else {
        next[id] = true;
        return { selectedIds: next, selectionCount: state.selectionCount + 1 };
      }
    }),
  clearSelected: () => set({ selectedIds: {}, selectionCount: 0 }),
}));
```

Source: `apps/web/store/links.ts` [apps/web/store/links.ts:12-42](../../apps/web/store/links.ts#L12).

## API Contract Map

- `GET /api/v1/search`: accepts `sort`, `cursor`, `collectionId`, `tagId`, `pinnedOnly`, and `searchQueryString`; returns `data.links` and `data.nextCursor` [apps/web/pages/api/v1/search/index.ts:13-35](../../apps/web/pages/api/v1/search/index.ts#L13), [apps/web/lib/api/controllers/search/searchLinks.ts:244-255](../../apps/web/lib/api/controllers/search/searchLinks.ts#L244).
- `POST /api/v1/links`: accepts `PostLinkSchemaType`, returns `{ response: newLink }`; protected by `verifyUser` and demo-mode guard [apps/web/pages/api/v1/links/index.ts:32-42](../../apps/web/pages/api/v1/links/index.ts#L32), [apps/web/lib/api/controllers/links/postLink.ts:12-25](../../apps/web/lib/api/controllers/links/postLink.ts#L12).
- `GET/POST /api/v1/collections`: returns or creates collections with user verification [apps/web/pages/api/v1/collections/index.ts:6-29](../../apps/web/pages/api/v1/collections/index.ts#L6).
- `GET/PUT /api/v2/dashboard`: fetches dashboard data or updates dashboard layout; protected and demo-mode guarded for writes [apps/web/pages/api/v2/dashboard/index.ts:7-38](../../apps/web/pages/api/v2/dashboard/index.ts#L7).
- `GET /api/v1/public/*`: public routes exist for collections, links, users, and tags under `pages/api/v1/public`; the public collection route validates an `id`, allows `GET`, and calls `getPublicCollection` [apps/web/pages/api/v1/public/collections/[id].ts:1-20](../../apps/web/pages/api/v1/public/collections/%5Bid%5D.ts#L1).

## Component Interaction Map

Dashboard is assembled like this:

```text
pages/dashboard.tsx
  uses useCollections, useDashboardData, useUser
  manages dashboardSections, viewMode, modals
  renders Section
    STATS -> DashboardItem
    RECENT_LINKS/PINNED_LINKS/COLLECTION -> DashboardLinks
    empty recent -> NewLinkModal trigger
  getLayout -> MainLayout
```

Evidence: imports and hooks [apps/web/pages/dashboard.tsx:1-45](../../apps/web/pages/dashboard.tsx#L1), section rendering [apps/web/pages/dashboard.tsx:133-205](../../apps/web/pages/dashboard.tsx#L133), `Section` switch [apps/web/pages/dashboard.tsx:229-428](../../apps/web/pages/dashboard.tsx#L229), layout attachment [apps/web/pages/dashboard.tsx:207-213](../../apps/web/pages/dashboard.tsx#L207).

## Full-Stack Feature Trace: Create A Link

1. UI state starts in `NewLinkModal.initial`; fields mirror `PostLinkSchemaType` [apps/web/components/ModalContent/NewLinkModal.tsx:23-44](../../apps/web/components/ModalContent/NewLinkModal.tsx#L23).
2. If the modal is opened inside a collection page, it preselects that collection from `useCollections` data [apps/web/components/ModalContent/NewLinkModal.tsx:45-82](../../apps/web/components/ModalContent/NewLinkModal.tsx#L45).
3. Submit validates `PostLinkSchema`, then calls `addLink.mutateAsync(link)` and closes the modal [apps/web/components/ModalContent/NewLinkModal.tsx:88-100](../../apps/web/components/ModalContent/NewLinkModal.tsx#L88).
4. `useAddLink` validates URL shape, posts JSON to `/api/v1/links`, and returns `data.response` [packages/router/links.tsx:397-424](../../packages/router/links.tsx#L397).
5. `onMutate` cancels link/dashboard queries, snapshots previous data, resolves existing collections/tags from cache, builds a negative temporary ID link, and inserts it into link and dashboard caches [packages/router/links.tsx:426-519](../../packages/router/links.tsx#L426).
6. API route calls `verifyUser`, blocks demo writes, then calls `postLink(req.body, user.id)` [apps/web/pages/api/v1/links/index.ts:9-42](../../apps/web/pages/api/v1/links/index.ts#L9).
7. `postLink` validates, checks SSRF safety, resolves the collection, checks duplicate preference and capacity, fetches title/headers if safe, infers content type, creates the link/tags, updates image path, creates archive folder, and returns status 200 [apps/web/lib/api/controllers/links/postLink.ts:16-166](../../apps/web/lib/api/controllers/links/postLink.ts#L16).
8. `onSuccess` replaces the optimistic link where possible and invalidates dashboard, collections, tags, and public links [packages/router/links.tsx:534-548](../../packages/router/links.tsx#L534).

## Diff Reading Exercise

Hypothetical change: "Add `metaDescription` to link cards."

Read this diff like an engineer:

1. Schema already has `Link.metaDescription` [packages/prisma/schema.prisma:190-191](../../packages/prisma/schema.prisma#L190), so the DB field exists.
2. Search omits only `textContent`, so `metaDescription` should be included in returned link records unless manually omitted elsewhere [apps/web/lib/api/controllers/search/searchLinks.ts:126-139](../../apps/web/lib/api/controllers/search/searchLinks.ts#L126), [apps/web/lib/api/controllers/search/searchLinks.ts:228-240](../../apps/web/lib/api/controllers/search/searchLinks.ts#L228).
3. The client type `LinkIncludingShortenedCollectionAndTags` extends `Link`, so the field is available through Prisma type inheritance [packages/types/global.ts:14-34](../../packages/types/global.ts#L14).
4. UI display probably belongs in `LinkCard`, `LinkMasonry`, or `LinkList`, not `Links`, because `Links` chooses layouts and delegates item rendering [apps/web/components/LinkViews/Links.tsx:118-150](../../apps/web/components/LinkViews/Links.tsx#L118), [apps/web/components/LinkViews/Links.tsx:321-355](../../apps/web/components/LinkViews/Links.tsx#L321).
5. Review risk: if the field can be long, it must respect existing card layout and settings controls [apps/web/store/localSettings.ts:7-17](../../apps/web/store/localSettings.ts#L7).

## Non-Obvious Architectural Patterns

- `packages/router` is a shared frontend API client layer, not a backend router; it centralizes query keys, fetch paths, optimistic cache logic, and mobile auth support [packages/router/links.tsx:1-24](../../packages/router/links.tsx#L1), [packages/router/collections.tsx:76-105](../../packages/router/collections.tsx#L76).
- The app uses optimistic cache writes for links because preservation is asynchronous and users need immediate feedback after saving a bookmark [packages/router/links.tsx:487-548](../../packages/router/links.tsx#L487), [apps/worker/worker.ts:15-18](../../apps/worker/worker.ts#L15).
- Shared collections use both owner relations and member permission flags; many queries therefore check `ownerId` OR `members.some({ userId })` [packages/prisma/schema.prisma:151-164](../../packages/prisma/schema.prisma#L151), [apps/web/lib/api/controllers/search/searchLinks.ts:197-213](../../apps/web/lib/api/controllers/search/searchLinks.ts#L197).
- User-facing dashboard layout is persisted as `DashboardSection` rows, not just local preferences [packages/prisma/schema.prisma:275-294](../../packages/prisma/schema.prisma#L275), [packages/router/dashboardData.tsx:37-83](../../packages/router/dashboardData.tsx#L37).

## Mid-Level Socratic Checkpoint

1. Why does `useLinks` call `/api/v1/search` instead of `/api/v1/links`?
2. What cache keys must be considered after creating a link?
3. Which permission rule decides if a user can create a link in a collection?
4. How do Meilisearch and Prisma fallback differ in pagination?
5. Why is `PostLinkSchemaType` better than a hand-written form interface here?
6. What is a possible bug in dashboard ordering?
7. Which state belongs in Zustand and which belongs in React Query?
8. How would you safely add a new field to link cards?

How to self-grade: strong answers cite `useFetchLinks` and search route [packages/router/links.tsx:72-103](../../packages/router/links.tsx#L72), invalidations after link creation [packages/router/links.tsx:543-546](../../packages/router/links.tsx#L543), `setCollection` permission logic [apps/web/lib/api/setCollection.ts:24-38](../../apps/web/lib/api/setCollection.ts#L24), Meilisearch offset and Prisma cursor paths [apps/web/lib/api/controllers/search/searchLinks.ts:64-82](../../apps/web/lib/api/controllers/search/searchLinks.ts#L64), [apps/web/lib/api/controllers/search/searchLinks.ts:192-250](../../apps/web/lib/api/controllers/search/searchLinks.ts#L192), the Zod schema [packages/lib/schemaValidation.ts:125-148](../../packages/lib/schemaValidation.ts#L125), the dashboard sort smell [apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts:154-159](../../apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts#L154), and local/server state boundaries [apps/web/store/links.ts:12-42](../../apps/web/store/links.ts#L12), [packages/router/links.tsx:72-103](../../packages/router/links.tsx#L72).
