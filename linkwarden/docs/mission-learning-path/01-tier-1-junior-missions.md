# Tier 1 Junior Missions

### Mission 1: The App's Heartbeat
**Tier:** Junior  
**Time Estimate:** 25 minutes  
**Goal:** Explain how the repo starts, builds, migrates, and tests.

**The Concept:** A codebase has a heartbeat: scripts that make it run repeatedly. In Linkwarden, the heartbeat starts the web app, the archive worker, Prisma migrations, and tests, like opening the archive desk, the preservation room, and the catalog database at the same time [package.json:10-26](../../package.json#L10).

**Design Intent Before You Read the Code:** `package.json` owns workspace scripts and the package manager; `.env.sample` owns setup variables; `docker-compose.yml` owns local service wiring. If this area is misunderstood, engineers run the app without a database, worker, or auth secret [package.json:3-26](../../package.json#L3), [.env.sample:3-12](../../.env.sample#L3), [docker-compose.yml:1-28](../../docker-compose.yml#L1).

**Find It In The Code:** Open `package.json:3-26`, `.env.sample:3-12`, `docker-compose.yml:1-28`.

```json
"packageManager": "yarn@4.12.0+sha512.f45ab632439a67f8bc759bf32ead036a1f413287b9042726b7cc4818b7b49e14e9423ba49b18f9e06ea4941c1ad062385b1d8760a8d5091a1a31e5f6219afca8",
// Use Yarn because the repo pins Yarn 4.
"workspaces": ["apps/*", "packages/*"],
// Runtime code is split into app packages and shared packages.
"web:dev": "dotenv -- yarn workspace @linkwarden/web dev",
// Web dev runs through the web workspace with .env loaded.
"worker:dev": "dotenv -- yarn workspace @linkwarden/worker dev",
// Worker dev is separate because preservation/index/RSS jobs are not page requests.
"prisma:dev": "dotenv -- yarn workspace @linkwarden/prisma dev",
// Schema changes must go through Prisma migrations.
"test": "vitest"
// Unit/API tests use Vitest from the repo root.
```

**The Aha Moment:** Before reading a feature, learn the commands that make the feature observable.

**Socratic Checkpoint:**  
1. Which command starts only the web app?  
2. Which command starts the worker?  
3. Which command applies development migrations?  
4. Which env variables are essential for auth and DB?  
5. What local services does Docker provide?

How to self-grade: strong answers cite `web:dev`, `worker:dev`, `prisma:dev`, `NEXTAUTH_URL`, `NEXTAUTH_SECRET`, `DATABASE_URL`, Postgres, and Meilisearch [package.json:10-26](../../package.json#L10), [.env.sample:3-12](../../.env.sample#L3), [docker-compose.yml:1-28](../../docker-compose.yml#L1).

**Connects To:** Mission 2, because scripts only make sense once you know the folders they target.

### Mission 2: The Folder Mental Map
**Tier:** Junior  
**Time Estimate:** 30 minutes  
**Goal:** Build a mental map of app packages and shared packages.

**The Concept:** A monorepo is a set of rooms around the same archive: the public desk is `apps/web`, the preservation lab is `apps/worker`, and shared policies/tools are in `packages/*` [package.json:6-9](../../package.json#L6), [apps/worker/worker.ts:1-20](../../apps/worker/worker.ts#L1).

**Design Intent Before You Read the Code:** `apps/web` owns UI and API routes, `apps/worker` owns background loops, `packages/router` owns client hooks, `packages/lib` owns reusable logic, `packages/types` owns shared types, and `packages/prisma` owns database schema [apps/web/pages/_app.tsx:1-118](../../apps/web/pages/_app.tsx#L1), [packages/router/links.tsx:1-24](../../packages/router/links.tsx#L1), [packages/lib/schemaValidation.ts:125-148](../../packages/lib/schemaValidation.ts#L125), [packages/types/global.ts:14-35](../../packages/types/global.ts#L14), [packages/prisma/schema.prisma:1-8](../../packages/prisma/schema.prisma#L1).

**Find It In The Code:** Open `package.json:6-9`, `apps/worker/worker.ts:1-20`, `packages/router/links.tsx:1-24`.

```ts
import { useInfiniteQuery, useQueryClient, useMutation, useQuery } from "@tanstack/react-query";
// packages/router is frontend data access, not backend routing.
import { LinkRequestQuery, MobileAuth } from "@linkwarden/types/global";
// Shared types cross web and mobile boundaries.
import { PostLinkSchemaType } from "@linkwarden/lib/schemaValidation";
// Runtime validation contracts are reused by hooks/components/controllers.
```

**The Aha Moment:** Folder names tell you ownership; imports tell you dependencies.

**Socratic Checkpoint:**  
1. Which folder owns API routes?  
2. Which package owns Prisma schema?  
3. Why is `packages/router` a confusing name?  
4. Which package should hold a shared enum?  
5. Which app owns preservation jobs?

How to self-grade: strong answers cite web pages/API, Prisma schema, router hook imports, shared type imports, and worker startup [apps/web/pages/api/v1/links/index.ts:1-9](../../apps/web/pages/api/v1/links/index.ts#L1), [packages/prisma/schema.prisma:1-8](../../packages/prisma/schema.prisma#L1), [packages/router/links.tsx:1-24](../../packages/router/links.tsx#L1), [apps/worker/worker.ts:15-19](../../apps/worker/worker.ts#L15).

**Connects To:** Mission 3, because shared folders become meaningful when you see TypeScript contracts.

### Mission 3: TypeScript Is a Contract
**Tier:** Junior  
**Time Estimate:** 35 minutes  
**Goal:** Understand how Zod schemas and shared types protect data flow.

**The Concept:** Types are the catalog cards for the archive. Zod checks what arrives at the desk; TypeScript tells the UI and API what fields a saved item should have [packages/lib/schemaValidation.ts:125-148](../../packages/lib/schemaValidation.ts#L125), [packages/types/global.ts:14-35](../../packages/types/global.ts#L14).

**Design Intent Before You Read the Code:** `PostLinkSchema` validates incoming link data at runtime; `PostLinkSchemaType` lets React state and controller parameters share the same shape; `LinkIncludingShortenedCollectionAndTags` represents a link after it comes back with tags and collection [packages/lib/schemaValidation.ts:125-148](../../packages/lib/schemaValidation.ts#L125), [apps/web/components/ModalContent/NewLinkModal.tsx:43-44](../../apps/web/components/ModalContent/NewLinkModal.tsx#L43), [apps/web/lib/api/controllers/links/postLink.ts:12-16](../../apps/web/lib/api/controllers/links/postLink.ts#L12).

**Find It In The Code:** Open `packages/lib/schemaValidation.ts:125-148`, `packages/types/global.ts:14-35`.

```ts
export const PostLinkSchema = z.object({
  type: z.enum(["url", "pdf", "image"]).nullish(),
  // Link creation accepts a small, explicit set of type values.
  url: z.string().trim().max(2048).url().optional(),
  // Bad URLs fail before backend logic tries to preserve them.
  collection: z.object({
    id: z.number().optional(),
    name: z.string().trim().max(2048).optional(),
  }).optional(),
  // A link can target an existing collection or name a collection.
  tags: z.array(z.object({
    id: z.number().optional(),
    name: z.string().trim().max(50),
  })).optional() || [],
  // Tags are reusable labels with a short validated name.
});
```

**The Aha Moment:** A good type is not decoration; it is a promise kept by both UI and backend.

**Socratic Checkpoint:**  
1. Which fields can a new link submit?  
2. Why is `url` optional?  
3. What is inferred by `PostLinkSchemaType`?  
4. Why does the frontend still need server validation?  
5. Which type includes tags and collection with a link?

How to self-grade: strong answers cite `PostLinkSchema`, form state, controller parameters, and `LinkIncludingShortenedCollectionAndTags` [packages/lib/schemaValidation.ts:125-148](../../packages/lib/schemaValidation.ts#L125), [apps/web/components/ModalContent/NewLinkModal.tsx:43-44](../../apps/web/components/ModalContent/NewLinkModal.tsx#L43), [apps/web/lib/api/controllers/links/postLink.ts:12-16](../../apps/web/lib/api/controllers/links/postLink.ts#L12), [packages/types/global.ts:14-35](../../packages/types/global.ts#L14).

**Connects To:** Mission 4, because component props and state are contracts too.

### Mission 4: Your First React Component
**Tier:** Junior  
**Time Estimate:** 25 minutes  
**Goal:** Read a small presentational component confidently.

**The Concept:** A small component is a reusable label printer: it receives data, formats it, and avoids owning business rules. `DashboardItem` receives a metric name, value, and icon [apps/web/components/DashboardItem.tsx:1-23](../../apps/web/components/DashboardItem.tsx#L1).

**Design Intent Before You Read the Code:** `DashboardItem` should not fetch data, mutate state, or decide which metric matters. The dashboard page computes metrics and passes props when rendering section type `STATS` [apps/web/pages/dashboard.tsx:242-270](../../apps/web/pages/dashboard.tsx#L242).

**Find It In The Code:** Open `apps/web/components/DashboardItem.tsx:1-23`, then `apps/web/pages/dashboard.tsx:242-270`.

```tsx
export default function dashboardItem({ name, value, icon }: Props) {
  // Props are the whole contract: display label, number, and icon class.
  return (
    <div className="flex items-center justify-between w-full rounded-xl border border-neutral-content p-3 bg-gradient-to-tr from-neutral-content/70 to-50% to-base-200">
      <i className={`${icon} text-primary text-3xl drop-shadow`}></i>
      {/* Caller decides the icon; this component only renders it. */}
      <p className="text-neutral text-xs tracking-wider text-right">{name}</p>
      <p className="font-thin text-4xl text-primary mt-0.5 text-right">
        {value || 0}
        {/* Prevents a blank metric when value is falsy. */}
      </p>
    </div>
  );
}
```

**The Aha Moment:** Small components are easy to trust when they have no hidden data dependencies.

**Socratic Checkpoint:**  
1. What props does this component need?  
2. What does it refuse to own?  
3. Where are the stat values chosen?  
4. Why is `{value || 0}` used?  
5. What would make this component harder to reuse?

How to self-grade: strong answers cite props and dashboard `STATS` rendering [apps/web/components/DashboardItem.tsx:1-23](../../apps/web/components/DashboardItem.tsx#L1), [apps/web/pages/dashboard.tsx:242-270](../../apps/web/pages/dashboard.tsx#L242).

**Connects To:** Mission 6, because props are typed contracts.

### Mission 5: Your First Node.js Route
**Tier:** Junior  
**Time Estimate:** 35 minutes  
**Goal:** Read a Next.js API route from request to controller.

**The Concept:** An API route is the intake counter: verify identity, understand the request method, pass the work to the correct specialist, and return a response [apps/web/pages/api/v1/links/index.ts:9-71](../../apps/web/pages/api/v1/links/index.ts#L9).

**Design Intent Before You Read the Code:** The route should not contain all business logic. It authenticates, converts query strings into typed values, guards demo writes, and delegates to controllers [apps/web/pages/api/v1/links/index.ts:13-71](../../apps/web/pages/api/v1/links/index.ts#L13).

**Find It In The Code:** Open `apps/web/pages/api/v1/links/index.ts:9-71`.

```ts
const user = await verifyUser({ req, res });
if (!user) return;
// Every route branch below requires a valid user.

if (req.method === "GET") {
  const convertedData: LinkRequestQuery = {
    sort: Number(req.query.sort as string),
    cursor: req.query.cursor ? Number(req.query.cursor as string) : undefined,
  };
  // Query params arrive as strings; controllers receive typed query values.
  const links = await getLinks(user.id, convertedData);
  return res.status(links.status).json({ response: links.response });
} else if (req.method === "POST") {
  if (process.env.NEXT_PUBLIC_DEMO === "true")
    return res.status(400).json({
      response:
        "This action is disabled because this is a read-only demo of Linkwarden.",
    });
  // Demo mode blocks writes before database logic runs.
  const newlink = await postLink(req.body, user.id);
  return res.status(newlink.status).json({ response: newlink.response });
}
```

**The Aha Moment:** Routes coordinate; controllers decide.

**Socratic Checkpoint:**  
1. What happens before method branching?  
2. Why are query params converted?  
3. Which controller creates links?  
4. Where is demo mode checked?  
5. What response shape does this route use?

How to self-grade: strong answers cite `verifyUser`, `convertedData`, `postLink`, demo guard, and `{ response }` JSON [apps/web/pages/api/v1/links/index.ts:9-42](../../apps/web/pages/api/v1/links/index.ts#L9).

**Connects To:** Mission 12, because API contracts are route plus controller plus client hook.

### Mission 6: Props Are a Typed Contract
**Tier:** Junior  
**Time Estimate:** 30 minutes  
**Goal:** See how a component contract controls what a caller must provide.

**The Concept:** Props are like the fields on a bookmark intake slip. If a link card renderer needs `isSelected`, `toggleSelected`, `collection`, and `editMode`, the caller must prepare those fields before rendering [apps/web/components/LinkViews/Links.tsx:22-48](../../apps/web/components/LinkViews/Links.tsx#L22).

**Design Intent Before You Read the Code:** `Links` builds shared context once, then passes props into `CardView`, `MasonryView`, or `ListView`; the child views render link items without fetching collections themselves [apps/web/components/LinkViews/Links.tsx:389-484](../../apps/web/components/LinkViews/Links.tsx#L389).

**Find It In The Code:** Open `apps/web/components/LinkViews/Links.tsx:22-48`, `358-484`.

```tsx
function CardView({
  links,
  collectionsById,
  isPublicRoute,
  t,
  user,
  disableDraggable,
  isSelected,
  toggleSelected,
  editMode,
}: {
  links: LinkIncludingShortenedCollectionAndTags[];
  // The view receives already-fetched link records.
  collectionsById: Map<number, CollectionIncludingMembersAndLinkCount>;
  // The parent precomputes collection lookup once.
  isSelected: (id: number) => boolean;
  toggleSelected: (id: number) => void;
  // Selection behavior is injected from Zustand-backed parent logic.
}) {
  const settings = useLocalSettingsStore((state) => state.settings);
  // The real component begins by reading display settings from local UI state.
}
```

**The Aha Moment:** A prop list reveals what a component depends on.

**Socratic Checkpoint:**  
1. Which props are data?  
2. Which props are behavior?  
3. Why pass `collectionsById` instead of raw collections?  
4. Which props come from Zustand?  
5. Which prop changes public/private behavior?

How to self-grade: strong answers cite `CardView` props, collection map, `useLinkStore`, and public route calculation [apps/web/components/LinkViews/Links.tsx:22-48](../../apps/web/components/LinkViews/Links.tsx#L22), [apps/web/components/LinkViews/Links.tsx:374-397](../../apps/web/components/LinkViews/Links.tsx#L374).

**Connects To:** Mission 9, because state has to come from somewhere.

### Mission 7: Following Data Into the App
**Tier:** Junior  
**Time Estimate:** 40 minutes  
**Goal:** Trace how dashboard data enters a page.

**The Concept:** Data enters the UI through named hooks, like archive clerks bringing collections, dashboard sections, and user preferences to the desk [apps/web/pages/dashboard.tsx:30-45](../../apps/web/pages/dashboard.tsx#L30).

**Design Intent Before You Read the Code:** The page calls hooks, stores minimal UI state, then renders sections. Fetching is centralized in router hooks; dashboard data comes from `/api/v2/dashboard` [packages/router/dashboardData.tsx:16-34](../../packages/router/dashboardData.tsx#L16).

**Find It In The Code:** Open `apps/web/pages/dashboard.tsx:30-69`, `packages/router/dashboardData.tsx:16-34`.

```tsx
const { data: collections = [] } = useCollections();
// Collection list is shared server state.

const {
  data: { links = [], numberOfPinnedLinks, numberOfTags = 0, collectionLinks = {} } = { links: [] },
  ...dashboardData
} = useDashboardData();
// Dashboard endpoint returns multiple section-ready datasets at once.

const { data: user } = useUser();
// User record includes dashboardSections used to decide what to render.
```

**The Aha Moment:** A page is often less about fetching itself and more about composing fetched results.

**Socratic Checkpoint:**  
1. Which hook fetches collections?  
2. Which hook fetches dashboard data?  
3. What defaults prevent render crashes?  
4. Where is `numberOfLinks` calculated?  
5. Which endpoint backs `useDashboardData`?

How to self-grade: strong answers cite hooks/defaults in dashboard, count calculation, and `/api/v2/dashboard` fetch [apps/web/pages/dashboard.tsx:30-69](../../apps/web/pages/dashboard.tsx#L30), [packages/router/dashboardData.tsx:16-34](../../packages/router/dashboardData.tsx#L16).

**Connects To:** Mission 14, because this is one slice of a full-stack feature trace.

### Mission 8: Navigation Is the App's Skeleton
**Tier:** Junior  
**Time Estimate:** 35 minutes  
**Goal:** Explain how pages, layouts, and redirects shape navigation.

**The Concept:** Navigation is the shelving layout of the archive: public shelves are visible, private shelves require a badge, and dashboard pages share a common frame [apps/web/layouts/AuthRedirect.tsx:41-81](../../apps/web/layouts/AuthRedirect.tsx#L41), [apps/web/layouts/MainLayout.tsx:48-73](../../apps/web/layouts/MainLayout.tsx#L48).

**Design Intent Before You Read the Code:** `AuthRedirect` decides whether to render children or redirect; pages can provide `getLayout`; `dashboard.tsx` uses `MainLayout` [apps/web/layouts/AuthRedirect.tsx:15-91](../../apps/web/layouts/AuthRedirect.tsx#L15), [apps/web/pages/_app.tsx:36-39](../../apps/web/pages/_app.tsx#L36), [apps/web/pages/dashboard.tsx:207-213](../../apps/web/pages/dashboard.tsx#L207).

**Find It In The Code:** Open `apps/web/layouts/AuthRedirect.tsx:41-91`, `apps/web/pages/_app.tsx:36-39`, `apps/web/pages/dashboard.tsx:207-213`.

```tsx
const routes = [
  { path: "/login", isProtected: false },
  { path: "/dashboard", isProtected: true },
  { path: "/settings", isProtected: true },
  { path: "/collections", isProtected: true },
  { path: "/links", isProtected: true },
];
// Protection is explicit path-prefix matching.

if (isUnauthenticated && routes.some((e) => router.pathname.startsWith(e.path) && e.isProtected)) {
  redirectTo("/login");
  // Private archive shelf: no badge, go to login.
}

Page.getLayout = function getLayout(page) {
  return <MainLayout>{page}</MainLayout>;
  // Dashboard chooses the authenticated app frame.
};
```

**The Aha Moment:** Page files define routes; layouts define frames; redirects define who may see them.

**Socratic Checkpoint:**  
1. Which paths are protected?  
2. Which pages bypass auth redirects?  
3. How does a page choose `MainLayout`?  
4. Why is client route protection not enough for APIs?  
5. What would you update after adding `/reports`?

How to self-grade: strong answers cite route list, public-page bypass, `getLayout`, and server `verifyUser` [apps/web/layouts/AuthRedirect.tsx:41-81](../../apps/web/layouts/AuthRedirect.tsx#L41), [apps/web/pages/dashboard.tsx:207-213](../../apps/web/pages/dashboard.tsx#L207), [apps/web/lib/api/verifyUser.ts:14-72](../../apps/web/lib/api/verifyUser.ts#L14).

**Connects To:** Mission 13, because navigation protection connects to middleware/auth chains.
