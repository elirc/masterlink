# Pattern Catalog

Fourteen patterns this repo actually uses. The goal is **recognition** — seeing the shape in any codebase — not name-dropping. Each card: problem → shape → real anchors → failure modes → drill.

---

## Pattern 1: Route → Controller split

Problem it solves: keeps HTTP mechanics (method dispatch, query parsing, status writing) out of business logic so the logic is testable and reusable.
General shape: thin `pages/api/**` file per URL; imports one controller per verb; controller returns `{response, status}` and never touches `req`/`res`.
Real example: `apps/web/pages/api/v1/links/index.ts:13-71` dispatching to `apps/web/lib/api/controllers/links/postLink.ts`.
Second example: `apps/web/pages/api/v1/search/index.ts:14-36` → `controllers/search/searchLinks.ts`.
Why this implementation works: controllers are plain async functions — the archives tests (`pages/api/v1/archives/[linkId].test.ts`) exercise them without a running server.
Failure modes: the contract `{response, status}` is a convention, not a type shared across all controllers — drift produces routes that forget to forward status codes.
Use it when: any framework with file-based routes. Avoid it when: a framework gives you real middleware/DI (Nest, Fastify plugins) — don't hand-roll two layers there.
Interview angle: "How do you structure an Express/Next API?" — describe this split plus where auth lives.
Drill: pick `pages/api/v1/tags/index.ts` and verify it follows the pattern; note any deviation.

## Pattern 2: Gate function that writes the response (`verifyUser`)

Problem it solves: every private route needs identical authn + account-state checks.
General shape: `const user = await verifyUser({req,res}); if (!user) return;` — the gate writes 401/404 itself.
Real example: `apps/web/lib/api/verifyUser.ts:14-73`; used at `pages/api/v1/links/index.ts:10-11`.
Second example: token-only variant returning data instead of writing responses: `apps/web/lib/api/isAuthenticatedRequest.ts`.
Why this implementation works: one-liner at each callsite; impossible to half-apply.
Failure modes: (1) it's opt-in — a new route that forgets the call is silently public; (2) coupling to `res` makes it unusable in non-HTTP contexts, hence the near-duplicate `isAuthenticatedRequest` (divergence risk).
Use it when: pages-router Next.js with no middleware layer. Avoid it when: you have `middleware.ts`/route groups that can enforce auth centrally.
Interview angle: "How do you make sure no endpoint ships unauthenticated?" — mid answer: convention + review; senior answer: move the **default** to deny (middleware, or a route-builder wrapper) so forgetting fails closed.
Drill: find one route that *intentionally* skips `verifyUser` (hint: `pages/api/v1/public/**`) and explain why that's safe.

## Pattern 3: Zod schema as the API contract

Problem it solves: request bodies are `any` at the boundary; you need one source of truth for shape + limits.
General shape: schemas in a shared package; controller starts with `Schema.safeParse(body)`; types derived via `z.infer`.
Real example: `packages/lib/schemaValidation.ts:125-148` (`PostLinkSchema`: `.trim().max(2048).url()`) consumed at `postLink.ts:16-25`.
Second example: the same schema reused **client-side** at `NewLinkModal.tsx:89` — one contract, two enforcement points.
Why this implementation works: max-lengths on every string are cheap DoS protection; sharing the schema kills client/server drift.
Failure modes: schemas that are functions of env (`PostUserSchema()` at `schemaValidation.ts:36`) change shape per deployment — tests must pin env; only the first zod issue is reported (`postLink.ts:20-22`), which hides multi-field errors.
Use it when: always, at every trust boundary. Avoid it when: never — but don't re-validate deep in internal calls; validate once at the edge.
Interview angle: "Where do you validate?" — edge, with the schema shared inward. [08/03 Q1](../08-interview-prep/03-api-and-data-modeling-questions.md).
Drill: add (locally) a `description` of 3000 chars via curl and confirm the 400 comes from zod, not the DB.

## Pattern 4: Permission resolver (`getPermission` / `setCollection`)

