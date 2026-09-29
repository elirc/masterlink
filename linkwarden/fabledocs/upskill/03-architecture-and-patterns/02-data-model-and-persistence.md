# Data Model and Persistence

Source of truth: [packages/prisma/schema.prisma](../../../packages/prisma/schema.prisma) (Postgres, `datasource:5-8`), 93 migrations, seed in `packages/prisma/seed.js`.

## The entity graph that matters

```
User ──owns──> Collection ──contains──> Link ──m:n──> Tag (owned by User)
  │                │  └─ parentId (self-nesting)         │
  │                └──m:n via UsersAndCollections────────┘ (permission booleans)
  ├── AccessToken (API keys + revocable sessions)
  ├── Subscription (Stripe; parentSubscription → seat-based teams)
  ├── Highlight ──> Link
  ├── DashboardSection ──> Collection?
  └── RssSubscription ──> Collection
```

Key relationship facts (each is a potential interview probe):

- **Tenancy is per-collection, not per-user**: a Link belongs to a Collection (`Link:175-176`); who can see it derives from the collection's owner/members/isPublic. All isolation queries phrase it that way (`getLinks.ts:100-109`).
- **`UsersAndCollections`** (`:151-164`): composite PK `[userId, collectionId]` — one grant per pair by construction; capability booleans; `onDelete: Cascade` both ways (removing a user silently removes grants — audit-log absence noted).
- **Two "who made this" fields**: `Collection.ownerId` vs `createdById` (`:137-141`), `Link.createdById` (`:173-174`) — ownership ≠ authorship in shared collections. Quota (`MAX_LINKS_PER_USER`) is charged to the collection **owner** (`archives/[linkId].ts:41-55`).
- **Tag uniqueness `@@unique([name, ownerId])`** (`:216`) — enables `connectOrCreate` upserts (`postLink.ts:120-137`); tag namespace is per *owner*.
- **Nullable format columns as a state machine** on Link (`:182-193`): null = not yet, `"unavailable"` = tried/declined, path string = done; plus `lastPreserved` (queue marker) and `indexVersion` (search sync marker). The schema encodes *workflow*, not just data — cheap and queryable, at the cost of magic strings.
- **Cascades everywhere** (`Link:175`, `Collection:134-137`): deleting a collection deletes sub-collections and links transitively. Convenient; also means one wrong delete has a big blast radius, and *files* aren't part of the cascade (orphaned archives — cleanup handled in code, `deleteCollectionById`/`removeFolder`, not by the DB).

## Indexes and query alignment

Declared: `Collection@@index([ownerId])` (`:148`), `Link@@index([collectionId])` (`:197`), `Tag@@index([ownerId])`, join-table PK + `@@index([userId])`. These match the hot paths (links-by-collection, collections-by-owner). Not indexed: `Link.url` (duplicate check `postLink.ts:53-60` scans owner's links via join — fine small, watch at 100k), `Link.lastPreserved` (the worker's queue predicate — *investigate*: partial index `WHERE lastPreserved IS NULL` would be the classic optimization), `Link.createdById`.

## Transactions and consistency expectations

Prisma default: each query is its own transaction. Multi-statement writes here are mostly **unwrapped** — inventory: link create + image-path update (`postLink.ts:104-162`), collection create + collectionOrder push (`setCollection.ts:57-75`), file delete → DB update → file move (`updateLinkById.ts:139-197`). Analysis discipline (this is the interview skill): for each, ask *what does the crash-between state look like, and does anything reconcile it?* Files can't join DB transactions, so DB-vs-file consistency is **eventual with code-level compensation** (re-archive heals broken paths). See [08/03 Q12](../08-interview-prep/03-api-and-data-modeling-questions.md).

Where a transaction is genuinely warranted and cheap: the two-statement create in `postLink` (both DB). Where it isn't: anything spanning Playwright/files (long-held connections, no atomicity possible anyway).

## How to change this schema safely (the procedure)

1. Additive first: new nullable column or table; `yarn prisma:dev` generates the migration; commit schema + migration together.
2. Deploy order awareness: web and worker share the client but deploy separately (Docker) — old worker + new column must coexist ⇒ avoid `NOT NULL` without default, avoid renames in one step (expand → backfill → contract; rename walkthrough in [08/03 Q4](../08-interview-prep/03-api-and-data-modeling-questions.md)).
3. Data backfills go in the **AppMigration** worker system (`schema.prisma:296-308`, `apps/worker/workers/migrationWorker.ts` — runs before loops start, `worker.ts:12`), not in DDL migrations. Status enum APPLIED/PENDING/FAILED gives you observability on backfills — a genuinely nice pattern; note it for interviews.
4. Rollback story: Prisma has no down migrations by default ⇒ policy is roll-forward. Write the compensating migration *before* you need it for risky changes.
5. Search coupling: if the change affects indexed fields, bump `MEILI_INDEX_VERSION` (`packages/lib/constants.ts`) to trigger global reindex (`linkIndexing.ts:79-81`) — schema changes here have **two** consumers to migrate.

Drill: design the migration for "links can belong to multiple collections." Which invariants break (quota-by-owner? file paths keyed by collectionId — `resolveAccessibleArchive.ts:70-72`)? Self-grade — Strong: you found that file paths embed collectionId, so the storage layout itself assumes the 1:n relationship; that's the kind of hidden coupling schema changes must surface.
