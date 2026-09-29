# Senior Engineer Guide

## Architectural Critique

Scores are 1 low, 5 high.

- Scalability: 3/5. React Query pagination and worker separation help [packages/router/links.tsx:72-103](../../packages/router/links.tsx#L72), [apps/worker/worker.ts:15-19](../../apps/worker/worker.ts#L15), but duplicated permission filters and mixed Meili/Prisma search paths raise long-term consistency risk [apps/web/lib/api/controllers/search/searchLinks.ts:54-153](../../apps/web/lib/api/controllers/search/searchLinks.ts#L54), [apps/web/lib/api/controllers/search/searchLinks.ts:155-255](../../apps/web/lib/api/controllers/search/searchLinks.ts#L155).
- TypeScript discipline: 3/5. Shared Zod schemas and domain types are strong [packages/lib/schemaValidation.ts:125-179](../../packages/lib/schemaValidation.ts#L125), [packages/types/global.ts:14-35](../../packages/types/global.ts#L14), but `any` appears in important UI and cache areas [apps/web/pages/dashboard.tsx:218-225](../../apps/web/pages/dashboard.tsx#L218), [packages/router/links.tsx:162-190](../../packages/router/links.tsx#L162).
- Separation of concerns: 4/5. API handlers delegate to controllers and shared hooks delegate fetch/cache behavior [apps/web/pages/api/v1/links/index.ts:1-7](../../apps/web/pages/api/v1/links/index.ts#L1), [packages/router/links.tsx:394-548](../../packages/router/links.tsx#L394). Some components still hold layout, polling, selection, and view selection together [apps/web/components/LinkViews/Links.tsx:358-484](../../apps/web/components/LinkViews/Links.tsx#L358).
- Testability: 3/5. Vitest and Playwright exist [vitest.config.mts:4-12](../../vitest.config.mts#L4), [apps/web/playwright.config.ts:9-24](../../apps/web/playwright.config.ts#L9), but risky link creation/search/cache behavior has no obvious direct tests in the inspected test list.
- Maintainability: 3/5. File boundaries are understandable, but route protection lists, cache mutation helpers, and permission filters need careful review when changed [apps/web/layouts/AuthRedirect.tsx:41-58](../../apps/web/layouts/AuthRedirect.tsx#L41), [packages/router/links.tsx:426-548](../../packages/router/links.tsx#L426), [apps/web/lib/api/getPermission.ts:9-38](../../apps/web/lib/api/getPermission.ts#L9).
- Security posture: 4/5. API auth, revoked-token checks, subscription/email checks, SSRF safety, and collection permission checks exist [apps/web/lib/api/verifyUser.ts:18-72](../../apps/web/lib/api/verifyUser.ts#L18), [apps/web/lib/api/verifyToken.ts:23-35](../../apps/web/lib/api/verifyToken.ts#L23), [apps/web/lib/api/controllers/links/postLink.ts:28-36](../../apps/web/lib/api/controllers/links/postLink.ts#L28). Remaining concern: unsupported methods and scattered permissions make edge cases easier to miss [apps/web/pages/api/v1/collections/index.ts:13-29](../../apps/web/pages/api/v1/collections/index.ts#L13).
- Performance: 3/5. Pagination, query stale time, and dashboard parallel queries help [apps/web/pages/_app.tsx:18-24](../../apps/web/pages/_app.tsx#L18), [apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts:23-41](../../apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts#L23), but polling every 5 seconds for unfinished previews and repeated collection lookup per render need scrutiny [apps/web/components/LinkViews/Links.tsx:405-425](../../apps/web/components/LinkViews/Links.tsx#L405), [apps/web/components/LinkViews/Links.tsx:391-395](../../apps/web/components/LinkViews/Links.tsx#L391).

## Performance Audit

Finding 1: Dashboard merge sorts integer IDs through `new Date`, which is both semantically wrong and fragile [apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts:154-159](../../apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts#L154).

Corrected snippet:

```ts
const merged = [...recentlyAddedLinks, ...pinnedLinks].sort(
  (a, b) => b.id - a.id
  // IDs are numeric monotonic identifiers in Prisma, so compare them directly.
);
```

Finding 2: `Links` polls every 5 seconds whenever any link preview is still not archived/unavailable [apps/web/components/LinkViews/Links.tsx:405-425](../../apps/web/components/LinkViews/Links.tsx#L405). This is understandable for preservation, but should stop as soon as the active query is hidden or preservation status is explicit.

Corrected snippet:

```tsx
const hasPendingPreview = links?.some(
  (link) => !link.preview?.startsWith("archives") && link.preview !== "unavailable"
);

useEffect(() => {
  if (!hasPendingPreview || !useData?.refetch) return;
  const interval = setInterval(() => {
    useData.refetch().catch(console.error);
  }, 5000);
  return () => clearInterval(interval);
}, [hasPendingPreview, useData?.refetch]);
```

Finding 3: `MainLayout` reads `localStorage` during render [apps/web/layouts/MainLayout.tsx:13-22](../../apps/web/layouts/MainLayout.tsx#L13). If this layout ever renders in a non-browser context, it can fail.

Corrected snippet:

```tsx
const [showAnnouncement, setShowAnnouncement] = useState(true);
const [sidebarIsCollapsed, setSidebarIsCollapsed] = useState(false);

useEffect(() => {
  setShowAnnouncement(localStorage.getItem("showAnnouncementBar") !== "false");
  setSidebarIsCollapsed(localStorage.getItem("sidebarIsCollapsed") === "true");
}, []);
```

## Security Audit

Finding 1: API routes protect link/collection writes with `verifyUser`, which checks token, user existence, username, email verification, and subscription when Stripe is configured [apps/web/lib/api/verifyUser.ts:18-72](../../apps/web/lib/api/verifyUser.ts#L18). Keep that pattern mandatory for new non-public APIs.

Finding 2: Token revocation is checked against `AccessToken.revoked` by `jti` [apps/web/lib/api/verifyToken.ts:23-35](../../apps/web/lib/api/verifyToken.ts#L23), and the Prisma model has revoked/session/expiry fields [packages/prisma/schema.prisma:234-246](../../packages/prisma/schema.prisma#L234). New token-like auth should reuse that behavior.

Finding 3: Link creation correctly avoids preserving unsafe URLs via `isUrlSafeForServerSideFetch` [apps/web/lib/api/controllers/links/postLink.ts:28-30](../../apps/web/lib/api/controllers/links/postLink.ts#L28), but still stores unsafe URLs with preserved fields marked unavailable [apps/web/lib/api/controllers/links/postLink.ts:138-148](../../apps/web/lib/api/controllers/links/postLink.ts#L138). Review any future server-side fetch path for the same SSRF gate.

Corrected pattern for future fetchers:

```ts
if (!(await isUrlSafeForServerSideFetch(url))) {
  return { status: 400, response: "URL cannot be fetched by the server." };
}
// Only call fetchTitleAndHeaders, preservation handlers, or indexers after this gate.
```

Finding 4: Unsupported methods should explicitly return 405. Current examples branch on known methods without an `else` response [apps/web/pages/api/v1/links/index.ts:13-71](../../apps/web/pages/api/v1/links/index.ts#L13), [apps/web/pages/api/v1/collections/index.ts:13-29](../../apps/web/pages/api/v1/collections/index.ts#L13).

Corrected snippet:

```ts
res.setHeader("Allow", ["GET", "POST", "PUT", "DELETE"]);
return res.status(405).json({ response: "Method not allowed." });
```

## TypeScript Discipline Review

Keep: Zod schema inference for request bodies [packages/lib/schemaValidation.ts:125-148](../../packages/lib/schemaValidation.ts#L125), Prisma-derived domain types [packages/types/global.ts:14-35](../../packages/types/global.ts#L14), enum-driven view/sort options [packages/types/global.ts:81-112](../../packages/types/global.ts#L81).

Tighten: replace dashboard `any` props with derived response types [apps/web/pages/dashboard.tsx:215-227](../../apps/web/pages/dashboard.tsx#L215), replace cache `any` helper signatures with typed infinite query payloads [packages/router/links.tsx:162-222](../../packages/router/links.tsx#L162), and remove `as any` collection lookups where possible [apps/web/components/LinkViews/Links.tsx:391-395](../../apps/web/components/LinkViews/Links.tsx#L391).

## Custom Abstractions Inventory

- API client hooks: `packages/router/*`, especially `links`, `collections`, `dashboardData`, `user` [packages/router/links.tsx:26-60](../../packages/router/links.tsx#L26), [packages/router/dashboardData.tsx:6-35](../../packages/router/dashboardData.tsx#L6).
- Auth helpers: `verifyToken`, `verifyUser`, NextAuth config [apps/web/lib/api/verifyToken.ts:9-35](../../apps/web/lib/api/verifyToken.ts#L9), [apps/web/lib/api/verifyUser.ts:14-72](../../apps/web/lib/api/verifyUser.ts#L14), [apps/web/pages/api/v1/auth/[...nextauth].ts:1313-1512](../../apps/web/pages/api/v1/auth/%5B...nextauth%5D.ts#L1313).
- Permission helpers: `getPermission`, `setCollection`, collection root/member helper [apps/web/lib/api/getPermission.ts:9-38](../../apps/web/lib/api/getPermission.ts#L9), [apps/web/lib/api/setCollection.ts:11-102](../../apps/web/lib/api/setCollection.ts#L11).
- State stores: `localSettings`, `links` [apps/web/store/localSettings.ts:28-123](../../apps/web/store/localSettings.ts#L28), [apps/web/store/links.ts:12-42](../../apps/web/store/links.ts#L12).
- Preservation URL/archive helpers: tested preserved token/view paths and resolve helpers [apps/web/pages/api/v1/preserved/token.test.ts:50-120](../../apps/web/pages/api/v1/preserved/token.test.ts#L50).

## Testing Assessment And A Complete Test To Add

Existing test tooling: root `vitest` excludes e2e files [vitest.config.mts:4-12](../../vitest.config.mts#L4), Playwright config points at `apps/web/e2e` and base URL `http://127.0.0.1:3000` [apps/web/playwright.config.ts:9-24](../../apps/web/playwright.config.ts#L9), and preserved API tests mock dependencies directly [apps/web/pages/api/v1/preserved/token.test.ts:1-24](../../apps/web/pages/api/v1/preserved/token.test.ts#L1).

Risky untested behavior to add: `setCollection` should reject link creation into a collection where the user is a member but lacks `canCreate`. This is business-critical because `postLink` trusts `setCollection` before creating a link [apps/web/lib/api/controllers/links/postLink.ts:32-39](../../apps/web/lib/api/controllers/links/postLink.ts#L32), and `setCollection` checks `canCreate` [apps/web/lib/api/setCollection.ts:24-38](../../apps/web/lib/api/setCollection.ts#L24).

Runnable test to add:

```ts
import { describe, expect, it, vi } from "vitest";
import setCollection from "./setCollection";
import getPermission from "./getPermission";

vi.mock("@linkwarden/prisma", () => ({
  prisma: {
    collection: {
      findUnique: vi.fn().mockResolvedValue({ id: 5, ownerId: 1 }),
    },
  },
}));

vi.mock("./getPermission", () => ({ default: vi.fn() }));

describe("setCollection", () => {
  it("rejects a member without canCreate when choosing an existing collection", async () => {
    vi.mocked(getPermission).mockResolvedValue({
      id: 5,
      ownerId: 1,
      members: [{ userId: 2, canCreate: false }],
    } as any);

    await expect(
      setCollection({ userId: 2, collectionId: 5 })
    ).resolves.toBeNull();
  });
});
```

## Bug Injection Exercise

Do not modify production code. For each scenario, write or run a test first.

1. Symptom: dashboard recent links appear in a strange order after pinning. Test scenario: create links with IDs 1, 2, 10 and assert descending numeric order. Anchor: dashboard merge sort [apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts:154-159](../../apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts#L154).
2. Symptom: a newly added protected page briefly shows logged-out content. Test scenario: add route under `/reports`, leave it out of `AuthRedirect.routes`, visit unauthenticated. Anchor: route list [apps/web/layouts/AuthRedirect.tsx:41-58](../../apps/web/layouts/AuthRedirect.tsx#L41).
3. Symptom: link appears immediately, then disappears after refresh. Test scenario: optimistic `useAddLink` adds temporary link, server rejects duplicate or inaccessible collection. Anchors: optimistic mutation and server collection check [packages/router/links.tsx:487-548](../../packages/router/links.tsx#L487), [apps/web/lib/api/controllers/links/postLink.ts:32-39](../../apps/web/lib/api/controllers/links/postLink.ts#L32).
4. Symptom: public search returns items from a private collection. Test scenario: call public search controller path and assert `isPublic` filter is applied. Anchor: public collection condition [apps/web/lib/api/controllers/search/searchLinks.ts:41-49](../../apps/web/lib/api/controllers/search/searchLinks.ts#L41).
5. Symptom: unsupported API method hangs or returns a confusing response. Test scenario: send `PATCH` to `/api/v1/collections`; expect 405 after fix. Anchor: missing final method branch [apps/web/pages/api/v1/collections/index.ts:13-29](../../apps/web/pages/api/v1/collections/index.ts#L13).

## Git History Learning Exercise

Use these realistic commit messages as prompts to inspect migrations and code:

1. `Add dashboard sections table and v2 dashboard endpoint` implies schema and API evolution; inspect `DashboardSection` and `/api/v2/dashboard` [packages/prisma/schema.prisma:275-294](../../packages/prisma/schema.prisma#L275), [apps/web/pages/api/v2/dashboard/index.ts:7-38](../../apps/web/pages/api/v2/dashboard/index.ts#L7).
2. `Replace link list fetch with search endpoint` implies API behavior moved from `/links` to `/search`; inspect `useFetchLinks` and search route [packages/router/links.tsx:72-103](../../packages/router/links.tsx#L72), [apps/web/pages/api/v1/search/index.ts:6-35](../../apps/web/pages/api/v1/search/index.ts#L6).
3. `Harden server-side URL preservation` implies SSRF protection; inspect `postLink` and SSRF helpers [apps/web/lib/api/controllers/links/postLink.ts:28-30](../../apps/web/lib/api/controllers/links/postLink.ts#L28), [packages/lib/ssrf.ts:193-331](../../packages/lib/ssrf.ts#L193).
4. `Add shared collection member permissions` implies access checks spread through collection/link controllers [packages/prisma/schema.prisma:151-164](../../packages/prisma/schema.prisma#L151), [apps/web/lib/api/setCollection.ts:24-38](../../apps/web/lib/api/setCollection.ts#L24).
5. `Add optimistic link creation cache update` implies UI responsiveness and rollback complexity; inspect `useAddLink` [packages/router/links.tsx:426-548](../../packages/router/links.tsx#L426).

## If I Owned This Codebase

- Centralize permission predicates. Effort: M. Impact: High. Current owner/member checks repeat across helpers and search [apps/web/lib/api/getPermission.ts:27-36](../../apps/web/lib/api/getPermission.ts#L27), [apps/web/lib/api/controllers/search/searchLinks.ts:197-213](../../apps/web/lib/api/controllers/search/searchLinks.ts#L197).
- Add 405 responses to API routes. Effort: S. Impact: Medium. Link and collection handlers lack explicit unsupported-method responses [apps/web/pages/api/v1/links/index.ts:13-71](../../apps/web/pages/api/v1/links/index.ts#L13).
- Type React Query cache payload helpers. Effort: M. Impact: Medium. Current cache helpers accept `any` [packages/router/links.tsx:162-222](../../packages/router/links.tsx#L162).
- Fix dashboard ordering and add tests. Effort: S. Impact: High. Current sort treats IDs as dates [apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts:154-159](../../apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts#L154).
- Move localStorage initialization into effects or guarded functions. Effort: S. Impact: Medium. `MainLayout` and settings store assume browser access [apps/web/layouts/MainLayout.tsx:13-22](../../apps/web/layouts/MainLayout.tsx#L13), [apps/web/store/localSettings.ts:84-123](../../apps/web/store/localSettings.ts#L84).
- Add integration tests for link creation and search fallback. Effort: L. Impact: High. These paths cross validation, permissions, search, and Prisma [apps/web/lib/api/controllers/links/postLink.ts:16-166](../../apps/web/lib/api/controllers/links/postLink.ts#L16), [apps/web/lib/api/controllers/search/searchLinks.ts:54-255](../../apps/web/lib/api/controllers/search/searchLinks.ts#L54).
