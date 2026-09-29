# Key Flows

Seven end-to-end traces. Each is referenced from other modules — read them here once, deeply. "Owner" means the layer responsible for that step (a **boundary** question: if this step misbehaves, whose bug is it?).

---

## Flow 1: Adding a link (UI → API → DB)

Why this flow matters: it's the product's core write path and shows every layer of the stack in one trip — client validation, optimistic caching, server re-validation, authorization, business rules, persistence.

Open these files first:
- `apps/web/components/ModalContent/NewLinkModal.tsx:88-100` — submit handler with client-side zod parse
- `packages/router/links.tsx:383-425` — `useAddLink` mutation fn
- `apps/web/pages/api/v1/links/index.ts:32-42` — route dispatch
- `apps/web/lib/api/controllers/links/postLink.ts` — the controller

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | UI | `NewLinkModal.tsx:88-96` | `PostLinkSchema.safeParse(link)` client-side; toast on failure | `PostLinkSchemaType` | UX-only guard; server must not trust it |
| 2 | Data layer | `links.tsx:398-404` | `new URL(link.url)` throws → "invalid_url_guide" | string url | duplicate of zod's `.url()`; two sources of truth |
| 3 | Data layer | `links.tsx:426-520` | `onMutate` inserts optimistic link with temp id `-Date.now()` into `["links"]` and `["dashboardData"]` caches | `LinkIncludingShortenedCollectionAndTags` | fabricated fields (`preview: ""`) may not match server response |
| 4 | HTTP | `links.tsx:406-418` | `POST /api/v1/links`, JSON body, cookie or Bearer auth | JSON | — |
| 5 | Route | `pages/api/v1/links/index.ts:10-11` | `verifyUser` gate; returns early writing 401/404 itself | `User` or null | dual responsibility: verifies *and* writes response |
| 6 | Route | `pages/api/v1/links/index.ts:33-37` | demo-mode guard (`NEXT_PUBLIC_DEMO`) | — | env-driven behavior |
| 7 | Controller | `postLink.ts:16-25` | server-side `PostLinkSchema.safeParse` — the *real* validation boundary | zod result | returns first issue only |
| 8 | Controller | `postLink.ts:28-30` | `isUrlSafeForServerSideFetch` (SSRF pre-check) decides `shouldPreserveUrl` | boolean | unsafe URLs are still *saved*, just never fetched |
| 9 | Controller | `postLink.ts:32-39` | `setCollection` — resolves target collection AND authorizes (`setCollection.ts:25-36` checks owner or `canCreate` member) | `Collection` or null | authz hidden inside a resolver — easy to miss in review |
| 10 | Controller | `postLink.ts:47-67` | duplicate check with www/trailing-slash normalization if `preventDuplicateLinks` | — | only checks collections the user *owns* |
| 11 | Controller | `postLink.ts:69-76` | `hasPassedLimit` plan capacity check | — | billing coupling |
| 12 | Controller | `postLink.ts:78-100` | fetch title/headers from the URL (server-side fetch!), infer type from content-type | Headers | latency on user's critical path; SSRF-gated by step 8 |
| 13 | DB | `postLink.ts:104-151` | `prisma.link.create` with `connectOrCreate` tags (unique on `[name, ownerId]`) | `Link` row | tags owned by *collection owner*, not creator |
| 14 | DB | `postLink.ts:138-148` | if not preservable: pre-mark all formats `"unavailable"` + `lastPreserved` now | — | keeps worker from ever picking it up |
| 15 | Data layer | `links.tsx` onSuccess/onError | replace temp id with server link, or roll back caches from `context.previousLinks` | — | rollback must restore *all* touched query caches |

Validation and authorization: validation at steps 1, 2, 7 (only step 7 is load-bearing); authorization at step 9 (`setCollection` → `getPermission({userId, collectionId})`, `apps/web/lib/api/setCollection.ts:25-36`).

Persistence and side effects: Prisma create (step 13), folder creation `postLink.ts:164`, and the *deferred* side effect: the worker will find this row because `lastPreserved` is null.

Tests that cover it: no direct controller test found (evidence: only 8 `*.test.ts` files exist, none for links controllers — see [09-reference/verification-log.md](../09-reference/verification-log.md)). E2E covers login only in CI (`.github/workflows/playwright-tests.yml:45` matrix `['@login']`).

