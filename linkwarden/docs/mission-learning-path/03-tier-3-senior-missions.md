# Tier 3 Senior Missions

### Mission 18: Reverse-Engineer the Architecture Decisions
**Tier:** Senior  
**Time Estimate:** 50 minutes  
**Goal:** Infer why the system is shaped around Next.js API routes, shared hooks, Prisma, and a worker.

**The Concept:** Architecture is the set of decisions that made future work easier or harder. Linkwarden separates UI/API request handling from background preservation work, like separating the front archive desk from the preservation lab [apps/web/pages/api/v1/links/index.ts:9-71](../../apps/web/pages/api/v1/links/index.ts#L9), [apps/worker/worker.ts:11-20](../../apps/worker/worker.ts#L11).

**Design Intent Before You Read the Code:** Request paths need fast responses and auth; preservation/indexing/RSS jobs need repeatable background loops. Shared hooks exist so web/mobile can reuse API contracts [packages/router/links.tsx:1-24](../../packages/router/links.tsx#L1), [apps/worker/worker.ts:15-19](../../apps/worker/worker.ts#L15).

**Find It In The Code:** Open `apps/web/pages/api/v1/links/index.ts:9-42`, `apps/worker/worker.ts:11-20`, `packages/router/links.tsx:397-424`.

```ts
// API route returns after controller work.
const newlink = await postLink(req.body, user.id);
return res.status(newlink.status).json({ response: newlink.response });

// Worker starts long-running background systems separately.
startRSSPolling();
linkProcessing(workerIntervalInSeconds);
autoTagPreservedLinks(workerIntervalInSeconds);
startIndexing(workerIntervalInSeconds);
```

**The Aha Moment:** The architecture optimizes link saving for responsiveness and preservation for background reliability.

**Socratic Checkpoint:**  
1. Why is preservation outside the page request path?  
2. Why put API hooks in `packages/router`?  
3. What does Prisma centralize?  
4. What risk comes from scattered permission checks?  
5. What would you document before changing worker behavior?

How to self-grade: strong answers cite API response, worker jobs, shared hooks, Prisma schema, and permission helpers [apps/web/pages/api/v1/links/index.ts:39-42](../../apps/web/pages/api/v1/links/index.ts#L39), [apps/worker/worker.ts:11-20](../../apps/worker/worker.ts#L11), [packages/router/links.tsx:1-24](../../packages/router/links.tsx#L1), [packages/prisma/schema.prisma:28-308](../../packages/prisma/schema.prisma#L28), [apps/web/lib/api/getPermission.ts:9-38](../../apps/web/lib/api/getPermission.ts#L9).

**Connects To:** Mission 19, because architecture decisions create predictable bug shapes.

### Mission 19: Find the Bugs Before They Happen
**Tier:** Senior  
**Time Estimate:** 55 minutes  
**Goal:** Practice spotting likely regressions from code shape alone.

**The Concept:** Senior debugging starts before the bug report. You look for places where the archive's catalog and shelves can disagree: cache updates, sorting, permissions, and fallback paths [packages/router/links.tsx:426-548](../../packages/router/links.tsx#L426), [apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts:154-159](../../apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts#L154).

**Design Intent Before You Read the Code:** Scan for manual synchronization, duplicated logic, weak types, and branches that should be symmetrical [apps/web/lib/api/controllers/search/searchLinks.ts:54-255](../../apps/web/lib/api/controllers/search/searchLinks.ts#L54), [apps/web/pages/dashboard.tsx:215-227](../../apps/web/pages/dashboard.tsx#L215).

**Find It In The Code:** Open `getDashboardDataV2.ts:154-159`, `packages/router/links.tsx:543-546`, `searchLinks.ts:54-255`.

```ts
const merged = [...recentlyAddedLinks, ...pinnedLinks].sort(
  (a, b) => new Date(b.id).getTime() - new Date(a.id).getTime()
);
// Smell: id is numeric; Date parsing is not the domain concept.

queryClient.invalidateQueries({ queryKey: ["dashboardData"] });
queryClient.invalidateQueries({ queryKey: ["collections"] });
queryClient.invalidateQueries({ queryKey: ["tags"] });
queryClient.invalidateQueries({ queryKey: ["publicLinks"] });
// Manual invalidation list: every new link-adjacent query key must be remembered.
```

**The Aha Moment:** Most bugs live where code manually keeps two truths aligned.

**Socratic Checkpoint:**  
1. Which line looks like an ordering bug?  
2. Which invalidation list could become incomplete?  
3. Which search branches must stay behaviorally aligned?  
4. Which props hide risk with `any`?  
5. Which route pattern risks missing 405 responses?

How to self-grade: strong answers cite dashboard sort, invalidation list, Meili/Prisma branches, dashboard `any`, and route branching [apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts:154-159](../../apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts#L154), [packages/router/links.tsx:543-546](../../packages/router/links.tsx#L543), [apps/web/lib/api/controllers/search/searchLinks.ts:54-255](../../apps/web/lib/api/controllers/search/searchLinks.ts#L54), [apps/web/pages/dashboard.tsx:215-227](../../apps/web/pages/dashboard.tsx#L215), [apps/web/pages/api/v1/collections/index.ts:13-29](../../apps/web/pages/api/v1/collections/index.ts#L13).

**Connects To:** Mission 20, because bug prediction becomes bug-injection testing.

### Mission 20: The Bug Injection Challenge
**Tier:** Senior  
**Time Estimate:** 60 minutes  
**Goal:** Design tests from user-visible symptoms without editing production code.

**The Concept:** A bug injection challenge is a rehearsal: describe the broken shelf from the user's perspective, then write the smallest test that proves the catalog is wrong.

**Design Intent Before You Read the Code:** Use existing test style: Vitest mocks dependencies for API units, Playwright checks UI behavior through fixtures [apps/web/pages/api/v1/preserved/token.test.ts:1-24](../../apps/web/pages/api/v1/preserved/token.test.ts#L1), [apps/web/e2e/tests/public/login.spec.ts:1-50](../../apps/web/e2e/tests/public/login.spec.ts#L1).

**Find It In The Code:** Open `apps/web/pages/api/v1/preserved/token.test.ts:50-120`, `apps/web/e2e/tests/public/login.spec.ts:9-48`.

```ts
it("returns the archive resolution error when access is denied", async () => {
  vi.mocked(verifyToken).mockResolvedValue("You must be logged in.");
  vi.mocked(resolveAccessibleArchive).mockResolvedValue({
    status: 401,
    response: "You don't have access to this collection.",
  } as any);
  // Test controls dependencies and asserts the route response.
  await handler({ method: "GET", query: { linkId: "10" } } as any, res);
  expect(state.statusCode).toBe(401);
});
```

**The Aha Moment:** Good bug tests start with the symptom and only mock what blocks reaching it.

**Socratic Checkpoint:**  
1. What dependencies does the preserved token test mock?  
2. What user-visible error does it assert?  
3. How would you test the dashboard sort bug?  
4. How would you test missing 405 behavior?  
5. Which bug belongs in E2E instead of unit tests?

How to self-grade: strong answers cite mocked modules, status assertions, dashboard sort, route branch, and login E2E fixture pattern [apps/web/pages/api/v1/preserved/token.test.ts:1-24](../../apps/web/pages/api/v1/preserved/token.test.ts#L1), [apps/web/pages/api/v1/preserved/token.test.ts:99-120](../../apps/web/pages/api/v1/preserved/token.test.ts#L99), [apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts:154-159](../../apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts#L154), [apps/web/pages/api/v1/collections/index.ts:13-29](../../apps/web/pages/api/v1/collections/index.ts#L13), [apps/web/e2e/tests/public/login.spec.ts:38-48](../../apps/web/e2e/tests/public/login.spec.ts#L38).

**Connects To:** Mission 23, because the next step is writing the missing test.

### Mission 21: Performance X-Ray
**Tier:** Senior  
**Time Estimate:** 50 minutes  
**Goal:** Identify concrete performance costs and suggest safe fixes.

**The Concept:** Performance is making sure the archive clerk does not re-sort every shelf, refetch forever, or scan the same catalog repeatedly [apps/web/components/LinkViews/Links.tsx:405-425](../../apps/web/components/LinkViews/Links.tsx#L405).

**Design Intent Before You Read the Code:** Look for polling, repeated work inside render, unbounded lists, query stale times, and parallel DB calls [apps/web/pages/_app.tsx:18-24](../../apps/web/pages/_app.tsx#L18), [apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts:23-41](../../apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts#L23).

**Find It In The Code:** Open `Links.tsx:391-425`, `_app.tsx:18-24`, `getDashboardDataV2.ts:23-41`.

```tsx
const collectionsById = useMemo(() => {
  const m = new Map<number, (typeof collections)[number]>();
  for (const c of collections) m.set(c.id as any, c);
  return m;
}, [collections]);
// Good: avoids rebuilding lookup unless collections change.

if (links?.some((e) => !e.preview?.startsWith("archives") && e.preview !== "unavailable")) {
  interval = setInterval(async () => {
    useData.refetch().catch(console.error);
  }, 5000);
}
// Risk: polling continues for any pending preview and needs a clear stop condition.
```

**The Aha Moment:** Performance review is about bounding work, not guessing what is slow.

**Socratic Checkpoint:**  
1. What work is memoized in `Links`?  
2. What triggers 5-second polling?  
3. What global query stale time is configured?  
4. Which dashboard queries run in parallel?  
5. What would you measure before changing polling?

How to self-grade: strong answers cite `useMemo`, polling, query stale time, `Promise.all`, and preservation status fields [apps/web/components/LinkViews/Links.tsx:391-425](../../apps/web/components/LinkViews/Links.tsx#L391), [apps/web/pages/_app.tsx:18-24](../../apps/web/pages/_app.tsx#L18), [apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts:23-41](../../apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts#L23), [packages/prisma/schema.prisma:183-191](../../packages/prisma/schema.prisma#L183).

**Connects To:** Mission 18, because performance is shaped by architecture.

### Mission 22: The Security Audit
**Tier:** Senior  
**Time Estimate:** 60 minutes  
**Goal:** Audit authentication, authorization, SSRF protection, and method handling.

**The Concept:** Security is deciding who may enter the archive, which shelves they may touch, and which outside URLs the server may fetch [apps/web/lib/api/verifyUser.ts:18-72](../../apps/web/lib/api/verifyUser.ts#L18), [apps/web/lib/api/controllers/links/postLink.ts:28-30](../../apps/web/lib/api/controllers/links/postLink.ts#L28).

**Design Intent Before You Read the Code:** Check identity, token revocation, email/subscription gates, collection membership flags, demo write guards, SSRF checks, and explicit method handling [apps/web/lib/api/verifyToken.ts:23-35](../../apps/web/lib/api/verifyToken.ts#L23), [apps/web/lib/api/setCollection.ts:24-38](../../apps/web/lib/api/setCollection.ts#L24), [apps/web/pages/api/v1/links/index.ts:32-71](../../apps/web/pages/api/v1/links/index.ts#L32).

**Find It In The Code:** Open `verifyUser.ts:18-72`, `verifyToken.ts:12-35`, `setCollection.ts:24-38`, `postLink.ts:28-39`.

```ts
const revoked = await prisma.accessToken.findFirst({
  where: { token: token.jti, revoked: true },
});
if (revoked) {
  return "Your session has expired, please log in again.";
}
// Server can invalidate a token before JWT maxAge.

const collectionIsAccessible = await getPermission({ userId, collectionId });
const memberHasAccess = collectionIsAccessible?.members.some(
  (e) => e.userId === userId && e.canCreate
);
// Creating a link requires owner status or canCreate membership.
```

**The Aha Moment:** Authentication says who you are; authorization says what shelf you can modify.

**Socratic Checkpoint:**  
1. Where is token revocation checked?  
2. Where is email verification enforced?  
3. Where is collection `canCreate` enforced?  
4. Where is SSRF safety checked?  
5. Which route pattern should add 405s?

How to self-grade: strong answers cite token revocation, email verification, `setCollection`, `isUrlSafeForServerSideFetch`, and method branch gaps [apps/web/lib/api/verifyToken.ts:23-35](../../apps/web/lib/api/verifyToken.ts#L23), [apps/web/lib/api/verifyUser.ts:49-58](../../apps/web/lib/api/verifyUser.ts#L49), [apps/web/lib/api/setCollection.ts:24-38](../../apps/web/lib/api/setCollection.ts#L24), [apps/web/lib/api/controllers/links/postLink.ts:28-30](../../apps/web/lib/api/controllers/links/postLink.ts#L28), [apps/web/pages/api/v1/collections/index.ts:13-29](../../apps/web/pages/api/v1/collections/index.ts#L13).

**Connects To:** Mission 23, because high-risk security rules deserve tests.

### Mission 23: Write the Test That Doesn't Exist
**Tier:** Senior  
**Time Estimate:** 60 minutes  
**Goal:** Draft a focused test for a risky untested behavior.

**The Concept:** A missing test is a blind shelf in the archive. The highest-value tests protect rules that are easy to break and hard for users to explain, such as collection permissions [apps/web/lib/api/setCollection.ts:24-38](../../apps/web/lib/api/setCollection.ts#L24).

**Design Intent Before You Read the Code:** Test the smallest unit that owns the rule. For collection create permission during link creation, `setCollection` owns the decision and `postLink` consumes its result [apps/web/lib/api/setCollection.ts:11-102](../../apps/web/lib/api/setCollection.ts#L11), [apps/web/lib/api/controllers/links/postLink.ts:32-39](../../apps/web/lib/api/controllers/links/postLink.ts#L32).

**Find It In The Code:** Open `setCollection.ts:24-38`, `postLink.ts:32-39`, `preserved/token.test.ts:1-24`.

```ts
vi.mock("@linkwarden/prisma", () => ({ prisma: { collection: { findUnique: vi.fn() } } }));
vi.mock("./getPermission", () => ({ default: vi.fn() }));
// Follow existing test style: isolate dependencies with vi.mock.

it("rejects a member without canCreate", async () => {
  vi.mocked(getPermission).mockResolvedValue({
    ownerId: 1,
    members: [{ userId: 2, canCreate: false }],
  } as any);
  await expect(setCollection({ userId: 2, collectionId: 5 })).resolves.toBeNull();
});
// The assertion protects the permission rule directly.
```

**The Aha Moment:** Test the rule at the layer that owns it, then add broader tests only when integration risk remains.

**Socratic Checkpoint:**  
1. Which function owns the `canCreate` decision?  
2. Which controller relies on that decision?  
3. What mocks are needed?  
4. What assertion proves denial?  
5. What second test would prove owner access?

How to self-grade: strong answers cite `setCollection`, `postLink`, existing mock style, and the owner/member rules [apps/web/lib/api/setCollection.ts:24-38](../../apps/web/lib/api/setCollection.ts#L24), [apps/web/lib/api/controllers/links/postLink.ts:32-39](../../apps/web/lib/api/controllers/links/postLink.ts#L32), [apps/web/pages/api/v1/preserved/token.test.ts:1-24](../../apps/web/pages/api/v1/preserved/token.test.ts#L1).

**Connects To:** Mission 24, because tests and history tell future engineers why rules exist.

### Mission 24: The Git History Tells a Story
**Tier:** Senior  
**Time Estimate:** 45 minutes  
**Goal:** Use migrations and filenames to infer product evolution.

**The Concept:** Even without reading actual commit history, migrations are fossil layers. Linkwarden's current schema shows accumulated product areas including users, subscriptions, access tokens, preservation fields, dashboard sections, themes, AI tagging, RSS subscriptions, and metadata [packages/prisma/schema.prisma:28-75](../../packages/prisma/schema.prisma#L28), [packages/prisma/schema.prisma:166-198](../../packages/prisma/schema.prisma#L166), [packages/prisma/schema.prisma:200-258](../../packages/prisma/schema.prisma#L200), [packages/prisma/schema.prisma:275-308](../../packages/prisma/schema.prisma#L275).

**Design Intent Before You Read the Code:** Read migration names as product history, then verify current schema and controllers. A migration named `add_dashboard_sections` should point you to `DashboardSection`, dashboard APIs, and dashboard UI [packages/prisma/schema.prisma:275-294](../../packages/prisma/schema.prisma#L275), [apps/web/pages/api/v2/dashboard/index.ts:7-38](../../apps/web/pages/api/v2/dashboard/index.ts#L7), [apps/web/pages/dashboard.tsx:49-93](../../apps/web/pages/dashboard.tsx#L49).

**Find It In The Code:** Open `packages/prisma/schema.prisma:275-294`, `apps/web/pages/api/v2/dashboard/index.ts:7-38`, `apps/web/pages/dashboard.tsx:49-93`.

```prisma
model DashboardSection {
  userId       Int
  collectionId Int?
  type         DashboardSectionType
  order        Int
  @@unique([userId, collectionId])
  // A user can customize dashboard sections; collection section is unique per user/collection.
}

enum DashboardSectionType {
  STATS
  RECENT_LINKS
  PINNED_LINKS
  COLLECTION
  // The enum mirrors the dashboard switch cases in the UI.
}
```

**The Aha Moment:** Schema history explains why today's code has the seams it has.

**Socratic Checkpoint:**  
1. What product feature does `DashboardSection` represent?  
2. Which endpoint reads/updates it?  
3. Which UI code renders by section type?  
4. What does the unique constraint imply?  
5. What migration theme suggests AI tagging exists?

How to self-grade: strong answers cite dashboard schema, v2 endpoint, UI section switch, unique constraint, and AI fields [packages/prisma/schema.prisma:275-294](../../packages/prisma/schema.prisma#L275), [apps/web/pages/api/v2/dashboard/index.ts:7-38](../../apps/web/pages/api/v2/dashboard/index.ts#L7), [apps/web/pages/dashboard.tsx:242-428](../../apps/web/pages/dashboard.tsx#L242), [packages/prisma/schema.prisma:52-55](../../packages/prisma/schema.prisma#L52), [packages/prisma/schema.prisma:206-212](../../packages/prisma/schema.prisma#L206).

**Connects To:** Mission 25, because permanent workflow docs should preserve the story for future work.
