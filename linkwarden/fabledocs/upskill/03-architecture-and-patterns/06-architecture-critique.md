# Architecture Critique

An honest design review, as if you owned this codebase for the next 3 months. Doubles as system-design interview material ([08/04](../08-interview-prep/04-system-design-from-this-repo.md)). Confirmed observations carry anchors; everything else is labeled hypothesis.

## Strongest design choices (defend these in interviews)

1. **DB-as-queue with row-state markers** — zero extra infra for self-hosters, fully inspectable, idempotent by column (`getLinkBatchFairly.ts:35-38`, `archiveHandler.ts:203-230`). Right-sized for the deployment reality.
2. **The SSRF stack** (`ssrf.ts` + `safeFetch.ts` + worker re-check) — genuinely production-grade: DNS pinning, redirect re-validation, TOCTOU awareness, *and tests*. Most codebases at this scale have a hostname blocklist and a prayer.
3. **Index finds, database decides** (`searchLinks.ts:67-120`) — stale search can hide but never leak. The safest possible failure direction, chosen deliberately.
4. **Separate content origin + scoped 5-min tokens for preserved HTML** (`createPreservedFormatUrl.ts`, `[linkId].ts:93-102`) — stored-XSS blast radius contained at the origin boundary.
5. **Shared hooks package with platform injection** (`packages/router/`) — one data layer for web+mobile with honest seams (`links.tsx:383-393`).
6. **Optional infrastructure via null clients** (`meilisearchClient.ts:1-10`, `s3Client.ts`) — graceful degradation as a first-class design stance; the product works with just Postgres.
7. **AppMigration worker for data backfills with status tracking** (`schema.prisma:296-308`) — separates DDL from data migration; most teams learn this one the hard way.

## Risks and tradeoffs (prioritized, with evidence)

| # | Risk | Evidence | Impact | Confidence |
| --- | --- | --- | --- | --- |
| 1 | **Test coverage inverted vs risk**: worker (highest blast radius) has zero tests; CI enforces only a login e2e | `find` results in [verification log](../09-reference/verification-log.md); `playwright-tests.yml:45` | regressions ship silently | confirmed |
| 2 | **Distributed authz with a misuse-prone primitive**: `getPermission({linkId})` ignores userId; every caller must remember the check | `getPermission.ts:14-26` | one forgotten check = IDOR | confirmed (as design; no exploit claimed) |
| 3 | **Duplicated auth gates drifting** | `verifyUser.ts:60-70` vs `isAuthenticatedRequest.ts:44-46` (subscription semantics differ) | inconsistent access decisions | confirmed divergence; impact hypothesis |
| 4 | **No retry semantics for archiving**; failure = permanent | `archiveHandler.ts:203-224` | transient network blip ⇒ link never preserved without manual re-trigger | confirmed behavior; severity depends on ops |
| 5 | **Single-worker assumption unstated**: no locking on batch pick | `getLinkBatchFairly.ts` (no FOR UPDATE/SKIP LOCKED) | scaling out double-processes | confirmed by absence |
| 6 | **Per-request DB revocation check** | `verifyToken.ts:24-33` | latency floor + DB load at scale | confirmed; cost hypothesis |
| 7 | **Error contract is prose strings** (`{response: "..."}`), some translated client-side by key | `postLink.ts:18-25`, `links.tsx:522` | API consumers can't program against errors; copy changes break clients | confirmed |
| 8 | **Unbounded fan-outs**: RSS `Promise.all`; N+1-ish scheduler queries | `rssPolling.ts:19-36`; `getLinkBatchFairly.ts:87-96,122-128` | large-instance pain | confirmed shape; scale threshold unknown |
| 9 | **Query-param coercion unvalidated (NaN)** | `links/index.ts:15-28` | benign today (falls to defaults); latent | confirmed |
| 10 | 401-for-403 convention | `updateLinkById.ts:92-101` | client UX / debuggability | confirmed |

## "What I'd change owning this for 3 months" (in order)

1. **Week 1–2: make CI mean something.** Unit-test job + worker typecheck job ([Ticket 1](../06-contribution-practice/01-good-first-tickets.md)); characterization tests for `getLinkBatchFairly` and `searchQueryBuilder`. *Migration path:* pure extraction where needed; no product code changes. *Test strategy:* the tests ARE the deliverable.
2. **Week 3–4: unify the auth gate.** Single `authenticateRequest()` returning a typed result; `verifyUser` becomes a thin HTTP adapter over it; delete the drift (risk 3). *Migration:* adapter keeps route signatures identical; regression = auth e2e + the archives test suite. *Rollback:* revert one module.
3. **Week 5–6: bounded retries for archiving.** Add `archiveAttempts Int @default(0)` + `lastArchiveError String?`; eligibility becomes `lastPreserved IS NULL AND attempts < 3`; failure increments instead of stamping done; keep the stamp after attempt 3 (preserves the poison-pill defense). *Migration:* additive column, old workers coexist. *Test:* unit on eligibility where-clause; manual poison-URL soak.
4. **Week 7–8: error codes.** `{code: "COLLECTION_NOT_ACCESSIBLE", message}` alongside existing strings (additive, non-breaking); client switches on code, falls back to message. Fixes risk 7 and half of risk 10's pain without breaking the API.
5. **Week 9–12: observability floor.** Boot config summary, queue-depth + failure counters (a `/api/v1/worker` stats route already exists — `getWorkerStats.ts` — extend it), memory gauge per batch. Then — and only then — decide whether risks 6/8 are real at observed load.

Deliberately **not** doing: app-router migration, queue library adoption, tRPC rewrite — each is a big-bang risk whose problem I haven't measured yet. Saying *what you'd defer and why* is the strongest senior signal in a design review.

Hypotheses to validate before acting (listed so they don't masquerade as facts): Meili result ordering lost on Prisma re-fetch; membership-change staleness window size; title-fetch p95 latency on `postLink`; whether deletes propagate to the search index.