What juniors usually miss: the client-side parse (step 1) is cosmetic. Delete it and the system is still correct; delete step 7 and any authenticated user can insert garbage.

What seniors notice: the **contract** between step 3 and step 15 — the optimistic object must be shaped closely enough to the server's response that the UI doesn't flicker; and the **invariant** established at step 14 (a link is either awaiting preservation, or all its format fields are non-null).

Interview angle: "How do you do optimistic updates safely?" and "Where should validation live?" — see [08-interview-prep/02](../08-interview-prep/02-frontend-framework-questions.md) Q4 and [08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md) Q1.

Drill: turn off the worker, add a link, and query the DB (`yarn prisma:studio`): which columns are null? Predict first.
Self-grade — Basic: you name the two zod parses. Solid: you explain why `setCollection` returning null means 400 not 401, and whether you agree. Strong: you can say what breaks if step 14 were removed (worker retries unsafe URLs forever) and how you'd test it.

---

## Flow 2: Authentication and session verification

Why this flow matters: every private API route starts here; it's also a case study in *stateful revocation of stateless tokens*.

Open these files first:
- `apps/web/pages/api/v1/auth/[...nextauth].ts:76-84` — env-driven provider config (60+ SSO imports above it)
- `apps/web/lib/api/verifyToken.ts` — JWT decode + revocation check
- `apps/web/lib/api/verifyUser.ts` — the full gate

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | NextAuth | `[...nextauth].ts` | Credentials (bcrypt) or one of ~60 SSO providers issues a JWT session | JWT with `id`, `jti`, `exp` | provider sprawl; config is env-driven |
| 2 | Route | any private route, e.g. `pages/api/v1/links/index.ts:10` | `verifyUser({req,res})` | — | routes must remember to call it (no middleware) |
| 3 | Auth lib | `verifyToken.ts:12-21` | `getToken` decodes JWT from cookie/header; checks `exp` manually | `JWT` or error string | string-typed errors (`typeof token === "string"`) |
| 4 | Auth lib | `verifyToken.ts:24-33` | DB lookup: is `token.jti` in `AccessToken` with `revoked: true`? | row or null | **a DB query on every request** — the price of revocable JWTs |
| 5 | Auth lib | `verifyUser.ts:27-47` | load user + subscriptions; reject if no username | `User` | second DB query per request |
| 6 | Auth lib | `verifyUser.ts:49-58` | reject unverified email when `NEXT_PUBLIC_EMAIL_PROVIDER === "true"` | — | env-conditional authz |
| 7 | Auth lib | `verifyUser.ts:60-70` | Stripe subscription check when `STRIPE_SECRET_KEY` set | — | cloud-only branch |

Validation and authorization: this flow is pure **authentication** (who are you). Authorization (what can you touch) happens later per-resource via `getPermission` — keep the distinction crisp for interviews.

Persistence and side effects: reads only. Note `AccessToken` (`packages/prisma/schema.prisma:234-246`) doubles as API keys (`isSession: false`) and revocable sessions (`isSession: true`).

Tests that cover it: e2e login spec (`apps/web/e2e/tests/public/login.spec.ts`) — the only CI-run e2e. No unit tests for `verifyToken`/`verifyUser` found.

What juniors usually miss: JWTs can't be "deleted", so the `jti` revocation table is what makes logout real. Without step 4, a stolen token is valid until `exp`.

What seniors notice: `verifyUser` writes HTTP responses itself, so controllers stay clean, but the function is now coupled to the HTTP layer and unusable elsewhere — compare `isAuthenticatedRequest.ts` which returns null instead. Two near-duplicate gates (`verifyToken.ts:19-33` vs `isAuthenticatedRequest.ts:19-33`) is a divergence risk — *possible risk: their subscription behavior already differs slightly*.

Interview angle: "JWT vs server sessions — how do you revoke a JWT?" Answer with step 4. See [08-interview-prep/01](../08-interview-prep/01-js-ts-node-deep-dive.md) Q12 and [03](../08-interview-prep/03-api-and-data-modeling-questions.md) Q5.