Problem it solves: authorization rules (owner OR member-with-flag) repeated across every controller.
General shape: one function answers "what is this user's relationship to this collection?"; callers apply the verb-specific rule.
Real example: `apps/web/lib/api/getPermission.ts:9-38`; verb rule applied at `updateLinkById.ts:72-101`.
Second example: `setCollection.ts:25-36` (resolve + `canCreate` check fused).
Why this implementation works: the `UsersAndCollections` join table (`schema.prisma:151-164`) makes membership one indexed query.
Failure modes: `getPermission({linkId})` (`getPermission.ts:14-26`) returns the collection **without filtering by user** — every caller must remember to check `ownerId`/`members`. A forgotten check = IDOR. This is a *sharp* contract; a resolver returning `{role}` would be safer.
Use it when: small permission matrices (3 boolean flags). Avoid it when: roles multiply — reach for a policy layer (CASL, OPA) before writing the 10th flag.
Interview angle: IDOR — "how do you prevent users reading each other's data?" [08/03 Q7](../08-interview-prep/03-api-and-data-modeling-questions.md).
Drill: grep for `getPermission(` and audit three call sites: does each one check the result correctly?

## Pattern 5: DB-as-queue with idempotency column

Problem it solves: background work without deploying Redis/BullMQ/SQS.
General shape: "pending" is a predicate on existing rows (`lastPreserved: null`); workers poll, process, stamp.
Real example: eligibility at `apps/worker/lib/getLinkBatchFairly.ts:35-38`; stamp at `archiveHandler.ts:203-224`.
Second example: Meili indexing uses `indexVersion != CURRENT` as its pending predicate (`linkIndexing.ts:73-81`), and version-bumping reindexes *everything* — a neat schema-migration trick for search.
Why this implementation works: zero extra infrastructure for a self-hosted product; state is inspectable with SQL.
Failure modes: no retry counts, no dead-letter, poll latency, and multi-worker deployments would double-process without row locking (`SELECT … FOR UPDATE SKIP LOCKED` is the classic fix).
Use it when: single worker, self-hosted, low throughput. Avoid it when: you need retries with backoff, fan-out, or >1 consumer.
Interview angle: "Build a job queue on Postgres" — canonical mid/senior system-design probe. [08/04](../08-interview-prep/04-system-design-from-this-repo.md).
Drill: write the SQL to count links stuck "pending" older than a day — your first observability query for this system.

## Pattern 6: Fair scheduling (anti-starvation round-robin)

Problem it solves: one user importing 10k bookmarks starves everyone else's archiving.
General shape: order users by `lastPickedAt` nulls-first; take `⌊batch/users⌋` items per user; stamp the served.
Real example: `apps/worker/lib/getLinkBatchFairly.ts:45-81` (user pick) and `:110-145` (round-robin fill).
Second example: No second example found (the tag-mode reuses the same function with a different where-clause, `:18-34`).
Why this implementation works: `lastPickedAt` turns fairness into an `orderBy` — no scheduler process needed.
Failure modes: per-user queries inside loops (O(users) round-trips per tick); fairness granularity is per-tick, not per-cost (one heavy PDF counts same as a favicon).
Use it when: multi-tenant shared workers. Avoid it when: single-tenant/self-host with one user — it's pure overhead there (note: it runs anyway; a self-hoster pays a little).
Interview angle: "How do you prevent noisy-neighbor problems?" — rate limiting on the way in, fair scheduling on the way through.
Drill: simulate on paper — users A(12 links), B(2), C(1), batch=5. Who gets what, and what does `lastPickedAt` look like after two ticks?

## Pattern 7: Timeout via `Promise.race` + AbortController

Problem it solves: a hung page load must not wedge the worker forever.
General shape: race the real work against a `setTimeout` promise that also fires `abortController.abort()`; clear the timer in `finally`.
Real example: `apps/worker/lib/archiveHandler.ts:63-75` (setup), `:110-198` (race), `:203-207` (cleanup).
Second example: `safeFetch`'s bounded redirect loop is the same "hard ceiling on untrusted work" idea (`packages/lib/safeFetch.ts:103-124`).
Why this implementation works: guarantees the *loop* proceeds even if Playwright hangs.
Failure modes: losing a race doesn't cancel the loser — Playwright calls keep running until the context closes; only `handleMonolith` receives the signal (`archiveHandler.ts:189`). Racing without abort **leaks work**; say this in interviews.
Use it when: any await on the outside world. Avoid it when: the library takes a native timeout option — prefer that.
Interview angle: "How do you time out an async operation in Node?" [08/01 Q5](../08-interview-prep/01-js-ts-node-deep-dive.md).
Drill: trace what happens to the open `page` when the timeout wins: who closes it, on which line?

## Pattern 8: Defense-in-depth SSRF guard

