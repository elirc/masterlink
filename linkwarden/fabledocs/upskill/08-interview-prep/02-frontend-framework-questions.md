# Frontend Framework Cards (React / React Query / Next.js)

12 cards anchored to the actual UI.

## Q1: Server state vs client state — how do you organize state in a React app?

Round: frontend
What it's really testing: whether you reach for Redux out of habit.
Repo anchor: server state entirely in React Query (`packages/router/*`); client state is two small zustand stores (`apps/web/store/links.ts`, `localSettings.ts`) plus component `useState` (`NewLinkModal.tsx:43-44`).
Junior: "I use Redux/context for state."
Mid: the taxonomy — server cache (React Query: dedup, staleness, refetch), UI state (local), preferences (small store) — and why treating server data as "state you own" causes the sync bugs Redux apps drown in.
Senior: the exception cases — optimistic writes blur the line (Q4); derived data placement; when a normalized client cache (Apollo-style) is actually warranted.
Likely follow-ups: where would you put "currently selected links for bulk edit"? (Answer in repo: `store/links.ts`.)
Practice drill: inventory every source of truth on the dashboard page.

## Q2: Explain React Query's caching model using a real query you know.

Round: frontend
What it's really testing: staleness/refetch mechanics beyond "it caches."
Repo anchor: `useFetchLinks` (`packages/router/links.tsx:62-104`): `queryKey: ["links", { params }]`, infinite query, `getNextPageParam` from server cursor, `refetchOnWindowFocus: false`, `enabled: status === "authenticated"`.
Junior: "it caches API responses."
Mid: cache entries keyed by serialized key — same params = shared cache across components; `enabled` gates on auth; infinite queries store `pages[]`, which is why cache updates must map over pages (`upsertLinkInInfiniteData:162-191`).
Senior: key design as **contract** — every mutation that touches links must know every key shape that contains links (`["links"]`, `["dashboardData"]`, `["tags"]`); that fan-out (see `useAddLink` onMutate `:426-513`) is the hidden cost of fine-grained keys, and `invalidateQueries` with prefix matching is the blunt-but-safe alternative.
Likely follow-ups: staleTime vs gcTime; when to invalidate vs setQueryData.
Practice drill: list every queryKey in `packages/router/` (grep `queryKey:`) and draw which mutations touch which.

## Q3: Why does this list use infinite scroll with a cursor instead of page numbers?

Round: frontend
What it's really testing: pagination UX + data consistency reasoning.
Repo anchor: `links.tsx:72-103` (`useInfiniteQuery`, `initialPageParam: 0`, cursor from `lastPage.nextCursor`); server side `getLinks.ts:93-96` (Prisma `cursor` + `skip: 1`).
Junior: "infinite scroll is nicer."
Mid: cursor pagination stays stable under inserts/deletes (no repeated/skipped rows when new links arrive), which matters for a feed users add to constantly; page numbers break there.
Senior: notes the repo's inconsistency — the Meili branch treats `cursor` as an offset (`searchLinks.ts:65`) — and what that means at scale (deep-offset cost, drift under writes); trade: cursors can't jump to page N.
Likely follow-ups: how do you build "jump to date" with cursors?
Practice drill: explain what `skip: 1` does and why it's needed with cursor pagination.

## Q4: Implement an optimistic update. What has to be true for it to be safe?

