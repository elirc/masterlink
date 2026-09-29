# API & Data-Modeling Cards

12 cards anchored to the real API and schema.

## Q1: Where should validation live in an API, and what belongs at each layer?

Round: API/data
What it's really testing: boundary discipline.
Repo anchor: four layers in one flow — client zod (`NewLinkModal.tsx:88-96`, UX), server zod (`postLink.ts:16-25`, the contract), business rules (duplicates `:47-67`, capacity `:69-76`), DB constraints (`@@unique[name, ownerId]` on Tag, `schema.prisma:216`).
Junior: "validate in the controller."
Mid: names all four and the rule — shape at the edge, *rules* in domain logic, *invariants* in the DB (last line of defense against every future code path).
Senior: notes what happens when layers disagree (DB unique violation surfacing as a 500 because code assumed the check above caught it — race between duplicate-check and insert in `postLink.ts:53-67` is exactly that TOCTOU); fix: rely on the constraint, catch the violation.
Likely follow-ups: where do you check authorization relative to validation? (After authn, before side effects; this repo validates shape first — defensible either way, know your order.)
Practice drill: label each check in `postLink.ts` with its layer, from memory.

## Q2: Design the "save a bookmark" endpoint. What does yours return and when?

Round: API/data
What it's really testing: async contract design.
Repo anchor: `POST /api/v1/links` returns the created row immediately (`postLink.ts:166`); preservation happens later (worker); the client learns completion by refetching (formats flip from null/`"unavailable"`).
Junior: returns 200 with the link.
Mid: identifies the implicit state machine in nullable columns and proposes making it explicit for clients (a `status: pending|done|failed` derived field) — plus 201 + Location as the REST-correct shape.
Senior: discusses notification options (polling vs SSE/websocket vs no-op) against actual user needs; idempotency of the POST (retried request = duplicate link unless `preventDuplicateLinks`; an `Idempotency-Key` header is the general fix).
Likely follow-ups: what status code while archiving is in flight? How does the UI show failure?
Practice drill: write the response body JSON you'd return, then diff it against the real one in devtools.

## Q3: Cursor vs offset pagination — this codebase uses both. Critique that.

Round: API/data
What it's really testing: data-consistency reasoning under writes.
Repo anchor: Prisma branch: true cursor (`getLinks.ts:93-96`, `cursor: {id}, skip: 1`); Meili branch: same `cursor` param used as **offset** (`searchLinks.ts:64-65`); client just forwards `nextCursor` (`links.tsx:99-101`).
Junior: defines both.
Mid: explains why cursors win for feeds (stable under concurrent inserts, no deep-scan cost) and spots that one query param silently means two things — a contract smell even if behavior is "close enough."
Senior: failure mode concretely — in offset mode, a link created mid-scroll shifts every subsequent page by one (duplicate row rendered); Meili can't cursor natively, so the honest options are: accept drift, search-after tokens, or cap search depth. Choosing *and naming the cost* is the senior move.
Likely follow-ups: how do you paginate with a sort on mutable fields (name)?
Practice drill: write the API doc paragraph for `cursor` that tells the truth about both modes.

## Q4: How do you evolve a database schema safely? This repo has 93 migrations — what's the discipline?

Round: API/data
What it's really testing: migration/rollback maturity.
Repo anchor: `packages/prisma/migrations/` (93 dirs, additive names like `20231031100017_add_last_preserved_field`); applied via `yarn prisma:deploy` in CI before boot (`playwright-tests.yml`); plus the separate **AppMigration** worker for data backfills (`schema.prisma:296-308`, runs before loops start — `worker.ts:12`).
Junior: "run migrate."
Mid: expand→migrate→contract for breaking changes; additive columns with defaults are the safe common case (which is most of these 93); schema migrations vs data migrations as different beasts (this repo separates them!).
Senior: deploy-order reasoning — old code must tolerate the new schema during rollout; rollback = *forward* fixes usually (down-migrations lie once data has flowed); notes Prisma has no down-migrations by default, so this repo's implicit policy is roll-forward — say that in interviews.
Likely follow-ups: rename a column with zero downtime, step by step.
Practice drill: write the expand/contract plan for renaming `Link.image` → `Link.screenshot`.

## Q5: Authn vs authz — separate them in an API you know.

