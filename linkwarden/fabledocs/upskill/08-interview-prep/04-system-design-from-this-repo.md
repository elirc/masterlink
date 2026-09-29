# System Design From This Repo

The exercise: **"Design a collaborative bookmark manager that permanently archives saved pages."** You are reverse-engineering Linkwarden onto a whiteboard. At every step: what the repo actually chose (with anchors), what junior/mid/senior answers sound like, and a stronger-or-simpler alternative.

Rehearse this aloud in 35 minutes: requirements 5, API 5, data model 8, archiving pipeline 10, scaling/tradeoffs 7.

---

## Step 1 — Requirements (5 min)

Functional: save link → auto-capture title/metadata; organize into shareable collections with per-member write permissions; preserve pages (screenshot/PDF/single-file HTML/article text); full-text search; tags (manual + AI); import/export; public collections; API keys for integrations.

Non-functional: archiving is slow (seconds–minutes) → **must be async**; archived files are large → object storage; fetching user URLs → **SSRF is in scope from minute one** (saying this unprompted is a senior signal); self-hostable → every piece of infra optional.

- Junior: lists features. Mid: separates sync path (save) from async path (archive) and says *why*. Senior: names the trust boundary — "the server will fetch attacker-controlled URLs; that constrains the whole design."

## Step 2 — API sketch (5 min)

What the repo chose: REST-ish `/api/v1/*` (links, collections, tags, archives, search, tokens…), cookie session or Bearer token, per-route gate function.

```
POST   /api/v1/links                 create (returns immediately; archiving deferred)
GET    /api/v1/search?cursor&query   list/search (paginated)
PATCH  /api/v1/links/:id             edit/move/pin
GET    /api/v1/archives/:linkId?format=  serve preserved file (authz inside)
POST   /api/v1/collections/:id/members   share
```

Anchors: route dispatch `apps/web/pages/api/v1/links/index.ts:13-71`; auth gate `verifyUser.ts`; the deliberate choice that `POST /links` returns **before** archiving (`postLink.ts` never touches Playwright — the worker does).