Drill: list every check that can reject a request in `verifyUser` and classify each as authn, authz, or product policy.
Self-grade — Basic: 3 checks. Solid: all 5 with classifications. Strong: you argue which belongs in middleware and what the migration path would be.

---

## Flow 3: The archiving pipeline (background/async)

Why this flow matters: this is the repo's most senior material — a polling job system with fairness, timeouts, idempotency, and browser lifecycle management, all without a queue library.

Open these files first:
- `apps/worker/worker.ts:11-20` — six loops started at boot
- `apps/worker/lib/getLinkBatchFairly.ts` — fair scheduling
- `apps/worker/lib/archiveHandler.ts` — the archive job itself
- `apps/worker/index.ts:3-12` — crash-restart wrapper (respawns `worker.ts` after 5s on exit)

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Worker | `linkProcessing.ts:29-43` | infinite loop; every `ARCHIVE_SCRIPT_INTERVAL` (10s) fetch a batch | `Link[]` with collection+owner+tags | poll latency vs queue push |
| 2 | Scheduler | `getLinkBatchFairly.ts:35-38` | eligibility: `url != null AND lastPreserved: null` | where-clause | `lastPreserved` is the **idempotency** marker |
| 3 | Scheduler | `getLinkBatchFairly.ts:45-81` | pick users ordered by `lastPickedAt` nulls-first (least-recently-served), filtered by active subscription/trial | `{id}[]` | fairness prevents one bulk-importer starving everyone |
| 4 | Scheduler | `getLinkBatchFairly.ts:110-145` | round-robin: take `⌊batch/users⌋` links per user until full | ids | per-user queries in a loop — O(users) round-trips |
| 5 | Scheduler | `getLinkBatchFairly.ts:159-163` | stamp `lastPickedAt` on served users | — | — |
| 6 | Worker | `linkProcessing.ts:71-72` | `Promise.allSettled` over the batch — one failure doesn't kill siblings | — | shared browser instance across parallel jobs |
| 7 | Job | `archiveHandler.ts:32-42` | **re**-check SSRF (defense in depth vs `postLink.ts:28-30`) | — | URL's DNS may have changed since creation (rebinding) |
| 8 | Job | `archiveHandler.ts:63-75, 110-198` | `Promise.race([work, timeoutPromise])` with `AbortController`; per-job browser *context* | — | race doesn't cancel Playwright work by itself |
| 9 | Job | `archiveHandler.ts:112-127` | determine type via HEAD-ish `fetchHeaders`; image/pdf get dedicated handlers | content-type | trusting remote headers |
| 10 | Job | `archiveHandler.ts:129-194` | `page.goto` → extract meta description → preview, readability, screenshot/PDF, monolith (each skipped if already present) | files via `@linkwarden/filesystem` | partial success is normal |
| 11 | Job | `archiveHandler.ts:203-230` | `finally`: stamp `lastPreserved`, mark missing formats `"unavailable"`, reset `indexVersion: null`, close context | — | **failure also stamps `lastPreserved` → no retry, ever** |
| 12 | Worker | `linkProcessing.ts:31-33` | browser restarted every 30 min ("prevent clogging") | — | acknowledges leak-by-design mitigation |
| 13 | Indexer | `linkIndexing.ts:73-159` | separate loop: links with stale `indexVersion` get pushed to Meilisearch, then `updateMany` marks them | Meili docs | dual-write consistency (see Flow 4) |

Validation and authorization: none — the worker trusts the DB. The **boundary** is the batch query; anything that gets a row into "eligible" state gets archived with the owner's settings (`archiveHandler.ts:85-107`, per-tag archival overrides else user defaults).

Persistence and side effects: heavy — Playwright fetches arbitrary internet content, files written to S3-or-local (`packages/filesystem/createFile.ts`), Wayback Machine submission is fire-and-forget (`archiveHandler.ts:118-120`, no await).

Tests that cover it: none found for the worker (no test files under `apps/worker/`). *Possible risk* — the most complex logic has the least coverage.

What juniors usually miss: there is no queue. The DB **is** the queue, and `lastPreserved: null` is the "pending" state. This is a legitimate, common pattern (cheaper ops than Redis/BullMQ) with known costs (polling, no per-job retry state, no dead-letter).