Round: API/data
What it's really testing: the most common security confusion.
Repo anchor: authn = `verifyUser.ts` (who are you: JWT + revocation + account state); authz = per-resource: `getPermission.ts` (collection relationship), `setCollection.ts:25-36` (create rights), `resolveAccessibleArchive.ts:30-45` (owner OR member OR public).
Junior: uses the words interchangeably — instant flag.
Mid: crisp split + the repo's shape: authn centralized in a gate, authz distributed per controller; every list query re-states the ownership filter (`getLinks.ts:100-109`).
Senior: the risk of distributed authz — one forgotten where-clause = IDOR; mitigations: resolver helpers, tests per endpoint (the archives tests do exactly this), or row-level security in Postgres as the systemic fix; also flags 401-vs-403 misuse here (`updateLinkById.ts:92-101` returns 401 for permission denials).
Likely follow-ups: design the test that proves tenant isolation.
Practice drill: build the owner/member/stranger × read/write matrix for links, with expected codes, then check it against the code.

## Q6: How would you serve private user files (images/PDFs/HTML)? 

Round: API/data
What it's really testing: a complete secure-download design.
Repo anchor: the full pattern — authz query first (`resolveAccessibleArchive.ts:30-45`), file path *derived from DB ids* never user strings (`:70-72` — path traversal dead by construction), `Cache-Control: private, immutable` (`[linkId].ts:125`), and HTML quarantined to a separate origin via 5-min scoped JWT (`createPreservedFormatUrl.ts:46-84`, enforcement `[linkId].ts:93-102`).
Junior: "check the user then send the file."
Mid: adds derived paths, private caching, content-type care, and why HTML is special (stored XSS on your origin).
Senior: compares to S3 presigned URLs (offload bandwidth + authz snapshot at URL-mint time), and articulates token design: scope claim, resource claims, short TTL, strict shape validation after signature check (`createPreservedFormatUrl.ts:26-44`).
Likely follow-ups: what if the user is removed from the collection 2 minutes after minting a 5-minute URL? (Access persists until expiry — TTL *is* the revocation latency; that's the tradeoff to name.)
Practice drill: whiteboard both variants (proxy-through-API vs presigned) with the failure modes of each.

## Q7: Model "share this folder with edit rights" — walk me through your tables.

Round: API/data
What it's really testing: relational modeling of permissions.
Repo anchor: `UsersAndCollections` (`schema.prisma:151-164`) — composite PK `[userId, collectionId]`, three capability booleans; consumed in `updateLinkById.ts:72-101` (including the rule that only owners move links across collections `:88-96`).
Junior: `shared_with` array column (flag: arrays of FKs).
Mid: proper join table with capability flags, composite PK prevents duplicate grants, `onDelete: Cascade` cleans up; walks a permission check query.
Senior: growth analysis — booleans → role enum → policy table; inheritance question (sub-collections `parentId` exist here, but membership does **not** inherit — *investigate/confirm*, and what UX bug that implies); auditability (who granted what when — `createdAt` on the row helps).
Likely follow-ups: add "viewer" role; add expiring shares.
Practice drill: write the SQL for "all links user 42 can see" and compare with `getLinks.ts:97-128`.

## Q8: Your search index and your database disagree. How, and what do you do?

Round: API/data
What it's really testing: dual-write consistency maturity.
Repo anchor: writes go DB-first, index catches up via the `indexVersion` polling loop (`linkIndexing.ts:73-159`); reads re-verify against the DB (`searchLinks.ts:95-120`); global reindex = bump `MEILI_INDEX_VERSION` constant.
Junior: "keep them in sync with events."
Mid: names the actual consistency model — eventual, single-writer, safe-direction failure (stale index hides, never leaks, *because* of the DB re-check) — and the reindex protocol.
Senior: enumerates the drift cases — membership change doesn't touch `indexVersion` (denormalized `collectionMemberIds` go stale until that link is otherwise reindexed — the DB re-check is the only guard), deletes (does the index get told? *investigate* `deleteLinksById`), and proposes: transactional outbox if this were event-driven, or accept-and-document since reads are guarded.
Likely follow-ups: how would you verify sync in production? (Sampled reconciliation job.)
Practice drill: find where link deletion updates/should update Meili; report what you find.

## Q9: What makes an API idempotent, and where does it matter in this system?

Round: API/data
What it's really testing: retry-safety vocabulary in practice.
Repo anchor: the archive job is idempotent-by-marker (`lastPreserved` + per-format `if (!link.image)` guards, `archiveHandler.ts:122-194` — a crashed job re-done skips finished formats); `connectOrCreate` for tags (`postLink.ts:120-137`) is an idempotent upsert; plain `POST /links` is *not* idempotent (retry = duplicate).
Junior: "same request twice = same result."
Mid: separates client-retry idempotency (needs keys or natural uniqueness) from worker-restart idempotency (needs state markers), with the anchors above.
Senior: designs the missing piece — `Idempotency-Key` on POST with a keyed-response table; and points out `updateMany`-then-process patterns where partial failure between "mark" and "do" leaves limbo (the `lastPickedAt` stamp at `getLinkBatchFairly.ts:159-163` is at-most-once bookkeeping — harmless here, and *why* it's harmless is the answer).
Likely follow-ups: idempotency vs exactly-once delivery.
Practice drill: classify five endpoints (POST /links, PUT /links, DELETE /links, POST /archives, GET /search) by retry safety.

## Q10: Find the N+1 risk in a batch job you've read, and fix it.

Round: API/data / performance
What it's really testing: query-shape awareness.
Repo anchor: `getLinkBatchFairly.ts:87-96` — a `prisma.link.findMany` **per user** inside a loop (then more per round at `:122-128`). With 100 eligible users that's 100+ queries per 10-second tick.
Junior: doesn't see it without prompting.
Mid: spots the loop, proposes one grouped query (fetch candidate links for all users, group in memory) or a window-function query (`ROW_NUMBER() OVER (PARTITION BY userId ORDER BY createdAt DESC)`).
Senior: measures first (is it actually hot? batch is small, tick is 10s — maybe fine), names the invariant to preserve (per-user fairness ordering), writes the SQL, and knows Prisma needs `$queryRaw` for window functions — a worked example of "ORM until it isn't."
Likely follow-ups: how would you catch N+1 in review/CI? (Query logging in dev, `prisma:query` events.)
Practice drill: write the window-function SQL that replaces the loop.

## Q11: Rate limiting — this API has none. Design it.

Round: API/data
What it's really testing: can you add a cross-cutting concern to an existing system.
Repo anchor: conceptual — no rate-limit middleware exists (verified by absence in `apps/web/lib/api/` and route files); sensitive targets: auth endpoints (`pages/api/v1/auth/*`, brute force), `POST /links` (quota bypass attempts are already handled by `verifyCapacity`, but request-rate isn't), public routes.
Junior: "use a library."
Mid: token bucket per key (IP for anonymous, userId for authed), 429 + Retry-After, and where it slots in this codebase (the `verifyUser` gate is the natural interception point — every private route already funnels through it).
Senior: storage tradeoff (in-memory per-instance vs Redis for multi-instance — this app may run single-instance, say so and choose simple), what *not* to limit (archive file GETs used by the UI), and abuse-vs-UX tuning with observability first (log would-be-429s before enforcing).
Likely follow-ups: distributed rate limiting; per-plan limits.
Practice drill: write the middleware signature and the 3 config knobs you'd ship with.

## Q12: When do you use a transaction? Audit a real write path for missing ones.

Round: API/data
What it's really testing: consistency-boundary judgment.
Repo anchor: `postLink.ts:104-162` — link create, then a *second* update for the image path, then folder creation: three steps, no `prisma.$transaction`. `updateLinkById.ts:139-197` — file deletion before DB update, file move after. `setCollection.ts:57-75` — collection create + user collectionOrder update, unwrapped.
Junior: "wrap everything in transactions."
Mid: transactions bound *database* atomicity only — the file operations can't join, so the real question is crash-ordering: which partial states are acceptable? Orphan file = tolerable; DB row pointing at missing file = renders broken preview (also tolerable, self-heals on re-archive). That analysis, not blanket wrapping, is the answer.
Senior: names the pattern for when it's *not* tolerable (write DB intent first, do side effect, confirm — i.e., outbox/saga thinking), and flags long transactions holding locks as the cost of over-wrapping; suggests the one genuinely worth wrapping: create-link + image-path update (`postLink.ts:104-162`, pure DB, two statements).
Likely follow-ups: isolation levels; what does Prisma's interactive transaction actually hold open?
Practice drill: for each of the three anchors, write the crash-between-steps state and label it tolerable/not.
