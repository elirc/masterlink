# Junior Engineer Guide

## Setup And Run

This repo uses Yarn 4.12.0, workspaces, and root scripts for web, worker, Prisma, tests, coverage, and formatting [package.json:3-26](../../package.json#L3). A reconstructed local setup from repo evidence:

```powershell
# Uses Yarn because packageManager pins yarn@4.12.0.
# Evidence: package.json:3.
yarn install

# Create .env from the sample. Required basics are NEXTAUTH_URL, NEXTAUTH_SECRET,
# and DATABASE_URL for manual install; Docker uses POSTGRES_PASSWORD too.
# Evidence: .env.sample:3-12.
Copy-Item .env.sample .env

# Start dependencies with Docker if you want Postgres and Meilisearch locally.
# Evidence: docker-compose.yml:1-28.
docker compose up -d postgres meilisearch

# Generate Prisma client and run migrations.
# Evidence: package.json:19-22 and packages/prisma/package.json:10-14.
yarn prisma:generate
yarn prisma:dev

# Run the web app and worker together.
# Evidence: package.json:17-18.
yarn concurrently:dev
```

Environment evidence: `.env.sample` includes `NEXTAUTH_URL`, `NEXTAUTH_SECRET`, `DATABASE_URL`, and `POSTGRES_PASSWORD` [`.env.sample:3-12`](../../.env.sample#L3); optional preservation/search/AI/storage variables are also present [`.env.sample:13-49`](../../.env.sample#L13), [`.env.sample:50-90`](../../.env.sample#L50). Docker evidence: the compose file defines Postgres, the Linkwarden container, port `3000:3000`, a `/data/data` volume, and Meilisearch [docker-compose.yml:1-28](../../docker-compose.yml#L1). Seed evidence exists through Prisma package metadata and `packages/prisma/seed.js` [packages/prisma/package.json:15-17](../../packages/prisma/package.json#L15).

## Top-Level Folder Orientation

- `apps/web`: Next.js web app; pages, API routes, components, hooks, layouts, store, styles, e2e config [apps/web/pages/_app.tsx:1-118](../../apps/web/pages/_app.tsx#L1), [apps/web/playwright.config.ts:9-24](../../apps/web/playwright.config.ts#L9).
- `apps/worker`: long-running background process for migration, RSS, link processing, AI tagging, indexing, and trial emails [apps/worker/worker.ts:1-20](../../apps/worker/worker.ts#L1).
- `apps/mobile`: mobile app workspace is present; mobile auth/data stores exist separately from web through an Expo/Zustand auth store that persists instance/token data [apps/mobile/store/auth.ts:1-27](../../apps/mobile/store/auth.ts#L1), [apps/mobile/store/auth.ts:28-80](../../apps/mobile/store/auth.ts#L28).
- `packages/router`: shared React Query hooks that call API routes [packages/router/links.tsx:1-24](../../packages/router/links.tsx#L1), [packages/router/collections.tsx:1-8](../../packages/router/collections.tsx#L1).
- `packages/lib`: shared validation, SSRF safety, search helpers, mail transport, archive format helpers [packages/lib/schemaValidation.ts:125-148](../../packages/lib/schemaValidation.ts#L125), [packages/lib/ssrf.ts:193-331](../../packages/lib/ssrf.ts#L193).
- `packages/types`: shared TypeScript domain types and enums [packages/types/global.ts:14-35](../../packages/types/global.ts#L14), [packages/types/global.ts:81-112](../../packages/types/global.ts#L81).
- `packages/prisma`: Prisma schema, migrations, generated client wrapper, and seed metadata [packages/prisma/schema.prisma:1-8](../../packages/prisma/schema.prisma#L1), [packages/prisma/package.json:10-17](../../packages/prisma/package.json#L10).

## One Level Deeper

Important frontend folders:

- `apps/web/pages`: Next.js Pages Router pages and API handlers; route filenames map to URLs [apps/web/pages/dashboard.tsx:207-213](../../apps/web/pages/dashboard.tsx#L207), [apps/web/pages/api/v1/links/index.ts:9-71](../../apps/web/pages/api/v1/links/index.ts#L9).
- `apps/web/components`: reusable UI pieces and domain components such as link cards, modals, dashboard widgets, navigation, and preservation UI [apps/web/components/ModalContent/NewLinkModal.tsx:102-180](../../apps/web/components/ModalContent/NewLinkModal.tsx#L102), [apps/web/components/LinkViews/Links.tsx:358-484](../../apps/web/components/LinkViews/Links.tsx#L358).
- `apps/web/store` and `apps/web/hooks`: local browser state and custom hooks, including local settings and selected links [apps/web/store/localSettings.ts:28-123](../../apps/web/store/localSettings.ts#L28), [apps/web/store/links.ts:12-42](../../apps/web/store/links.ts#L12), [apps/web/hooks/useMediaQuery.tsx:3-20](../../apps/web/hooks/useMediaQuery.tsx#L3).

Important backend folders:

- `apps/web/pages/api`: HTTP entry points; they parse request methods and call controllers [apps/web/pages/api/v1/links/index.ts:13-71](../../apps/web/pages/api/v1/links/index.ts#L13), [apps/web/pages/api/v2/dashboard/index.ts:14-38](../../apps/web/pages/api/v2/dashboard/index.ts#L14).
- `apps/web/lib/api/controllers`: business logic; examples include link creation, search, dashboard data, collection creation [apps/web/lib/api/controllers/links/postLink.ts:12-166](../../apps/web/lib/api/controllers/links/postLink.ts#L12), [apps/web/lib/api/controllers/search/searchLinks.ts:16-255](../../apps/web/lib/api/controllers/search/searchLinks.ts#L16).
- `apps/web/lib/api`: cross-cutting API helpers for auth, permissions, collection resolution, preserved URLs, payment, and email [apps/web/lib/api/verifyUser.ts:14-72](../../apps/web/lib/api/verifyUser.ts#L14), [apps/web/lib/api/getPermission.ts:9-38](../../apps/web/lib/api/getPermission.ts#L9), [apps/web/lib/api/setCollection.ts:11-102](../../apps/web/lib/api/setCollection.ts#L11).

## Frontend Entry Point Walkthrough

Source: `_app.tsx` [apps/web/pages/_app.tsx:18-118](../../apps/web/pages/_app.tsx#L18).

```tsx
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 30, // Server data is considered fresh for 30 seconds.
    },
  },
});

function App({ Component, pageProps }: AppPropsWithLayout) {
  const getLayout = Component.getLayout ?? ((page) => page);
  // Pages can opt into layouts by assigning Page.getLayout.

  return (
    <QueryClientProvider client={queryClient}>
      {/* All @linkwarden/router hooks rely on this provider. */}
      <SessionProvider
        session={pageProps.session}
        refetchOnWindowFocus={false}
        basePath="/api/v1/auth"
        // NextAuth endpoints live under /api/v1/auth, not the default /api/auth.
      >
        <AuthRedirect>
          {/* Route access decisions happen before page content renders. */}
          {getLayout(<Component {...pageProps} />)}
          {/* Layout is page-specific; dashboard uses MainLayout. */}
        </AuthRedirect>
      </SessionProvider>
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

## Backend Entry Point Walkthrough

Source: `/api/v1/links` [apps/web/pages/api/v1/links/index.ts:9-71](../../apps/web/pages/api/v1/links/index.ts#L9).

```ts
export default async function links(req: NextApiRequest, res: NextApiResponse) {
  const user = await verifyUser({ req, res });
  if (!user) return;
  // Every method below assumes a real authenticated user.

  if (req.method === "GET") {
    const convertedData: LinkRequestQuery = {
      sort: Number(req.query.sort as string),
      cursor: req.query.cursor ? Number(req.query.cursor as string) : undefined,
      collectionId: req.query.collectionId
        ? Number(req.query.collectionId as string)
        : undefined,
      tagId: req.query.tagId ? Number(req.query.tagId as string) : undefined,
      pinnedOnly: req.query.pinnedOnly
        ? req.query.pinnedOnly === "true"
        : undefined,
      searchQueryString: req.query.searchQueryString
        ? (req.query.searchQueryString as string)
        : undefined,
    };
    // Query strings are strings; the controller expects typed numbers/booleans.

    const links = await getLinks(user.id, convertedData);
    return res.status(links.status).json({ response: links.response });
  } else if (req.method === "POST") {
    if (process.env.NEXT_PUBLIC_DEMO === "true")
      return res.status(400).json({
        response:
          "This action is disabled because this is a read-only demo of Linkwarden.",
      });
    // Demo mode is enforced before writes.

    const newlink = await postLink(req.body, user.id);
    return res.status(newlink.status).json({ response: newlink.response });
  }
}
```

## TypeScript Orientation

- `NextPageWithLayout` extends Next.js pages with optional `getLayout`, which dashboard uses to wrap content in `MainLayout` [apps/web/pages/_app.tsx:26-34](../../apps/web/pages/_app.tsx#L26), [apps/web/pages/dashboard.tsx:207-213](../../apps/web/pages/dashboard.tsx#L207).
- `PostLinkSchemaType` is inferred from Zod, so the form and controller share one runtime-and-compile-time contract [packages/lib/schemaValidation.ts:125-148](../../packages/lib/schemaValidation.ts#L125), [apps/web/components/ModalContent/NewLinkModal.tsx:43-44](../../apps/web/components/ModalContent/NewLinkModal.tsx#L43), [apps/web/lib/api/controllers/links/postLink.ts:12-16](../../apps/web/lib/api/controllers/links/postLink.ts#L12).
- `LinkIncludingShortenedCollectionAndTags` adapts Prisma `Link` for client usage by making IDs/dates optional/string-like and always including tags and collection [packages/types/global.ts:14-34](../../packages/types/global.ts#L14).
- `ViewMode`, `Sort`, `ArchivedFormat`, and `LinkRequestQuery` are cross-layer enums/types used by UI and APIs [packages/types/global.ts:81-112](../../packages/types/global.ts#L81), [packages/types/global.ts:160-173](../../packages/types/global.ts#L160).

## React Component Anatomy 1: Simple Dashboard Metric

Source: `DashboardItem` [apps/web/components/DashboardItem.tsx:1-23](../../apps/web/components/DashboardItem.tsx#L1).

```tsx
export default function dashboardItem({
  name,
  value,
  icon,
}: {
  name: string; // The label shown under/near the number.
  value: number; // The count to display; falls back to 0 in render.
  icon: string; // Bootstrap icon class, passed by the dashboard section.
}) {
  return (
    <div className="flex items-center justify-between w-full rounded-xl border border-neutral-content p-3 bg-gradient-to-tr from-neutral-content/70 to-50% to-base-200">
      <div className="w-14 aspect-square flex justify-center items-center bg-primary/20 rounded-xl select-none">
        <i className={`${icon} text-primary text-3xl drop-shadow`}></i>
        {/* This component delegates icon choice to its caller. */}
      </div>
      <div className="ml-4 flex flex-col justify-center">
        <p className="text-neutral text-xs tracking-wider text-right">{name}</p>
        <p className="font-thin text-4xl text-primary mt-0.5 text-right">
          {value || 0}
          {/* Defensive display: undefined/null/0 all render as 0. */}
        </p>
      </div>
    </div>
  );
}
```

## React Component Anatomy 2: Link Creation Modal

Source: `NewLinkModal` [apps/web/components/ModalContent/NewLinkModal.tsx:23-100](../../apps/web/components/ModalContent/NewLinkModal.tsx#L23).

```tsx
export default function NewLinkModal({ onClose }: Props) {
  const initial = {
    name: "",
    url: "",
    description: "",
    type: "url",
    tags: [],
    collection: { id: undefined, name: "" },
  } as PostLinkSchemaType;
  // The form state is shaped like the backend schema.

  const addLink = useAddLink({ toast, t });
  // This hook owns HTTP, optimistic update, rollback, and invalidation.

  const [link, setLink] = useState<PostLinkSchemaType>(initial);

  const submit = async () => {
    const dataValidation = PostLinkSchema.safeParse(link);
    // Client validates before sending, but the server validates again.

    if (!dataValidation.success)
      return toast.error(`Error: ${dataValidation.error.issues[0].message}`);

    addLink.mutateAsync(link);
    onClose();
    // The modal closes immediately because the mutation performs optimistic UI.
  };
}
```

## React Component Anatomy 3: Complex Link List Renderer

Source: `Links` [apps/web/components/LinkViews/Links.tsx:358-484](../../apps/web/components/LinkViews/Links.tsx#L358).

```tsx
export default function Links({ layout, links, editMode, useData }: Props) {
  const { ref, inView } = useInView();

  useEffect(() => {
    if (!inView) return; // Only paginate when the sentinel is visible.
    if (!useData.hasNextPage) return; // Stop at the last page.
    if (useData.isFetchingNextPage) return; // Avoid duplicate fetches.

    useData.fetchNextPage();
  }, [inView, useData.hasNextPage, useData.isFetchingNextPage, useData.fetchNextPage]);

  const collectionsById = useMemo(() => {
    const m = new Map<number, (typeof collections)[number]>();
    for (const c of collections) m.set(c.id as any, c);
    return m;
    // Build a lookup table so every link card can resolve its collection quickly.
  }, [collections]);

  useEffect(() => {
    if (!editMode) clearSelected();
    // Bulk-selection state should not leak after leaving edit mode.
  }, [editMode]);

  if (layout === ViewMode.List) {
    return (
      <ListView
        links={links || []}
        collectionsById={collectionsById}
        isPublicRoute={isPublicRoute}
        t={t}
        disableDraggable={disableDraggable}
        user={user}
        toggleSelected={toggleSelected}
        isSelected={isSelected}
        editMode={editMode || false}
        isLoading={useData?.isLoading}
        hasNextPage={useData?.hasNextPage}
        placeHolderRef={ref}
      />
    );
  }
  // The source continues with the same prop contract for MasonryView and CardView.
}
```

## Backend Route/Controller Anatomy 1: Collections Route

Source: `/api/v1/collections` [apps/web/pages/api/v1/collections/index.ts:6-30](../../apps/web/pages/api/v1/collections/index.ts#L6).

```ts
export default async function collections(req: NextApiRequest, res: NextApiResponse) {
  const user = await verifyUser({ req, res });
  if (!user) return;
  // No collection data leaves the API unless auth is valid.

  if (req.method === "GET") {
    const collections = await getCollections(user.id);
    return res.status(collections.status).json({ response: collections.response });
  } else if (req.method === "POST") {
    if (process.env.NEXT_PUBLIC_DEMO === "true")
      return res.status(400).json({
        response:
          "This action is disabled because this is a read-only demo of Linkwarden.",
      });
    const newCollection = await postCollection(req.body, user.id);
    return res.status(newCollection.status).json({ response: newCollection.response });
  }
}
```

## Backend Route/Controller Anatomy 2: Post Collection

Source: `postCollection` [apps/web/lib/api/controllers/collections/postCollection.ts:15-132](../../apps/web/lib/api/controllers/collections/postCollection.ts#L15).

```ts
const dataValidation = PostCollectionSchema.safeParse(body);
if (!dataValidation.success) {
  return {
    response: `Error: ${
      dataValidation.error.issues[0].message
    } [${dataValidation.error.issues[0].path.join(", ")}]`,
    status: 400,
  };
}
// Runtime schema protects the DB from malformed collection data.

if (collection.parentId) {
  const permissionCheck = await getPermission({ userId, collectionId: collection.parentId });
  // Subcollections require permission on the parent collection.

  const memberHasAccess = permissionCheck?.members.some(
    (e) => e.userId === userId && e.canCreate && e.canUpdate && e.canDelete
  );
  if (!memberHasAccess && permissionCheck?.ownerId !== userId) return { status: 403 };
}

const newCollection = await prisma.collection.create({
  data: {
    name: collection.name.trim(),
    owner: { connect: { id: rootOwnerId } },
    createdBy: { connect: { id: userId } },
    parent: collection.parentId ? { connect: { id: collection.parentId } } : undefined,
  },
});
// Prisma expresses ownership, creator, and parent relation in one write.
```

## Backend Route/Controller Anatomy 3: Post Link

Source: `postLink` [apps/web/lib/api/controllers/links/postLink.ts:16-166](../../apps/web/lib/api/controllers/links/postLink.ts#L16).

```ts
const dataValidation = PostLinkSchema.safeParse(body);
if (!dataValidation.success) {
  return {
    response: `Error: ${
      dataValidation.error.issues[0].message
    } [${dataValidation.error.issues[0].path.join(", ")}]`,
    status: 400,
  };
}

const shouldPreserveUrl = link.url
  ? await isUrlSafeForServerSideFetch(link.url)
  : false;
// The app treats saved URLs as future server-side fetch targets, so SSRF safety matters.

const linkCollection = await setCollection({ userId, collectionId, collectionName });
if (!linkCollection) return { response: "Collection is not accessible.", status: 400 };
// A link must land in a collection the user can create in.

const newLink = await prisma.link.create({
  data: {
    url: link.url?.trim() || null,
    name,
    createdBy: { connect: { id: userId } },
    collection: { connect: { id: linkCollection.id } },
    tags: {
      connectOrCreate: link.tags?.map((tag) => ({
        where: { name_ownerId: { name: tag.name.trim(), ownerId: linkCollection.ownerId } },
        create: { name: tag.name.trim(), owner: { connect: { id: linkCollection.ownerId } } },
      })),
    },
  },
  include: { tags: true, collection: true },
});
// Tags behave like reusable labels within an owner's archive.
```

## Domain Glossary

- Link: saved item with URL/file metadata and preserved-format fields such as `preview`, `image`, `pdf`, `readable`, and `monolith` [packages/prisma/schema.prisma:166-198](../../packages/prisma/schema.prisma#L166).
- Collection: folder-like permission boundary that owns links, can be nested, public, shared, and ordered [packages/prisma/schema.prisma:126-149](../../packages/prisma/schema.prisma#L126).
- Tag: user-owned label that can also carry archival/AI tagging settings [packages/prisma/schema.prisma:200-218](../../packages/prisma/schema.prisma#L200).
- Preserved format: archived representation such as PNG, JPEG, PDF, readability, or monolith [packages/types/global.ts:160-166](../../packages/types/global.ts#L160).
- Highlight: annotation range on a preserved/readable link, tied to a user and link [packages/prisma/schema.prisma:260-273](../../packages/prisma/schema.prisma#L260).
- Access token/session token: persisted token metadata with revoked/session flags and expiry [packages/prisma/schema.prisma:234-246](../../packages/prisma/schema.prisma#L234), [apps/web/lib/api/verifyToken.ts:23-35](../../apps/web/lib/api/verifyToken.ts#L23).
- Dashboard section: user-customizable dashboard block for stats, recent links, pinned links, or a specific collection [packages/prisma/schema.prisma:275-294](../../packages/prisma/schema.prisma#L275).

## Junior Socratic Checkpoint

1. Where does the app create React Query and NextAuth providers?
2. Why does link creation validate in both the modal and controller?
3. What database model makes a collection collaborative?
4. Which file decides whether an unauthenticated user can see `/dashboard`?
5. What is the difference between `packages/router` and `apps/web/pages/api`?
6. Where would you look first if tags are not appearing on new links?
7. Why can a link appear in the UI before the server response is fully processed?

How to self-grade: a strong answer cites `_app.tsx` for providers [apps/web/pages/_app.tsx:48-114](../../apps/web/pages/_app.tsx#L48), `PostLinkSchema` plus `NewLinkModal` and `postLink` for validation [packages/lib/schemaValidation.ts:125-148](../../packages/lib/schemaValidation.ts#L125), `UsersAndCollections` for collaboration [packages/prisma/schema.prisma:151-164](../../packages/prisma/schema.prisma#L151), `AuthRedirect` for route gating [apps/web/layouts/AuthRedirect.tsx:41-81](../../apps/web/layouts/AuthRedirect.tsx#L41), `packages/router/links.tsx` for client hooks [packages/router/links.tsx:394-548](../../packages/router/links.tsx#L394), and the Prisma `connectOrCreate` block for tag persistence [apps/web/lib/api/controllers/links/postLink.ts:120-137](../../apps/web/lib/api/controllers/links/postLink.ts#L120).
