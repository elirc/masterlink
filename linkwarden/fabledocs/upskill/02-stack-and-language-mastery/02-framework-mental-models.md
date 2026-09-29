# Framework Mental Models (React 19 / React Query / Next.js pages router)

## React: render vs commit

Mental model: components are functions of (props, state) → UI description; React calls them (**render**, must be pure), diffs, then touches the DOM (**commit**), then runs effects. State updates schedule re-renders; each render captures its own props/state (closures — hence stale-closure bugs).

In this repo: pages-router app, so everything is a client component; state placement follows "server state in React Query, UI state local" (`NewLinkModal.tsx:43-44` local form state; zustand only for cross-cutting UI like bulk-selection `apps/web/store/links.ts`). `useLayoutEffect` used correctly for pre-paint focus (`NewLinkModal.tsx:84-86`).

Sharp edges here: conditional `useSession()` inside shared hooks (`links.tsx:65-70` — rules-of-hooks held by convention, see [08/02 Q5](../08-interview-prep/02-frontend-framework-questions.md)); memo keyed on `dataUpdatedAt` instead of data identity (`links.tsx:52-54`); temp-id swap remounting rows (key stability, [08/02 Q8](../08-interview-prep/02-frontend-framework-questions.md)).

## React Query: a cache with subscriptions

Mental model: a client-side database keyed by serialized `queryKey`; components subscribe; mutations either **invalidate** (refetch — blunt, safe) or **surgically edit** the cache (`setQueryData` — fast, fragile). This repo chose surgical editing throughout `packages/router/links.tsx:146-381` (a 200-line library of cache editors) plus full optimistic writes (`:426-534`). Study it as the *maximal* version of the approach; the maintenance cost (every cache × every mutation) is visible in the code size.

Infinite queries: data is `{pages: [{links, nextCursor}]}` — every helper must map over pages (`upsertLinkInInfiniteData:162-191`); flattening happens once in `useLinks` (`:52-54`).

`enabled` gating on auth status (`:102`) is the idiomatic way to sequence queries — no manual "if logged in fetch" effects anywhere. Correct.

## Next.js pages router

Mental model: `pages/*.tsx` = routes rendered client-side with optional per-page SSR (`getServerSideProps`); `pages/api/**` = serverless-style handlers — **the entire backend of this product** lives there. No RSC, no server actions. Data fetching is therefore uniform: React Query hitting its own API routes; SSR is used thinly (session/props bootstrap via `apps/web/lib/client/getServerSideProps.ts`).

Config sharp edges used here: per-route API config (`bodyParser: false`, `responseLimit` — `archives/[linkId].ts:17-22`); `NEXT_PUBLIC_*` build-time inlining (see [01/04 runtime map](../01-codebase-cartography/04-runtime-and-tooling-map.md)); i18n via next-i18next with per-user locale (`schema.prisma:37`).

## Data-fetching layer decision record (reconstructed)

REST + React Query, not tRPC/GraphQL — consistent with: a public API for third parties (API keys exist), a mobile client consuming the same endpoints with Bearer auth (`links.tsx:82-91`), and controllers testable without HTTP. Cost: hand-maintained types between server and client (`packages/types/global.ts`), the `any`-typed cache helpers. This exact tradeoff discussion is [08/02 Q11](../08-interview-prep/02-frontend-framework-questions.md) + [08/03 Q2](../08-interview-prep/03-api-and-data-modeling-questions.md) material.

## Pitfall checklist

☐ am I editing every cache this entity appears in? (list them first) ☐ optimistic rows: negative temp ids + rollback snapshot ☐ `enabled` for dependent queries, not effects ☐ keys: stable, serializable, shaped like the data's identity ☐ no server-state copies in useState.

## Drills

1. `grep -n "queryKey" packages/router/*.tsx` — draw the full key inventory and which mutations write each.
2. Break it on purpose (local): comment out the `onError` rollback in `useAddLink`, fail a request, and describe the ghost row's lifecycle.
3. Explain to a rubber duck why `useLinks` flattens pages in a memo instead of in the queryFn.

## Interview angle

[08/02](../08-interview-prep/02-frontend-framework-questions.md) Q1 (state taxonomy), Q2 (cache model), Q4 (optimistic updates), Q8 (re-render tracing), Q11 (router migration judgment). The one-sentence summary to carry: *"server state is a cache you rent, not state you own — and this repo shows both the power and the price of editing that cache by hand."*
