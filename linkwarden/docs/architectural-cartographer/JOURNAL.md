# Architectural Cartographer Journal

## First-Pass Mental Model

Linkwarden is organized as a Yarn workspace with app packages under `apps/*` and shared packages under `packages/*` [package.json:6-9](../../package.json#L6). The main user-facing web runtime is `apps/web`, the background preservation/indexing runtime is `apps/worker`, and shared boundaries live in `packages/router`, `packages/lib`, `packages/types`, and `packages/prisma` [package.json:10-26](../../package.json#L10), [apps/worker/worker.ts:1-20](../../apps/worker/worker.ts#L1).

The central system shape is:

```text
Next.js pages -> layout/auth wrapper -> React Query hook -> API route -> controller -> Prisma -> database
```

That shape is visible in the link feature: `NewLinkModal` validates a `PostLinkSchema` payload, `useAddLink` posts it to `/api/v1/links`, the API route calls `verifyUser`, and `postLink` validates again before creating a `Link` with tags and a collection [apps/web/components/ModalContent/NewLinkModal.tsx:88-100](../../apps/web/components/ModalContent/NewLinkModal.tsx#L88), [packages/router/links.tsx:397-424](../../packages/router/links.tsx#L397), [apps/web/pages/api/v1/links/index.ts:9-42](../../apps/web/pages/api/v1/links/index.ts#L9), [apps/web/lib/api/controllers/links/postLink.ts:16-36](../../apps/web/lib/api/controllers/links/postLink.ts#L16), [apps/web/lib/api/controllers/links/postLink.ts:104-150](../../apps/web/lib/api/controllers/links/postLink.ts#L104).

## Key Discoveries

- The project uses Yarn 4.12.0 and workspace scripts for web, worker, Prisma, formatting, tests, and coverage [package.json:3-26](../../package.json#L3).
- The web shell creates one React Query `QueryClient`, wraps the app in `SessionProvider`, points NextAuth at `/api/v1/auth`, and applies page-level layouts through `Component.getLayout` [apps/web/pages/_app.tsx:18-24](../../apps/web/pages/_app.tsx#L18), [apps/web/pages/_app.tsx:48-114](../../apps/web/pages/_app.tsx#L48).
- Auth gating is client-side in `AuthRedirect`, while API requests are protected server-side through `verifyUser` and `verifyToken` [apps/web/layouts/AuthRedirect.tsx:23-91](../../apps/web/layouts/AuthRedirect.tsx#L23), [apps/web/lib/api/verifyUser.ts:14-72](../../apps/web/lib/api/verifyUser.ts#L14), [apps/web/lib/api/verifyToken.ts:9-35](../../apps/web/lib/api/verifyToken.ts#L9).
- Prisma defines the bookmark domain: `User`, `Collection`, membership permissions, `Link`, `Tag`, subscriptions, access tokens, RSS subscriptions, highlights, and dashboard sections [packages/prisma/schema.prisma:28-75](../../packages/prisma/schema.prisma#L28), [packages/prisma/schema.prisma:126-246](../../packages/prisma/schema.prisma#L126), [packages/prisma/schema.prisma:260-294](../../packages/prisma/schema.prisma#L260).
- React Query is the server-state layer; Zustand is used for local UI settings and selected link IDs [packages/router/links.tsx:72-103](../../packages/router/links.tsx#L72), [apps/web/store/localSettings.ts:28-123](../../apps/web/store/localSettings.ts#L28), [apps/web/store/links.ts:12-42](../../apps/web/store/links.ts#L12).
- Search uses Meilisearch when configured and falls back to Prisma queries when it is not configured [apps/web/lib/api/controllers/search/searchLinks.ts:54-153](../../apps/web/lib/api/controllers/search/searchLinks.ts#L54), [apps/web/lib/api/controllers/search/searchLinks.ts:155-255](../../apps/web/lib/api/controllers/search/searchLinks.ts#L155).
- The worker process restarts itself from `apps/worker/index.ts` and starts RSS polling, link processing, AI tagging, search indexing, and trial email work from `apps/worker/worker.ts` [apps/worker/index.ts:1-16](../../apps/worker/index.ts#L1), [apps/worker/worker.ts:11-20](../../apps/worker/worker.ts#L11).

## Why These Files Became Teaching Anchors

- `apps/web/pages/_app.tsx` shows global providers, metadata, layout plumbing, and the app-level dependency order [apps/web/pages/_app.tsx:48-114](../../apps/web/pages/_app.tsx#L48).
- `apps/web/layouts/AuthRedirect.tsx` is a small but consequential route gate, useful for learning app navigation and auth assumptions [apps/web/layouts/AuthRedirect.tsx:41-81](../../apps/web/layouts/AuthRedirect.tsx#L41).
- `packages/router/links.tsx` exposes React Query fetch/mutation patterns, optimistic updates, cache invalidation, mobile auth support, and typed link contracts [packages/router/links.tsx:26-60](../../packages/router/links.tsx#L26), [packages/router/links.tsx:394-548](../../packages/router/links.tsx#L394).
- `apps/web/lib/api/controllers/links/postLink.ts` is a compact full-stack backend lesson: validate, check SSRF safety, resolve collection permission, prevent duplicates, enforce limits, infer content type, write Prisma relations, and create archive folders [apps/web/lib/api/controllers/links/postLink.ts:16-166](../../apps/web/lib/api/controllers/links/postLink.ts#L16).
- `packages/prisma/schema.prisma` is the domain map; everything else is behavior around these models [packages/prisma/schema.prisma:28-75](../../packages/prisma/schema.prisma#L28), [packages/prisma/schema.prisma:126-246](../../packages/prisma/schema.prisma#L126).

## Where A Junior Might Get Confused

- `packages/router` is not server routing; it is a shared client hook layer that wraps `fetch` and React Query [packages/router/links.tsx:1-24](../../packages/router/links.tsx#L1), [packages/router/collections.tsx:76-105](../../packages/router/collections.tsx#L76).
- There are two auth layers: client redirect rules in `AuthRedirect`, and server request validation in `verifyUser`/`verifyToken` [apps/web/layouts/AuthRedirect.tsx:60-80](../../apps/web/layouts/AuthRedirect.tsx#L60), [apps/web/lib/api/verifyUser.ts:18-72](../../apps/web/lib/api/verifyUser.ts#L18).
- A collection is both a folder-like UI concept and a permission boundary; membership flags are explicit booleans in the join model [packages/prisma/schema.prisma:126-149](../../packages/prisma/schema.prisma#L126), [packages/prisma/schema.prisma:151-164](../../packages/prisma/schema.prisma#L151).

## Where A Mid-Level Engineer Should Slow Down

- Cache mutation code in `useAddLink` manually constructs optimistic links and updates multiple query families; mistakes here create UI lies that only appear after refetch or navigation [packages/router/links.tsx:426-548](../../packages/router/links.tsx#L426).
- Search has two paths with different pagination semantics: Meilisearch uses offset/limit, while the Prisma fallback uses cursor/skip [apps/web/lib/api/controllers/search/searchLinks.ts:64-82](../../apps/web/lib/api/controllers/search/searchLinks.ts#L64), [apps/web/lib/api/controllers/search/searchLinks.ts:192-250](../../apps/web/lib/api/controllers/search/searchLinks.ts#L192).
- Permission checks often need both owner and member paths; `setCollection`, `getPermission`, and search all repeat that model in different shapes [apps/web/lib/api/setCollection.ts:24-38](../../apps/web/lib/api/setCollection.ts#L24), [apps/web/lib/api/getPermission.ts:27-36](../../apps/web/lib/api/getPermission.ts#L27), [apps/web/lib/api/controllers/search/searchLinks.ts:197-213](../../apps/web/lib/api/controllers/search/searchLinks.ts#L197).

## Where A Senior Engineer Should Be Skeptical

- The dashboard merge sorts by `new Date(b.id).getTime()` even though `id` is an integer, which is a likely correctness smell in ordering logic [apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts:154-159](../../apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts#L154).
- `MainLayout` reads `localStorage` during render, which is fragile for server rendering or hydration if the layout is ever rendered where `window` is unavailable [apps/web/layouts/MainLayout.tsx:13-22](../../apps/web/layouts/MainLayout.tsx#L13).
- Client route protection is explicit path-prefix matching; adding a new protected page means updating the route list manually [apps/web/layouts/AuthRedirect.tsx:41-58](../../apps/web/layouts/AuthRedirect.tsx#L41).
- API route methods often fall through without a 405 response when the method is unsupported [apps/web/pages/api/v1/links/index.ts:13-71](../../apps/web/pages/api/v1/links/index.ts#L13), [apps/web/pages/api/v1/collections/index.ts:13-29](../../apps/web/pages/api/v1/collections/index.ts#L13).

## How To Use Checkpoints

Use every checkpoint as a proof drill. A strong answer names the behavior, cites the file and line range, explains why it exists in the app's bookmark-preservation domain, and predicts one failure mode. For example, when explaining link creation, mention schema validation on the client and server, the permission-resolving collection step, the Prisma `connectOrCreate` tag write, and the optimistic cache update [apps/web/components/ModalContent/NewLinkModal.tsx:88-100](../../apps/web/components/ModalContent/NewLinkModal.tsx#L88), [apps/web/lib/api/controllers/links/postLink.ts:16-36](../../apps/web/lib/api/controllers/links/postLink.ts#L16), [apps/web/lib/api/controllers/links/postLink.ts:120-150](../../apps/web/lib/api/controllers/links/postLink.ts#L120), [packages/router/links.tsx:507-548](../../packages/router/links.tsx#L507).