Round: frontend (staple question)
What it's really testing: cache-consistency discipline.
Repo anchor: `useAddLink` (`links.tsx:383-534`): `onMutate` cancels in-flight queries, snapshots `previousLinks`/`previousDashboard`, inserts a temp link with id `-Date.now()` (`:458`) into every affected cache, returns context; `onError` restores snapshots; success swaps temp for real.
Junior: "update the UI before the server responds."
Mid: the four required pieces — cancel, snapshot, patch all caches, rollback — and the temp-id trick (negative = can't collide with autoincrement).
Senior: the invariants — optimistic shape must be close enough to the server's (here the server may *rename* the link from its fetched title, so the optimistic name is knowingly wrong-then-corrected); every new cache that renders links silently opts out of both patch and rollback (maintenance trap); when *not* to be optimistic (low success rate, heavy server transformation, money).
Likely follow-ups: what if two optimistic mutations overlap?
Practice drill: code-read `:426-534` and write the bug report for a cache the rollback would miss if someone added `["collections", id, "links"]`.

## Q5: Rules of hooks — find the violation risk in this real hook.

Round: frontend
What it's really testing: depth beyond "don't put hooks in ifs."
Repo anchor: `useFetchLinks` (`links.tsx:62-71`) calls `useSession()` **inside** `if (!auth)`.
Junior: "hooks can't be conditional" (and stops).
Mid: explains *why* — hook state is positional per render; conditional calls shift the order and corrupt state; this code survives because `auth` is fixed for a component's lifetime (web passes nothing, mobile always passes it), so the branch never flips.
Senior: calls it what it is — an unenforced invariant held by convention across two apps; fixes: split into two hooks, or always call `useSession` and ignore it on mobile (but RN lacks the provider — hence the mess); this is the *real* cost of sharing hooks across platforms.
Likely follow-ups: how does the lint rule catch this? (It should — *investigate* how it's suppressed or missed here.)
Practice drill: write the two-hook refactor signature that removes the conditional.

## Q6: useEffect dependencies — when is an empty array correct, and when is it a bug?

Round: frontend
What it's really testing: effect-model understanding (top interview filter).
Repo anchor: `NewLinkModal.tsx:63-82` — `[]` effect intentionally reads mount-time route/collections to prefill the form (correct: it's an initialization); `useLayoutEffect:84-86` for focus (correct: must fire before paint). Counter-example shape: `useLinks`'s `links` memo depends on `query.dataUpdatedAt` (`links.tsx:52-54`) — a deliberate "recompute when data timestamp changes" instead of on the array identity.
Junior: "empty array = run once."
Mid: the question is *what values does the effect close over, and is stale acceptable?* — initialization yes, subscriptions no; exhaustive-deps as default with justified exceptions.
Senior: effects as synchronization with external systems (not lifecycle); the `dataUpdatedAt` memo is a performance workaround with a subtle contract (identity of `query.data` alone is not trusted) — evaluate whether it can miss updates.
Likely follow-ups: why does StrictMode double-run effects; cleanup functions.
Practice drill: for each of the three anchors, state what staleness would look like as a user-visible bug.

## Q7: How does this app avoid prop drilling without global state everywhere?

Round: frontend
What it's really testing: composition instincts.
Repo anchor: data hooks called where needed (`useCollections()` directly in `NewLinkModal.tsx:46` — React Query's cache *is* the shared layer); permissions computed via a hook (`apps/web/hooks/useCollectivePermissions.ts`) instead of threaded through props.
Junior: "context or Redux."
Mid: React Query dedupes identical queries, so "just call the hook again" replaces drilling for server data; context reserved for true cross-cutting values (session, theme, i18n).
Senior: the tradeoff — hook-everywhere hides data dependencies from the component tree (harder to see what a page fetches; waterfalls possible); server-side prefetch + hydration is the counterweight.
Likely follow-ups: when does context cause re-render storms; selector patterns.
Practice drill: map every hook `NewLinkModal` calls and what cache each reads.

## Q8: What re-renders when a link is added, and how would you find out?

Round: frontend / performance
What it's really testing: render-model + measurement-first mindset.
Repo anchor: `useAddLink`'s `setQueriesData` touches every `["links"]` query (`links.tsx:507-513`) → all subscribed components re-render; list rendering lives in `apps/web/components/LinkViews/`.
Junior: "the list re-renders."
Mid: React re-renders subscribers of changed cache entries, then reconciles; keys matter for list diffing (stable link ids, and the temp-id swap on success is exactly a key-stability hazard — same entity, id changes from negative to real, so React remounts that row).
Senior: measure first (React DevTools Profiler), then targeted fixes — `select` in useQuery, memoized row components; and the remount-on-id-swap is a real flicker source worth verifying before "optimizing."
Likely follow-ups: React.memo vs useMemo; when memoization hurts.
Practice drill: predict, then profile (if running locally), adding one link with a 200-link list mounted.

## Q9: How do you share UI data logic between web and React Native?

Round: frontend / architecture
What it's really testing: abstraction judgment across platforms.
Repo anchor: entire `packages/router/` — hooks take `auth?: MobileAuth` (Bearer vs cookie, `links.tsx:82-91`), and side-effect deps (`toast`, `Alert`, `t`) are injected (`:383-393, 521-524`).
Junior: "put shared code in a shared folder."
Mid: what can be shared (server-state logic, validation, types) vs what can't (navigation, storage, styling); DI at the hook boundary keeps platform imports out.
Senior: evaluates the seams — the injected-arg list grows without design pressure (adapter interface would scale); auth-mode branching inside hooks (Q5's conditional `useSession`) is the friction point; alternative: platform-specific thin wrappers around platform-free core functions.
Likely follow-ups: how would you test these hooks? (renderHook + mock fetch + QueryClientProvider.)
Practice drill: sketch the `PlatformAdapter` type that replaces `{auth, Alert, toast, t}`.

## Q10: Where does this app validate user input on the client, and is it enough?

Round: frontend / security
What it's really testing: trust-boundary clarity from the UI side.
Repo anchor: `NewLinkModal.tsx:88-96` client zod parse (same `PostLinkSchema` as the server); `links.tsx:398-404` `new URL()` guard; the *real* boundary is server-side `postLink.ts:16-25`.
Junior: "validate in the form."
Mid: client validation = UX; server validation = security; sharing one schema keeps them honest; DOMPurify appears where user HTML is rendered (dompurify in web+worker deps — used for readable content sanitization).
Senior: enumerates what client validation can never do (authz, quota, uniqueness, SSRF) and connects each to its server-side line; XSS story for preserved HTML = separate origin (pattern 13 in the [catalog](../03-architecture-and-patterns/05-pattern-catalog.md)).
Likely follow-ups: where would you sanitize — on write or on render? (Render-side, and why.)
Practice drill: submit (locally) a 3000-char description with devtools JS disabled-validation and confirm the server 400.

## Q11: Next.js pages router — how do SSR and API routes fit together here? Would you migrate to the app router?

Round: frontend / framework
What it's really testing: framework-evolution judgment, not hype.
Repo anchor: pages router throughout (`apps/web/pages/`); `getServerSideProps` wrapper (`apps/web/lib/client/getServerSideProps.ts`); API routes as the *entire* backend (`pages/api/v1/**`); `_app.tsx`/`_document.tsx` shell.
Junior: describes file-based routing.
Mid: pages router = per-page SSR functions + client components; app router = RSC, layouts, streaming; and the honest assessment that this app is a highly interactive authenticated dashboard — RSC's biggest wins (static content, less client JS) apply weakly.
Senior: migration math — 30+ pages, a React Query architecture that already works, mobile app sharing the same hooks; incremental adoption is possible but the payoff doesn't clear the risk today; "stay, but keep API routes framework-agnostic (controllers already are) so the option stays open."
Likely follow-ups: what *would* change your mind? (Public marketing/SEO surface growth; server-heavy pages.)
Practice drill: pick one page and sketch its app-router equivalent honestly, including where React Query still lives.

## Q12: i18n in a real app — what's involved beyond t('key')?

Round: frontend
What it's really testing: production-completeness thinking.
Repo anchor: next-i18next config (`apps/web/next-i18next.config.js`), `useTranslation` everywhere (`NewLinkModal.tsx:24`), locale on the User model (`schema.prisma:37`), crowdin pipeline (`crowdin.yml`, `locale-action.yml` workflow), and error messages translated client-side by key (`links.tsx:522` — server sends a key-ish string, client `t(error.message)`).
Junior: "extract strings to JSON."
Mid: SSR locale detection, per-user persisted locale, translation workflow (crowdin = external translators), and the interesting pattern of translating *server errors* on the client by treating messages as keys.
Senior: flags that error-message-as-translation-key couples the API contract to UI copy — an API consumer sees `invalid_url_guide`; better: stable error *codes* field, message separate.
Likely follow-ups: pluralization/RTL; date formatting.
Practice drill: trace one error string from `postLink.ts` to rendered toast and note every transformation.
