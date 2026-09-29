# Domain Glossary

Product nouns first, then confusables. Every term anchored to its home in code.

| Term | Meaning | Code home |
| --- | --- | --- |
| **Link** | a saved bookmark/upload; the core entity | `schema.prisma:166-198` |
| **Collection** | folder of links; unit of sharing/ownership; can nest (`parentId`) and be public (`isPublic`) | `schema.prisma:126-149` |
| **Member** | a user granted access to someone else's collection with `canCreate/canUpdate/canDelete` flags | `UsersAndCollections`, `schema.prisma:151-164` |
| **Unorganized** | the implicit default collection, created on demand | `setCollection.ts:42-60` |
| **Preservation / archiving** | capturing a link as files: screenshot, PDF, monolith, readable, preview | `apps/worker/lib/archiveHandler.ts` |
| **Monolith** | single-file HTML snapshot of a page (name comes from the CLI tool) | `apps/worker/lib/preservationScheme/handleMonolith.ts` |
| **Readable** | Mozilla Readability extraction (article text) | `handleReadability.ts`; user font prefs on `User:57-60` |
| **Preview** | small image used on link cards | `handleArchivePreview.ts`, `packages/lib/generatePreview.ts` |
| **`"unavailable"`** | magic string in a format column: preservation was attempted and this format wasn't produced | written at `archiveHandler.ts:213-224`, `postLink.ts:138-148` |
| **`lastPreserved`** | timestamp; null = waiting for the worker (the queue predicate) | `getLinkBatchFairly.ts:35-38` |
| **Archival tag** | a Tag carrying per-tag archive settings that override the user's defaults | `Tag:206-211`, `packages/lib/isArchivalTag.ts`, applied `archiveHandler.ts:85-107` |
| **AI tagging** | LLM-generated/matched tags; methods DISABLED/GENERATE/EXISTING/PREDEFINED | enum `schema.prisma:83-88`, `apps/worker/lib/autoTagLink.ts` |
| **Pinned link** | user-specific favorite (many-to-many `pinnedBy`) shown on dashboard | `schema.prisma:172`, pin logic `updateLinkById.ts:36-65` |
| **Highlight** | user-selected text range (+comment) on a readable view | `schema.prisma:260-273` |
| **Dashboard section** | configurable dashboard blocks (stats/recent/pinned/collection) | `schema.prisma:275-294` |
| **RSS subscription** | feed polled hourly into a collection | `schema.prisma:248-258`, `rssPolling.ts` |
| **Access token** | API key *or* revocable session — same table, `isSession` flag | `schema.prisma:234-246` |
| **Whitelisted user** | usernames allowed to register (closed-instance mode) | `schema.prisma:99-106` |
| **Subscription / parent subscription** | Stripe billing; parent = seat-based team owner | `schema.prisma:220-232`, `User:38-39` |
| **Index version** | per-link marker of which Meili schema version indexed it; bump constant → global reindex | `linkIndexing.ts:79-81`, `packages/lib/constants.ts` |
| **App migration** | worker-run data migrations tracked in DB (distinct from Prisma migrations!) | `schema.prisma:296-308`, `apps/worker/workers/migrationWorker.ts` |
| **Demo mode** | env-flagged read-only deployment | `NEXT_PUBLIC_DEMO` guards, e.g. `links/index.ts:33-37` |
| **User content domain** | separate origin for serving preserved HTML | `createPreservedFormatUrl.ts`, `[linkId].ts:93-102` |

## Confusables

- **Prisma migrations** (`packages/prisma/migrations/`, schema DDL) vs **AppMigration** (worker-run data backfills). Different mechanisms, both called "migration."
- **`getLinks`** (deprecated Prisma-only list, `controllers/links/getLinks.ts:5-10` can be disabled by env) vs **`searchLinks`** (the live path — the UI's "list links" actually calls `/api/v1/search`, see `links.tsx:75-77`).
- **owner** vs **createdBy** on Collection/Link: ownership can differ from authorship in shared collections (`schema.prisma:137-141, 173-174`). Tags always belong to the collection **owner** (`postLink.ts:120-137`).
- **AccessToken-as-session** vs **AccessToken-as-API-key**: same table; revocation checks apply to both (`verifyToken.ts:24-33`).
- **preview** (card image) vs **image** (full screenshot format). Both are Link columns.
- **router** here means `packages/router` (React Query hooks), *not* Next.js routing.
