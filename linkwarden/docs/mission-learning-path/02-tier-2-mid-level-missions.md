# Tier 2 Mid-Level Missions

### Mission 9: State Has a Home and a Reason
**Tier:** Mid-Level  
**Time Estimate:** 40 minutes  
**Goal:** Decide whether a value belongs in React Query, Zustand, local state, or localStorage.

**The Concept:** State is like archive inventory: database-backed shelves need a shared catalog, while temporary UI selections can live on a clipboard. React Query owns server state; Zustand owns local UI state [packages/router/links.tsx:72-103](../../packages/router/links.tsx#L72), [apps/web/store/links.ts:12-42](../../apps/web/store/links.ts#L12).

**Design Intent Before You Read the Code:** Server data should refetch, invalidate, and sync across components. Selection state should be fast, local, and disposable when edit mode ends [apps/web/components/LinkViews/Links.tsx:397-403](../../apps/web/components/LinkViews/Links.tsx#L397).

**Find It In The Code:** Open `apps/web/store/links.ts:12-42`, `apps/web/components/LinkViews/Links.tsx:397-403`.

```ts
const useLinkStore = create<LinkStore>()((set, get) => ({
  selectedIds: {},
  // Local clipboard of selected link IDs, not persisted server data.
  isSelected: (id) => !!get().selectedIds[id],
  toggleSelected: (id) => set((state) => {
    const next = { ...state.selectedIds };
    // Immutable copy keeps state updates predictable.
    if (next[id]) delete next[id];
    else next[id] = true;
    return { selectedIds: next, selectionCount: Object.keys(next).length };
  }),
}));
```

**The Aha Moment:** Server truth and UI convenience are different kinds of state.

**Socratic Checkpoint:**  
1. Why is selected link state not in the database?  
2. Why is link list data not just Zustand?  
3. Which line clears selection when leaving edit mode?  
4. What state is persisted to localStorage?  
5. What query key stores link pages?

How to self-grade: strong answers cite selected store, `clearSelected`, local settings persistence, and `["links", { params }]` [apps/web/store/links.ts:12-42](../../apps/web/store/links.ts#L12), [apps/web/components/LinkViews/Links.tsx:397-403](../../apps/web/components/LinkViews/Links.tsx#L397), [apps/web/store/localSettings.ts:46-123](../../apps/web/store/localSettings.ts#L46), [packages/router/links.tsx:72-103](../../packages/router/links.tsx#L72).

**Connects To:** Mission 10, because hooks are how state gets reused.

### Mission 10: The Custom Hook Ecosystem
**Tier:** Mid-Level  
**Time Estimate:** 45 minutes  
**Goal:** Trace custom hooks and explain their ownership.

**The Concept:** Hooks are specialized archive clerks. `useDashboardData` fetches dashboard shelf data; `useMediaQuery` listens to the browser; `useInitialData` hydrates local settings after session changes [packages/router/dashboardData.tsx:6-35](../../packages/router/dashboardData.tsx#L6), [apps/web/hooks/useMediaQuery.tsx:3-20](../../apps/web/hooks/useMediaQuery.tsx#L3), [apps/web/hooks/useInitialData.tsx:5-13](../../apps/web/hooks/useInitialData.tsx#L5).

**Design Intent Before You Read the Code:** A hook should hide a reusable interaction pattern. If it talks to the server, expect query keys. If it talks to browser APIs, expect `useEffect` and cleanup [packages/router/dashboardData.tsx:16-34](../../packages/router/dashboardData.tsx#L16), [apps/web/hooks/useMediaQuery.tsx:5-18](../../apps/web/hooks/useMediaQuery.tsx#L5).

**Find It In The Code:** Open `apps/web/hooks/useMediaQuery.tsx:3-20`, `apps/web/hooks/useInitialData.tsx:5-13`, `packages/router/dashboardData.tsx:16-34`.

```tsx
export default function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState<boolean>(false);
  useEffect(() => {
    const mediaQueryList = window.matchMedia(query);
    // Browser API is touched only after render.
    const handleChange = (event: MediaQueryListEvent) => setMatches(event.matches);
    setMatches(mediaQueryList.matches);
    mediaQueryList.addEventListener("change", handleChange);
    return () => mediaQueryList.removeEventListener("change", handleChange);
    // Cleanup prevents leaked listeners after components unmount.
  }, [query]);
  return matches;
}
```

**The Aha Moment:** A good hook packages one reusable relationship: server data, browser signal, or local setup.

**Socratic Checkpoint:**  
1. Which hooks are server-state hooks?  
2. Which hook listens to a browser API?  
3. Why does `useInitialData` depend on session status/data?  
4. What query key backs dashboard data?  
5. What cleanup does `useMediaQuery` perform?

How to self-grade: strong answers cite each hook's imports/effects/query keys [packages/router/dashboardData.tsx:1-35](../../packages/router/dashboardData.tsx#L1), [apps/web/hooks/useMediaQuery.tsx:3-20](../../apps/web/hooks/useMediaQuery.tsx#L3), [apps/web/hooks/useInitialData.tsx:5-13](../../apps/web/hooks/useInitialData.tsx#L5).

**Connects To:** Mission 11, because hooks often contain side effects.

### Mission 11: Side Effects Are Promises to the System
**Tier:** Mid-Level  
**Time Estimate:** 45 minutes  
**Goal:** Identify side effects and their cleanup/rollback obligations.

**The Concept:** A side effect is a promise to leave the archive in a coherent state: if you open a listener, close it; if you optimistically add a link, roll it back on failure [apps/web/hooks/useMediaQuery.tsx:13-17](../../apps/web/hooks/useMediaQuery.tsx#L13), [packages/router/links.tsx:521-533](../../packages/router/links.tsx#L521).

**Design Intent Before You Read the Code:** Side effects appear in React effects, fetch calls, localStorage writes, Prisma writes, cache writes, and worker loops. Each needs a matching guard, cleanup, rollback, or idempotent design [apps/web/store/localSettings.ts:46-83](../../apps/web/store/localSettings.ts#L46), [apps/worker/index.ts:3-16](../../apps/worker/index.ts#L3).

**Find It In The Code:** Open `packages/router/links.tsx:426-548`.

```ts
onMutate: async (link) => {
  await queryClient.cancelQueries({ queryKey: ["links"] });
  // Stop outgoing refetches before writing optimistic cache.
  const previousLinks = queryClient.getQueriesData({ queryKey: ["links"] });
  const previousDashboard = queryClient.getQueryData(["dashboardData"]);
  // Snapshot old state so failure can restore it.
  queryClient.setQueriesData({ queryKey: ["links"] }, (oldData) =>
    upsertLinkInInfiniteData(oldData, optimisticLink, tempId)
  );
  // Show the saved link immediately.
  return { previousLinks, previousDashboard, optimisticId: tempId };
},
onError: (_error, _variables, context) => {
  context.previousLinks?.forEach(([queryKey, data]) => {
    queryClient.setQueryData(queryKey, data);
  });
  queryClient.setQueryData(["dashboardData"], context.previousDashboard);
  // Rollback is the repayment of the optimistic promise.
}
```

**The Aha Moment:** Every optimistic effect should come with a rollback story.

**Socratic Checkpoint:**  
1. What queries are canceled before optimistic link creation?  
2. What previous state is saved?  
3. What cache is updated optimistically?  
4. What happens on error?  
5. What is invalidated on success?

How to self-grade: strong answers cite cancel/snapshot/write/error/success sections [packages/router/links.tsx:426-548](../../packages/router/links.tsx#L426).

**Connects To:** Mission 14, because optimistic UI is one phase of the full trace.

### Mission 12: The Full API Contract
**Tier:** Mid-Level  
**Time Estimate:** 50 minutes  
**Goal:** Map request payload, response shape, and client expectations.

**The Concept:** An API contract is the agreement between the intake desk and the catalog room: what the UI sends, what the API verifies, what the controller returns, and what the UI expects [packages/router/links.tsx:397-424](../../packages/router/links.tsx#L397), [apps/web/pages/api/v1/links/index.ts:39-42](../../apps/web/pages/api/v1/links/index.ts#L39).

**Design Intent Before You Read the Code:** The hook sends JSON and throws on non-OK; the route wraps controller output in `{ response }`; the controller returns `{ response, status }` [packages/router/links.tsx:406-424](../../packages/router/links.tsx#L406), [apps/web/pages/api/v1/links/index.ts:39-42](../../apps/web/pages/api/v1/links/index.ts#L39), [apps/web/lib/api/controllers/links/postLink.ts:166](../../apps/web/lib/api/controllers/links/postLink.ts#L166).

**Find It In The Code:** Open `packages/router/links.tsx:397-424`, `apps/web/pages/api/v1/links/index.ts:32-42`, `apps/web/lib/api/controllers/links/postLink.ts:12-25`.

```ts
const response = await fetch("/api/v1/links", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(link),
});
// Client sends exactly the link form object as JSON.

const data = await response.json();
if (!response.ok) throw new Error(data.response);
return data.response;
// Client expects errors and success payloads under response.
```

**The Aha Moment:** To change an API safely, update all four points: schema, hook, route, controller.

**Socratic Checkpoint:**  
1. What does the client send?  
2. What does the route return?  
3. What error shape does the hook expect?  
4. Where is server validation?  
5. What would break if the controller returned `{ data }` instead?

How to self-grade: strong answers cite hook response parsing, route JSON shape, controller status/response, and schema validation [packages/router/links.tsx:406-424](../../packages/router/links.tsx#L406), [apps/web/pages/api/v1/links/index.ts:39-42](../../apps/web/pages/api/v1/links/index.ts#L39), [apps/web/lib/api/controllers/links/postLink.ts:16-25](../../apps/web/lib/api/controllers/links/postLink.ts#L16).

**Connects To:** Mission 13, because auth/middleware sits before the contract is honored.

### Mission 13: The Middleware Chain
**Tier:** Mid-Level  
**Time Estimate:** 45 minutes  
**Goal:** Trace auth from route to token to user checks.

**The Concept:** The middleware chain is the archive badge check. `verifyUser` asks `verifyToken` for a valid badge, then checks the user record, email verification, and subscription status [apps/web/lib/api/verifyUser.ts:14-72](../../apps/web/lib/api/verifyUser.ts#L14).

**Design Intent Before You Read the Code:** API security should happen server-side even though client redirects exist. `AuthRedirect` is UX; `verifyUser` is enforcement [apps/web/layouts/AuthRedirect.tsx:23-91](../../apps/web/layouts/AuthRedirect.tsx#L23), [apps/web/lib/api/verifyUser.ts:18-72](../../apps/web/lib/api/verifyUser.ts#L18).

**Find It In The Code:** Open `apps/web/lib/api/verifyToken.ts:9-35`, `apps/web/lib/api/verifyUser.ts:18-72`.

```ts
const token = await getToken({ req });
const userId = token?.id;
if (!userId) return "You must be logged in.";
// No token ID means no authenticated user.

if (token.exp < Date.now() / 1000) {
  return "Your session has expired, please log in again.";
}
// Expired JWTs are rejected.

const revoked = await prisma.accessToken.findFirst({
  where: { token: token.jti, revoked: true },
});
if (revoked) {
  return "Your session has expired, please log in again.";
}
// Server-side revocation overrides the JWT.
```

**The Aha Moment:** Client redirects guide users; API auth protects data.

**Socratic Checkpoint:**  
1. What token fields are checked?  
2. How can a JWT be revoked?  
3. What user checks happen after token validation?  
4. What happens if email is not verified?  
5. Why is this needed even with `AuthRedirect`?

How to self-grade: strong answers cite token ID/expiry/revocation and user/email/subscription checks [apps/web/lib/api/verifyToken.ts:12-35](../../apps/web/lib/api/verifyToken.ts#L12), [apps/web/lib/api/verifyUser.ts:27-72](../../apps/web/lib/api/verifyUser.ts#L27).

**Connects To:** Mission 22, because security audit starts with these gates.

### Mission 14: End-to-End Feature Trace
**Tier:** Mid-Level  
**Time Estimate:** 60 minutes  
**Goal:** Trace creating a link from UI to database to UI update.

**The Concept:** A saved link is a parcel moving through Linkwarden's archive: label it in the modal, hand it to the hook, stamp identity at the API, file it in Prisma, then update the visible shelf [apps/web/components/ModalContent/NewLinkModal.tsx:88-100](../../apps/web/components/ModalContent/NewLinkModal.tsx#L88), [apps/web/lib/api/controllers/links/postLink.ts:104-166](../../apps/web/lib/api/controllers/links/postLink.ts#L104).

**Design Intent Before You Read the Code:** Each layer owns one job. UI owns form input, hook owns fetch/cache, route owns HTTP/auth, controller owns business rules, Prisma owns persistence, and React Query owns visible refresh [packages/router/links.tsx:397-548](../../packages/router/links.tsx#L397).

**Find It In The Code:** Open `NewLinkModal.tsx:88-100`, `packages/router/links.tsx:397-548`, `api/v1/links/index.ts:9-42`, `postLink.ts:16-166`.

```ts
// UI
const dataValidation = PostLinkSchema.safeParse(link);
addLink.mutateAsync(link);

// Hook
const response = await fetch("/api/v1/links", { method: "POST", body: JSON.stringify(link) });
queryClient.setQueriesData({ queryKey: ["links"] }, (oldData: any) =>
  upsertLinkInInfiniteData(oldData, optimisticLink, tempId)
);

// Route
const user = await verifyUser({ req, res });
const newlink = await postLink(req.body, user.id);

// Controller
const linkCollection = await setCollection({ userId, collectionId, collectionName });
const newLink = await prisma.link.create({ data: { collection: { connect: { id: linkCollection.id } } } });
```

**The Aha Moment:** Full-stack skill is knowing who owns each step, not memorizing every line.

**Socratic Checkpoint:**  
1. Where does validation happen?  
2. Where is the optimistic link created?  
3. Where is the user verified?  
4. Where is collection permission resolved?  
5. Where does the database write happen?

How to self-grade: strong answers cite each layer in chronological order [apps/web/components/ModalContent/NewLinkModal.tsx:88-100](../../apps/web/components/ModalContent/NewLinkModal.tsx#L88), [packages/router/links.tsx:397-548](../../packages/router/links.tsx#L397), [apps/web/pages/api/v1/links/index.ts:9-42](../../apps/web/pages/api/v1/links/index.ts#L9), [apps/web/lib/api/controllers/links/postLink.ts:16-166](../../apps/web/lib/api/controllers/links/postLink.ts#L16).

**Connects To:** Mission 15, because diff review means knowing what path a change touches.

### Mission 15: Read the Diff Like an Engineer
**Tier:** Mid-Level  
**Time Estimate:** 45 minutes  
**Goal:** Practice reviewing a realistic change across layers.

**The Concept:** A diff is not a pile of edits; it is a proposed change to a data flow. If a new link field is added, follow it through schema, hook, route, controller, model, and UI [packages/lib/schemaValidation.ts:125-148](../../packages/lib/schemaValidation.ts#L125), [packages/types/global.ts:14-35](../../packages/types/global.ts#L14).

**Design Intent Before You Read the Code:** Review by contract first, implementation second. The safest changes align Prisma fields, Zod schema, API payload, client type, and UI display [packages/prisma/schema.prisma:166-198](../../packages/prisma/schema.prisma#L166), [apps/web/lib/api/controllers/links/postLink.ts:104-150](../../apps/web/lib/api/controllers/links/postLink.ts#L104).

**Find It In The Code:** Open `schema.prisma:166-198`, `schemaValidation.ts:125-148`, `postLink.ts:104-150`.

```ts
// Prisma model says what can be stored.
model Link {
  name String @default("")
  description String @default("")
  metaDescription String?
}

// Zod schema says what can be accepted.
description: z.string().trim().max(2048).optional(),

// Controller says what actually gets written.
description: link.description,
```

**The Aha Moment:** A field is not real until it passes through every layer that should own it.

**Socratic Checkpoint:**  
1. Where would you add accepted input?  
2. Where would you persist it?  
3. Where would you display it?  
4. Where could cache become stale?  
5. What test would prove the flow works?

How to self-grade: strong answers cite Prisma, Zod, controller write, link components, and invalidation [packages/prisma/schema.prisma:166-198](../../packages/prisma/schema.prisma#L166), [packages/lib/schemaValidation.ts:125-148](../../packages/lib/schemaValidation.ts#L125), [apps/web/lib/api/controllers/links/postLink.ts:104-150](../../apps/web/lib/api/controllers/links/postLink.ts#L104), [packages/router/links.tsx:543-546](../../packages/router/links.tsx#L543).

**Connects To:** Mission 19, because good review predicts bugs.

### Mission 16: Composition Over Inheritance
**Tier:** Mid-Level  
**Time Estimate:** 35 minutes  
**Goal:** See how the UI composes views instead of subclassing them.

**The Concept:** Linkwarden builds shelves from smaller shelves: `Links` composes `CardView`, `MasonryView`, and `ListView`, and each view composes item components [apps/web/components/LinkViews/Links.tsx:22-150](../../apps/web/components/LinkViews/Links.tsx#L22), [apps/web/components/LinkViews/Links.tsx:294-355](../../apps/web/components/LinkViews/Links.tsx#L294).

**Design Intent Before You Read the Code:** Layout choice should be a runtime prop (`ViewMode`) that selects a component, not an inheritance hierarchy. Shared dependencies are prepared once in the parent [apps/web/components/LinkViews/Links.tsx:358-484](../../apps/web/components/LinkViews/Links.tsx#L358).

**Find It In The Code:** Open `apps/web/components/LinkViews/Links.tsx:431-484`.

```tsx
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
  // List view uses same data contract but a different renderer.
} else if (layout === ViewMode.Masonry) {
  return <MasonryView links={links || []} collectionsById={collectionsById} isPublicRoute={isPublicRoute} t={t} disableDraggable={disableDraggable} user={user} toggleSelected={toggleSelected} isSelected={isSelected} editMode={editMode || false} isLoading={useData?.isLoading} hasNextPage={useData?.hasNextPage} placeHolderRef={ref} />;
  // This single-line excerpt preserves the actual prop names used by the source.
} else {
  return <CardView links={links || []} collectionsById={collectionsById} isPublicRoute={isPublicRoute} t={t} user={user} disableDraggable={disableDraggable} toggleSelected={toggleSelected} isSelected={isSelected} editMode={editMode || false} isLoading={useData?.isLoading} hasNextPage={useData?.hasNextPage} placeHolderRef={ref} />;
  // Card view is the default and receives the same shared contract.
}
```

**The Aha Moment:** Composition lets one data flow support multiple presentations.

**Socratic Checkpoint:**  
1. Which enum chooses the view?  
2. What props are shared across all views?  
3. Why compute `collectionsById` in the parent?  
4. Where does list rendering differ from card rendering?  
5. What would inheritance add here that composition avoids?

How to self-grade: strong answers cite `ViewMode`, the conditional render, and shared props [packages/types/global.ts:81-85](../../packages/types/global.ts#L81), [apps/web/components/LinkViews/Links.tsx:431-484](../../apps/web/components/LinkViews/Links.tsx#L431).

**Connects To:** Mission 17, because TypeScript keeps composition contracts honest.

### Mission 17: TypeScript's Hidden Work
**Tier:** Mid-Level  
**Time Estimate:** 45 minutes  
**Goal:** Notice where TypeScript helps silently and where `any` weakens it.

**The Concept:** TypeScript is the quiet catalog auditor. It can prove link shapes, enum values, and props, but `any` is a handwritten note slipped past the auditor [packages/types/global.ts:14-35](../../packages/types/global.ts#L14), [apps/web/pages/dashboard.tsx:218-225](../../apps/web/pages/dashboard.tsx#L218).

**Design Intent Before You Read the Code:** Use generated/shared types at boundaries. Avoid `any` in areas where the shape matters for permissions, dashboard rendering, or cache updates [packages/router/links.tsx:162-222](../../packages/router/links.tsx#L162).

**Find It In The Code:** Open `packages/types/global.ts:14-35`, `apps/web/pages/dashboard.tsx:215-227`, `packages/router/links.tsx:162-222`.

```ts
export interface LinkIncludingShortenedCollectionAndTags
  extends Omit<
    Link,
    | "id"
    | "createdAt"
    | "collectionId"
    | "updatedAt"
    | "lastPreserved"
    | "importDate"
  > {
  id?: number;
  createdAt?: string;
  tags: Tag[];
  collection: OptionalExcluding<Collection, "name" | "ownerId">;
  // This tells UI code that tags and collection are present with each link.
}

type SectionProps = {
  collection: any;
  links: any[];
  dashboardData: any;
  // These anys hide what Section really needs and make regressions easier.
};
```

**The Aha Moment:** The most valuable types are the ones that protect cross-layer assumptions.

**Socratic Checkpoint:**  
1. What does `LinkIncludingShortenedCollectionAndTags` guarantee?  
2. Which dashboard props use `any`?  
3. Why is `any` risky in cache helpers?  
4. Which enums prevent typo-driven UI states?  
5. Where does Zod complement TypeScript?

How to self-grade: strong answers cite link type, dashboard `any`, cache helper `any`, `ViewMode`/`Sort`, and Zod schemas [packages/types/global.ts:14-35](../../packages/types/global.ts#L14), [apps/web/pages/dashboard.tsx:215-227](../../apps/web/pages/dashboard.tsx#L215), [packages/router/links.tsx:162-222](../../packages/router/links.tsx#L162), [packages/types/global.ts:81-112](../../packages/types/global.ts#L81), [packages/lib/schemaValidation.ts:125-148](../../packages/lib/schemaValidation.ts#L125).

**Connects To:** Mission 23, because better types make better tests easier.