Problem it solves: the server fetches user-supplied URLs (title fetch, archiving, RSS) — a direct line to your cloud metadata endpoint unless blocked.
General shape: parse URL → protocol allowlist → block private hostnames/CIDRs → resolve DNS and check **every** IP → pin the socket to vetted IPs via custom lookup → re-validate on every redirect hop → re-check at time-of-use.
Real example: `packages/lib/ssrf.ts:306-329` (`assertUrlIsSafeForServerSideFetch`), CIDR tables `:37-61`.
Second example: socket pinning `safeFetch.ts:16-57`; redirect loop `:103-124`; time-of-use re-check `archiveHandler.ts:32-42`.
Why this implementation works: it closes the two classic bypasses — redirects and DNS rebinding (check-then-fetch TOCTOU).
Failure modes: allowlist-of-blocklists always has edge cases (new cloud metadata ranges); `ALLOW_PRIVATE_NETWORK_ACCESS=true` (ssrf.ts:319-324) is a footgun for self-hosters who archive intranet pages.
Use it when: any server-side fetch of user input. Avoid it when: never avoid; but egress-proxy/network-policy is the stronger production control.
Interview angle: "What's SSRF and how do you stop it?" — this repo is your worked answer. [08/01 Q14](../08-interview-prep/01-js-ts-node-deep-dive.md).
Drill: read `packages/lib/ssrf.test.ts` and list which bypass each test encodes.

## Pattern 9: Optimistic mutation with rollback context

Problem it solves: adding a link should feel instant despite a slow POST (which may fetch the page title synchronously).
General shape: `onMutate` cancels queries, snapshots previous data, writes a fake row with a **temp negative id**, returns snapshot as context; `onError` restores; `onSuccess` swaps temp for real.
Real example: `packages/router/links.tsx:426-520` (`onMutate` — temp id `-Date.now()` at `:458`), rollback `:521-534`.
Second example: the upsert helpers reused across caches: `upsertLinkInInfiniteData` `:162-191`, `upsertLinkInDashboardData` `:193-222`.
Why this implementation works: negative ids can't collide with Postgres autoincrement; every touched cache (`["links"]`, `["dashboardData"]`) has a matching rollback.
Failure modes: the optimistic object is hand-assembled (`:487-505`) — every server-side field you forget (e.g. `pinnedBy`) is a UI flicker; new caches added later (a new dashboard widget) silently miss both upsert and rollback.
Use it when: high-frequency, high-success writes. Avoid it when: server materially transforms the entity (here the server may *rename* the link from the fetched title — the optimistic name can be wrong until success).
Interview angle: staple React Query question. [08/02 Q4](../08-interview-prep/02-frontend-framework-questions.md).
Drill: kill the network in devtools, add a link, and narrate the cache timeline (write → error → rollback) against the code.

## Pattern 10: Search-index find, database decide

Problem it solves: full-text search needs an index; authorization needs the source of truth.
General shape: index returns candidate **ids only**; DB re-fetches with the full authz where-clause.
Real example: `apps/web/lib/api/controllers/search/searchLinks.ts:67-71` (`attributesToRetrieve: ["id"]`) then `:95-120` (Prisma re-check).
Second example: denormalized authz fields in the index for cheap *pre*-filtering: `linkIndexing.ts:133-143`.
Why this implementation works: a stale index can cause missing results (annoying) but never leaked results (dangerous) — it fails in the safe direction.
Failure modes: two stores to keep in sync (`indexVersion` protocol); ranking done in Meili but final list from Postgres must preserve Meili's order — *investigate:* whether result order survives the `id IN (...)` re-fetch.
Use it when: any search sidecar (Elastic, Meili, Typesense) next to a relational source of truth.
Interview angle: dual-store consistency. [08/03 Q8](../08-interview-prep/03-api-and-data-modeling-questions.md).
Drill: diagram the write path (Flow 3 step 13) and read path (Flow 4) and mark where they can disagree.

## Pattern 11: Storage abstraction with env-selected backend