- Junior: CRUD endpoints. Mid: points at the 202-style async contract — create returns a link whose `preview/image/pdf` fields fill in later; client polls or refetches. Senior: also versions the API (`v1`, and `v2/dashboard` already exists — `pages/api/v2/dashboard/index.ts`), and defines error contract (this repo's is weak: string `response` bodies, 401-for-403 — an honest critique to volunteer).

Alternative: tRPC/GraphQL for the first-party clients — but the public API-key audience (`AccessToken`, `schema.prisma:234-246`) is exactly who REST serves best.

## Step 3 — Data model (8 min)

Whiteboard the five core tables (all real — `packages/prisma/schema.prisma`):

```
User (id, …settings)                          :28-75
Collection (id, ownerId, parentId, isPublic)  :126-149
UsersAndCollections (userId, collectionId,    :151-164   ← the sharing/permission join
                     canCreate, canUpdate, canDelete; PK [userId, collectionId])
Link (id, collectionId, url, name, type,      :166-198
      preview/image/pdf/readable/monolith,    ← archive artifact paths (or "unavailable")
      lastPreserved, indexVersion, textContent)
Tag (id, name, ownerId, UNIQUE[name, ownerId]):200-218
```

Design decisions to narrate:

1. **Permissions as three booleans on the join row** — not roles. Fine at this scale; the moment you need "admin" or inheritance you migrate to a role enum or policy table. (`updateLinkById.ts:72-101` shows the read side.)
2. **Job state embedded in the entity**: `lastPreserved IS NULL` = pending; `"unavailable"` string = tried-and-failed for a format. Cheap, inspectable; but no retry counts/timestamps per attempt. Alternative: a separate `archive_jobs` table (id, linkId, state, attempts, lastError) — more moving parts, real retry semantics.
3. **Ownership subtleties**: links live in collections; the collection **owner** owns quota and tags even when a member created the link (`postLink.ts:120-137`, quota counted by owner at `archives/[linkId].ts:41-55`). Multi-tenancy is per-collection, not per-user.
4. Indexes: `Link@@index([collectionId])`, `Collection@@index([ownerId])`, join PK — the exact columns every hot where-clause hits (compare `getLinks.ts:97-128`).

- Junior: gets tables. Mid: explains the join-table permissions and the null-as-state trick with its costs. Senior: calls out what's *missing* — soft deletes, audit trail, per-attempt job history — and whether each is worth its cost here.

## Step 4 — The archiving pipeline (10 min — the heart of the interview)

What the repo chose (Flow 3 in [key flows](../01-codebase-cartography/05-key-flows.md)):

```
web writes Link(lastPreserved=null) ──> Postgres
worker loop (every 10s):
  pick batch FAIRLY across users        getLinkBatchFairly.ts:45-145
  re-check SSRF per link                archiveHandler.ts:32-42
  per link: Playwright context, Promise.race(work, timeout)   :63-198
  finally: stamp lastPreserved, mark missing formats "unavailable"  :203-230
files → S3-or-local                     packages/filesystem/
separate loop indexes into Meilisearch  linkIndexing.ts:73-159
supervisor restarts worker on crash     apps/worker/index.ts:3-16
browser rotated every 30 min            linkProcessing.ts:29-33
```

Narrate the four load-bearing decisions:

1. **DB-as-queue, no broker.** Right call for self-hosted (one less service); costs: polling latency, no built-in retry/DLQ, and — critically — **horizontal scaling requires row locking** (two workers would grab the same batch; fix = `FOR UPDATE SKIP LOCKED` or advisory locks). Saying that unprompted is the senior move.
2. **Fair scheduling** (`lastPickedAt` round-robin) — solves noisy-neighbor at the scheduler rather than rate-limiting ingestion.
3. **Failure = done.** A crashed archive still gets `lastPreserved` stamped (`archiveHandler.ts:203-224`), trading retries for guaranteed progress (no poison pills). Alternative: bounded retries with exponential backoff + dead-letter state; requires the jobs table from step 3.
4. **Untrusted content isolation**: SSRF re-check at time-of-use (DNS may have changed since save), per-job browser *context* not shared pages, hard timeout, and preserved HTML served from a **separate origin** with signed 5-minute scoped tokens (`createPreservedFormatUrl.ts:46-84`, enforced `[linkId].ts:93-102`).

- Junior: "a cron job archives links." Mid: DB-as-queue with idempotency column + timeout + why failures are marked done. Senior: adds multi-worker locking, backpressure (`ARCHIVE_TAKE_COUNT` caps batch; `countUnprocessedBillableLinks` gives queue-depth observability, `linkProcessing.ts:74-81`), and the security containment story.

## Step 5 — Search (5 min)

Chose: Meilisearch sidecar, **ids-only** results re-verified against Postgres with the full ownership where-clause (`searchLinks.ts:67-71` → `:95-120`); authz fields denormalized into index docs for pre-filtering (`linkIndexing.ts:133-143`); `indexVersion` constant bump = global reindex. Fallback: plain Postgres `contains` when Meili unconfigured.

Key line for the interviewer: *"the index finds, the database decides — a stale index can hide results but can never leak them."*

Alternative: Postgres FTS (`tsvector`) — one less service, weaker typo tolerance; the right *first* choice for most products, and worth saying so.

## Step 6 — Scaling & tradeoffs summary (7 min)

| Pressure | First bottleneck | Fix path |
| --- | --- | --- |
| 10× users reading | search + list queries | Meili already offloads text search; add read replicas; watch `verifyToken`'s per-request revocation query (`verifyToken.ts:24-33`) → cache with short TTL |
| 10× archive volume | single worker, single browser | multiple workers + `SKIP LOCKED`; browser pool or remote browsers (`PLAYWRIGHT_WS_URL` already supported) |
| Big files | disk | S3 backend already env-selectable (`packages/filesystem/s3Client.ts`) |
| Abuse | server fetches attacker URLs | SSRF stack (in place) + rate limiting (absent — honest gap) + per-plan quotas (`verifyCapacity.ts`) |

## Variation prompts (practice each in 10 min)

1. **"Add real-time collaboration"** — presence + live updates in shared collections. Discuss: WebSocket/SSE fan-out keyed by collectionId; who invalidates React Query caches; why polling might be fine (write rates are low).
2. **"Now it's multi-tenant SaaS for companies"** — orgs above users; permission model migration from 3 booleans to roles; per-org quotas; data isolation tests (`orgId` on every where-clause — extend the pattern in `getLinks.ts:100-109`).
3. **"Handle 10× archive traffic"** — worker fleet + locking; queue-depth metrics; per-domain politeness (don't hammer one site); browser pool sizing.
4. **"Add end-to-end encryption for private collections"** — what breaks: server-side archiving (server must read pages), search indexing, AI tagging. A great "when requirements conflict" conversation — E2EE and server-side preservation are fundamentally at odds; senior answer negotiates scope (encrypt notes/highlights, not archives).

## Cross-links

Architecture critique that doubles as design-review material: [03-architecture-and-patterns/06-architecture-critique.md](../03-architecture-and-patterns/06-architecture-critique.md). Pattern vocabulary: [pattern catalog](../03-architecture-and-patterns/05-pattern-catalog.md) cards 5–8, 10, 13.