What seniors notice: the choice in step 11. Marking failures as done trades *retry* for *self-healing simplicity* (no poison-pill loops from permanently-broken URLs). A retry count column would buy both. Also step 8: when the timeout fires, the `Promise.race` resolves but the underlying Playwright calls keep running until context close — the `AbortController` is only honored by `handleMonolith` (`archiveHandler.ts:189`).

Interview angle: this *is* the system-design interview — "design a URL archiver / web crawler." Walkthrough in [08-interview-prep/04](../08-interview-prep/04-system-design-from-this-repo.md).

Drill: a link fails with a network blip. Using only the code, answer: will it retry? What columns say "failed"? How would a user force a retry?
Self-grade — Basic: correctly say "no retry." Solid: name `lastPreserved` + `"unavailable"` markers. Strong: find the re-archive path (`PUT /api/v1/links/[id]/archive` triggers by resetting those fields) and propose a bounded-retry design.

---

## Flow 4: Search (Meilisearch + Postgres)

Why this flow matters: dual-store reads are a classic consistency/authorization trap; this repo handles it the right way — the index finds, the DB decides.

Open these files first:
- `apps/web/pages/api/v1/search/index.ts:10-33` — route
- `apps/web/lib/api/controllers/search/searchLinks.ts` — controller
- `apps/worker/workers/linkIndexing.ts:133-143` — what actually gets indexed

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | UI | `packages/router/links.tsx:73-98` | `useInfiniteQuery` keyed `["links", {params}]` fetches `/api/v1/search?cursor=N&...` | pages of links | note: the *links list* hook hits the search endpoint |
| 2 | Route | `search/index.ts:10-11` | `verifyUser` | — | — |
| 3 | Route | `search/index.ts:15-28` | untyped `req.query` strings → numbers (`Number(...)`) | `LinkRequestQuery` | `sort=abc` → `NaN` silently falls to default |
| 4 | Controller | `searchLinks.ts:54-82` | if Meili configured *and* text query: token-parse, build filters, search index for **ids only** (`attributesToRetrieve: ["id"]`), limit/offset | id list | offset pagination here… |
| 5 | Controller | `searchLinks.ts:95-120` | …then Prisma re-fetches those ids **re-applying the ownership/membership where-clause** | full links | the authz re-check — never trust the index |
| 6 | Controller | `searchLinks.ts` (non-meili branch) | fallback: pure Prisma `contains` search (same shape as `getLinks.ts:93-138`) | — | ILIKE scans at scale |
| 7 | Indexer | `linkIndexing.ts:133-143` | docs denormalize `collectionOwnerId`, `collectionMemberIds`, `collectionIsPublic` for filterable authz | Meili doc | membership changes → stale index until reindex |

Validation and authorization: authn step 2; authz **twice** — Meili filter (fast, possibly stale) and Prisma where (authoritative). The invariant: *no link leaves this endpoint unless Postgres agrees you can see it.*

Persistence and side effects: read-only; index writes happen in Flow 3 step 13.

Tests that cover it: none found for `searchLinks`. The `searchQueryBuilder.ts` token parser is pure and eminently testable — good first ticket (see [06/01](../06-contribution-practice/01-good-first-tickets.md)).

What juniors usually miss: why fetch ids from Meili and then hit Postgres at all? Answer: authorization and freshness. The index is a *view*, not a source of truth.

What seniors notice: pagination semantics differ per branch — Meili uses `cursor` as an **offset** (`searchLinks.ts:65`), the Prisma fallback in `getLinks.ts:95-96` uses it as a **cursor** (`cursor: {id}` + `skip: 1`). Same query param, two meanings. *Investigate:* whether infinite scroll behaves identically in both modes when links are inserted mid-scroll.

Interview angle: "How do you keep a search index consistent with your DB?" and "cursor vs offset pagination" — [08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md) Q3, Q8.

Drill: write the exact sequence of events after a user is removed from a collection until their search results stop including its links. Which store is wrong, and for how long?
Self-grade — Basic: identify the stale-index window. Solid: point to `indexVersion` and what triggers reindexing (nothing does, on membership change — the Prisma re-check is what saves correctness). Strong: propose the minimal change to close the freshness gap and its cost.