Problem it solves: self-hosters want a folder; cloud wants S3 — same code paths.
General shape: module-level client that's `undefined` unless env is set; every function branches `if (s3Client) … else fs …`.
Real example: `packages/filesystem/s3Client.ts` (conditional construction), `createFile.ts` (dual write paths).
Second example: `meilisearchClient.ts:1-10` — identical trick; all search code checks `if (meiliClient)`.
Why this implementation works: features degrade gracefully; the *absence of config* is the feature flag.
Failure modes: behavior differences between backends (local path joins vs S3 keys, error types) surface only in the deployment you didn't test; module-level singletons resist test injection (note `ssrf.ts` solves this by accepting a `lookup` parameter — compare!).
Use it when: honest 2-backend needs. Avoid it when: a third backend appears — switch to an interface + factory.
Interview angle: "How do you support both local dev and cloud storage?" plus testability tradeoffs of singletons.
Drill: list every `process.env` read in `packages/filesystem/` and `packages/lib/meilisearchClient.ts`; write the one-paragraph ops doc a self-hoster would need.

## Pattern 12: Shared data-layer package across web + mobile

Problem it solves: two clients (Next.js, React Native) need identical server-state logic.
General shape: all React Query hooks live in `packages/router/*`; platform differences injected as parameters (`auth?: MobileAuth`, `toast`, `Alert`, `t`).
Real example: `packages/router/links.tsx:26-60` (`useLinks(params, auth?)`), Bearer-vs-cookie branch `:82-91`, toast/Alert injection `:383-393` and `:521-524`.
Second example: `apps/web/components/ModalContent/NewLinkModal.tsx:17,37-40` consuming it with web dependencies.
Why this implementation works: dependency injection at the hook boundary keeps `packages/router` free of platform imports.
Failure modes: the injected surface grows argument-by-argument (`{auth, Alert, toast, t}`) — a context object or adapter interface scales better; conditional `useSession()` call at `links.tsx:65-70` runs a hook inside an `if`, which is only safe because `auth` never changes between renders — *fragile invariant, know it exists* (React's rules-of-hooks).
Use it when: ≥2 JS clients share a backend. Avoid it when: one client — premature abstraction.
Interview angle: monorepo code-sharing strategy; also a great rules-of-hooks discussion. [08/02 Q9](../08-interview-prep/02-frontend-framework-questions.md).
Drill: find every platform-injected dependency in `useAddLink` and sketch the `PlatformAdapter` interface that would replace them.

## Pattern 13: Signed, scoped, short-TTL URL for user content

Problem it solves: serving stored HTML on your app origin = stored XSS with session cookies in reach.
General shape: separate content domain + 5-minute JWT carrying `scope`, resource id, and exact file path; strict type-guard on decode.
Real example: `apps/web/lib/api/preserved/createPreservedFormatUrl.ts:46-84` (encode), `:26-44` (type guard), TTL `:7`.
Second example: enforcement that monolith HTML *must* use this path when the domain is configured: `pages/api/v1/archives/[linkId].ts:93-102`.
Why this implementation works: the browser treats the content domain as a different origin — scripts in preserved pages can't touch app cookies. The `scope` claim prevents replaying a session JWT here (one secret, two token types).
Failure modes: forgetting to scope claims lets any signed JWT open any file; long TTLs turn links into de-facto permanent capabilities.
Use it when: serving user uploads, exports, previews. Avoid it when: content is public anyway — plain CDN.
Interview angle: this is the S3 presigned-URL pattern, hand-rolled — perfect compare/contrast. [08/03 Q6](../08-interview-prep/03-api-and-data-modeling-questions.md).
Drill: read `pages/api/v1/preserved/token.test.ts` and `view.test.ts`; list the negative cases they cover and one they don't.

## Pattern 14: Supervisor self-restart wrapper

Problem it solves: a long-running worker that crashes must come back without a human.
General shape: tiny parent process spawns the real worker, listens for `exit`, respawns after a delay.
Real example: `apps/worker/index.ts:3-16` (spawn `tsx worker.ts`, restart after 5s).
Second example: browser rotation every 30 minutes inside the worker (`linkProcessing.ts:29-33`) — same philosophy (assume leaks, reset proactively) at a different granularity.
Why this implementation works: 14 lines replace pm2 for the self-hosted case; Docker restart policies cover the container case.
Failure modes: no crash-loop backoff (5s flat → tight loop on persistent failure); no alerting — it hides failures instead of surfacing them (an **observability** gap).
Use it when: simple deployments without an init system. Avoid it when: k8s/systemd/pm2 already supervise you — double supervision confuses restarts.
Interview angle: "What happens when your worker crashes at 3am?" — restart story + the follow-up you should volunteer: how do you *know* it crashed?
Drill: add (locally) a `console.error` counter — after how many restarts in 5 minutes would you want a page? Where would that alert live?
