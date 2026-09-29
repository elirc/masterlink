# Performance Thinking

Rule zero: **measure first**. Every hotspot below is a *hypothesis with evidence*, ranked by suspicion — the workflow is: reproduce load → profile → fix the top item → re-measure. Guessing-then-optimizing is the junior tell; this file is practice in forming ranked hypotheses.

## Performance domains in this system

| Domain | Where | Measurement tool |
| --- | --- | --- |
| Server/API latency | API routes + controllers | timing middleware (absent — add locally), Prisma query logs (`DEBUG=true`, `packages/prisma/index.ts`) |
| DB query shape | controllers, worker scheduler | `EXPLAIN ANALYZE`, prisma query log |
| Render performance | LinkViews lists, dashboard | React DevTools Profiler |
| Network/payload | list endpoints, archive files | devtools; `responseLimit` 50MB ceiling (`[linkId].ts:20`) |
| Worker throughput | archiving/indexing loops | batch logs ("N left") + wall-clock |
| Memory | worker (browser+buffers) | `process.memoryUsage`, heap snapshots (Scenario 5 in [03](03-systematic-debugging.md)) |
| Bundle | web client | `next build` output, bundle analyzer |

## Ranked hotspot hypotheses (with anchors)

1. **Per-request auth queries** — every private call pays revocation lookup + user load (`verifyToken.ts:24-33`, `verifyUser.ts:27-35`): 2 queries before any work. At self-host scale: irrelevant. At cloud scale: the first thing to cache (short-TTL, keyed by jti). Evidence to gather: p50 route timing with/without.
2. **Title fetch on the write path** — `postLink.ts:78-79` blocks the user on an arbitrary website's response time. Measure p95 of `POST /links`; the fix (defer to worker) is [Ticket M2](../06-contribution-practice/02-mid-level-feature-tickets.md).
3. **Scheduler round-trips** — O(users) queries per tick ×2 phases (`getLinkBatchFairly.ts:87-96,122-128`). Only matters with many concurrent-eligible users; the window-function rewrite is [08/03 Q10](../08-interview-prep/03-api-and-data-modeling-questions.md).
4. **Missing partial index on the queue predicate** — `WHERE lastPreserved IS NULL` scans (`Link` has no index on it; `schema.prisma:166-198`). At 1M links the eligibility query degrades; `CREATE INDEX ... WHERE "lastPreserved" IS NULL` is the textbook fix. *Hypothesis — EXPLAIN first.*
5. **Prisma `contains` search fallback** — un-indexed ILIKE across name/url/description/tags (`getLinks.ts:23-69`) — fine to ~10k links, then Meili exists for a reason. Know which mode an instance runs.
6. **List over-fetching** — search returns full rows + tags + collection per link (`searchLinks.ts:95-120`+includes); page size `PAGINATION_TAKE_COUNT` default 50. Payload measurement before judging.
7. **Whole-file buffering** — archive serving reads entire file to memory (`[linkId].ts:122-127`); concurrency × size math in [08/01 Q10](../08-interview-prep/01-js-ts-node-deep-dive.md).
8. **Render: unvirtualized lists** — `LinkViews/Links.tsx` renders all loaded pages; with infinite scroll the DOM grows unbounded. Profile at 500+ links before reaching for virtualization (*investigate: whether any windowing already exists*).

## How to find each class (transferable)

N+1: prisma query log, look for repeated identical shapes with different params. Serial-await chains: read controllers for sequential awaits with no data dependency (e.g. `postLink` runs duplicate-check → capacity → title fetch serially; the first two could parallelize — micro, but the *reading skill* scales). Expensive renders: Profiler flame widths, not vibes. Oversized bundles: build output per-page first-load JS. Missing indexes: `EXPLAIN` anything in a loop or a poll.

Drill: pick hypothesis 4. Write the exact `EXPLAIN ANALYZE` you'd run, predict the plan (seq scan), write the `CREATE INDEX` migration, and state the write-amplification cost you're accepting. Self-grade — Strong: you also noted the index is partial *and* that the worker's ORDER BY (`createdAt desc`) may want to be in it.