---

## Flow 5: Serving preserved files (authorization boundary)

Why this flow matters: serving user-generated files is where IDOR and XSS live. This repo has three defenses worth memorizing.

Open these files first:
- `apps/web/pages/api/v1/archives/[linkId].ts:85-128` — GET handler
- `apps/web/lib/api/archives/resolveAccessibleArchive.ts` — the authz resolver
- `apps/web/lib/api/preserved/createPreservedFormatUrl.ts` — signed URLs for a separate content domain

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Route | `[linkId].ts:93-102` | monolith (raw HTML!) may only be served with Bearer auth when a dedicated `NEXT_PUBLIC_USER_CONTENT_DOMAIN` is configured | — | stored-XSS containment |
| 2 | Route | `[linkId].ts:105-106` | token optional → `userId` may be undefined (public collections) | — | — |
| 3 | Authz | `resolveAccessibleArchive.ts:30-45` | one query: link's collection where owner=me OR member=me OR `isPublic` | `Collection` or null | `userId || -1` sentinel for anonymous |
| 4 | Authz | `resolveAccessibleArchive.ts:70-72` | build file path `archives/{collectionId}/{linkId}{suffix}` from *DB-derived* ids, never from user input | path string | path traversal impossible by construction |
| 5 | FS | `[linkId].ts:122-127` | `readFile` from S3-or-local; `Cache-Control: private, max-age=31536000, immutable` | bytes | correct: private + immutable (files never change in place) |
| 6 | Alt path | `createPreservedFormatUrl.ts:46-84` | for the content domain: 5-minute JWT (`scope: "preserved-format"`, linkId, filePath) → `https://content-domain/api/v1/preserved/view?token=…` | signed URL | short TTL, scoped claims |
| 7 | Alt path | `createPreservedFormatUrl.ts:86-97` | `decodePreservedFormatToken` validates scope+shape strictly (type guard `isPreservedFormatToken:26-44`) | token claims | reuses NEXTAUTH_SECRET — one secret, two token types, disambiguated by `scope` |

Validation and authorization: the whole flow *is* authorization. Uploads (POST, `[linkId].ts:130-290`) additionally enforce `canCreate` membership (`userHasCreatePermission:29-38`), MIME allowlist (`:219-231`), and size limits via formidable (`:193-197`).

Persistence and side effects: file writes on POST; DB update marks which formats exist (`:262-280`).

Tests that cover it: yes — the best-tested corner of the repo: `resolveAccessibleArchive.test.ts`, `[linkId].test.ts`, `preserved/token.test.ts`, `preserved/view.test.ts`. Read them as the house style for API tests.

What juniors usually miss: why a *separate domain* for monolith HTML? Because a preserved page's scripts would otherwise run on the app's origin with the app's cookies — same-origin policy makes the domain the **blast radius** boundary.

What seniors notice: the type-guard on decoded tokens (step 7) treats a JWT as untrusted input *even after signature verification* — signature proves who wrote it, not that the shape is what this endpoint expects.

Interview angle: "How would you serve private user uploads?" — this is a complete senior answer: authz query + derived paths + signed short-TTL URLs + separate origin for HTML. [08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md) Q6.

Drill: enumerate the three ways a request can be entitled to a file in step 3 and design the test matrix (owner / member / public / stranger × preview / monolith).
Self-grade — Basic: 3 entitlements. Solid: full matrix with expected statuses. Strong: you also cover the anonymous+`userId||-1` edge and the Bearer-only monolith rule.

---

## Flow 6: Updating / moving a link (permission matrix)

Why this flow matters: densest authorization logic in the repo; the difference between *can edit* and *can move between collections*.

Open these files first:
- `apps/web/lib/api/controllers/links/linkId/updateLinkById.ts`
- `packages/prisma/schema.prisma:151-164` — `UsersAndCollections` (`canCreate/canUpdate/canDelete`)

Trace:

