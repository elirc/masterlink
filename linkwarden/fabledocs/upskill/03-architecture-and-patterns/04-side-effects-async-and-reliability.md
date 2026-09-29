# Side Effects, Async, and Reliability

A **side effect** is anything a request/job does beyond returning a value: writes, emails, external calls. Reliability engineering is deciding, per effect: *what happens when it half-happens?*

## Side-effect inventory

| Effect | Trigger | Where | Retry story | Failure visibility |
| --- | --- | --- | --- | --- |
| Preservation files (screenshot/pdf/monolith/readable/preview) | worker loop | `archiveHandler.ts:109-198` | none — failure marked done (`:203-224`) | worker logs only |
| Meilisearch index writes | worker loop | `linkIndexing.ts:145-159` | implicit — row stays stale-versioned until success | "N left" logs |
| Wayback Machine submission | during archive | `archiveHandler.ts:118-120` (no await) | none, fire-and-forget | none (deliberate) |
| Title/header fetch | **on request path** | `postLink.ts:78-79` | none; degrades to url-as-name | user sees odd name |
| Emails (verify/invite/reset; trial-end) | request path + worker loop | `apps/web/lib/api/send*.ts`; `trialEndEmailWorker.ts` | *investigate per callsite* | `trialEndEmailSent` flag on User (`schema.prisma:72`) prevents re-send = idempotency marker |
| Stripe seat updates | user CRUD | `apps/web/lib/api/stripe/updateSeats.ts` | *investigate* | — |
| File delete/move on link edits | request path | `updateLinkById.ts:139,196` | none; DB and FS can diverge | broken preview until re-archive |
| RSS-created links | worker loop | `rssPolling.ts` → `rssHandler.ts` | next poll retries naturally (lastBuildDate gate) | error log per feed |
| AI tagging | worker loop | `autoTagPreservedLinks.ts` → `autoTagLink.ts` | `aiTagged` flag = one shot | logs |

## The reliability toolkit, as used here

- **Idempotency markers in rows**: `lastPreserved`, `indexVersion`, `aiTagged`, `trialEndEmailSent` — one column per effect, checked before acting. This is the repo's core reliability idea and it generalizes: *make the effect's completion queryable.*
- **Timeouts**: browser work raced against `BROWSER_TIMEOUT` (`archiveHandler.ts:63-75,110-198`); redirect count capped (`safeFetch.ts:103-124`). Missing: explicit timeout on `postLink`'s title fetch (*investigate `fetchTitleAndHeaders` internals — `apps/web/lib/shared/fetchTitleAndHeaders.ts`*).
- **Bulkheads**: per-link try/catch (`linkProcessing.ts:58-68`), per-feed try/catch (`rssPolling.ts:24-35`), `Promise.allSettled` batches — one bad item never sinks the batch.
- **Supervision + rotation**: process respawn (`worker/index.ts:3-16`), 30-min browser recycle (`linkProcessing.ts:29-33`).
- **Backpressure (partial)**: batch sizes capped (`ARCHIVE_TAKE_COUNT`, `INDEX_TAKE_COUNT`); RSS fan-out *not* capped (unbounded `Promise.all`, `rssPolling.ts:19` — flagged risk).
- **Not present**: outbox/transactional messaging (unneeded — DB is already the queue), retry-with-backoff, dead-letter states, distributed locks (single-worker assumption — the load-bearing constraint of the whole design).

## Concepts defined against this code (interview vocabulary)

**Idempotency**: `archiveHandler` re-run on a half-done link skips completed formats (`if (!link.image)` guards, `:122-194`) — safe to repeat. **At-least-once vs at-most-once**: indexing is at-least-once (mark *after* success); archiving is at-most-once (mark even on failure). Every async system picks one per effect; this repo demonstrates both and *why* — duplicate indexing is harmless, duplicate archiving of a poison URL is not. **Compensation**: link deleted mid-archive → `finally` removes the just-written files (`archiveHandler.ts:225-227`). **Outbox**: not used; describe it as the upgrade path if web and worker ever stop sharing a DB.

## Side effects in risky places (flags)

1. `postLink.ts:78-79` — external fetch on the user's critical path (latency + partial SSRF surface even with the guard). Move-to-worker is a clean mid-level ticket ([06/02](../06-contribution-practice/02-mid-level-feature-tickets.md) Ticket M2).
2. `updateLinkById.ts:139` — file deletion *before* validation completes fully and before the DB write; wrong-order destructive effect (see [08/03 Q12](../08-interview-prep/03-api-and-data-modeling-questions.md)).
3. Wayback fire-and-forget with no `.catch` (`archiveHandler.ts:119`) — unhandled rejection risk depending on `sendToWayback` internals; *investigate*.

Drill: pick one row of the inventory and write its "half-happened" postmortem: what state is visible, who notices, what heals it, in ≤5 sentences. Self-grade — Strong: your answer names the marker column and whether the effect is at-least-once or at-most-once.