| Step | Owner | File | What happens | Risk |
| --- | --- | --- | --- | --- |
| 1 | Controller | `updateLinkById.ts:17-26` | zod `UpdateLinkSchema` | — |
| 2 | Controller | `updateLinkById.ts:30` | `getPermission({userId, linkId})` — *note: returns the link's collection regardless of user; caller must inspect members/owner* (`getPermission.ts:14-26`) | misuse-prone contract |
| 3 | Controller | `updateLinkById.ts:36-65` | member-of-collection may pin/unpin (writes only the `pinnedBy` relation) | early-return special case |
| 4 | Controller | `updateLinkById.ts:67-96` | target-collection access re-checked; **non-owners cannot move links across collections** (`:88-96`) | 401 used where 403 fits |
| 5 | Controller | `updateLinkById.ts:97-101` | else require owner or member `canUpdate` | — |
| 6 | Controller | `updateLinkById.ts:133-163` | if URL changed: validate, delete old preserved files, null all format fields + `lastPreserved` → **re-queues archiving** (Flow 3 eligibility) | destructive side effect before DB write |
| 7 | DB | `updateLinkById.ts:146-193` | update with tag `connectOrCreate` (deduped `:109-118`), pin connect/disconnect | tags again owned by collection owner |
| 8 | FS | `updateLinkById.ts:195-197` | if collection changed, `moveFiles` to new path | file/DB consistency if move fails |

What juniors usually miss: resetting `lastPreserved` (step 6) is how "edit URL" implicitly means "re-archive" — state machines hiding in nullable columns.

What seniors notice: steps 6–8 do file deletion *before* the DB update and file move *after* — neither is transactional with the DB. A crash between leaves orphaned or missing files. Acceptable? Depends on blast radius (worst case: re-archive). Say exactly that in an interview.

Interview angle: role/permission modeling — [08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md) Q7.

Drill: build the truth table: (owner, member+canUpdate, member−canUpdate, stranger) × (edit name, pin, move collection) → expected status.
Self-grade — Basic: owner row correct. Solid: all rows with line anchors. Strong: you spot that pinning only requires membership, not `canUpdate`, and can defend or challenge that.

---

## Flow 7: RSS ingestion (scheduled intake)

Why this flow matters: smallest complete background flow — good for teaching, and it composes Flows 1 and 3 (feeds create links, links get archived).

Open these files first:
- `apps/worker/workers/rssPolling.ts:12-38`
- `packages/lib/rssHandler.ts`
- `packages/lib/safeFetch.ts:95-124` — redirect-validating fetch

Trace:

| Step | Owner | File | What happens | Risk |
| --- | --- | --- | --- | --- |
| 1 | Worker | `rssPolling.ts:13-15` | every `NEXT_PUBLIC_RSS_POLLING_INTERVAL_MINUTES` (default 60m) load **all** subscriptions | unbounded set |
| 2 | Worker | `rssPolling.ts:19-36` | `Promise.all` over every feed; per-feed try/catch logs and continues | unbounded parallel fetches (*possible risk:* thundering herd on large installs) |
| 3 | Sec | `rssPolling.ts:22` + `safeFetch.ts:95-124` | SSRF assert, then fetch with `redirect: "manual"` loop — **each redirect hop re-validated**, max 5 | redirects are the classic SSRF bypass; handled |
| 4 | Lib | `rssHandler.ts` | compare `lastBuildDate`; create links for new items into the subscription's collection | dedupe by build date only |

What seniors notice: `safeFetch` also installs a custom DNS `lookup` (`safeFetch.ts:16-57`) so the *socket* connects to the same vetted IP — closing the DNS-rebinding TOCTOU gap between "check the hostname" and "fetch it." That plus per-hop redirect checks is a textbook SSRF defense stack (checklist: [05-quality-engineering/05](../05-quality-engineering/05-security-checklist.md)).

Interview angle: "How would you safely fetch user-supplied URLs?" — answer straight from `ssrf.ts` + `safeFetch.ts`. [08-interview-prep/01](../08-interview-prep/01-js-ts-node-deep-dive.md) Q14.

Drill: list the four independent layers between "user saves RSS url" and "worker's HTTP GET reaches a target" (schema URL check, `assertUrlIsSafeForServerSideFetch`, custom lookup, redirect loop). For each: what attack does it stop?
Self-grade — Basic: 2 layers. Solid: all 4 with the attack each blocks. Strong: you can explain why checking the URL *once* at save time is insufficient (DNS changes; that's why the worker re-validates).
