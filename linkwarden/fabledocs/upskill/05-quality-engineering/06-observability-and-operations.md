# Observability and Operations

**Observability** = can you answer "what is it doing / why is it wrong" from the outside, in production. This repo's current answer is mostly *console logs + inspectable DB rows*. That's a real (minimal) strategy — the DB-as-queue design makes state queryable by construction — but there are no metrics, no tracing, no error tracker.

## What exists

| Signal | Where | Notes |
| --- | --- | --- |
| Worker batch logs | `linkProcessing.ts:76-81` ("Processed N, M left"), `linkIndexing.ts:170-175`, per-feed RSS errors (`rssPolling.ts:28-34`) | queue depth in every log line — the one metric that matters, already computed (`countUnprocessedBillableLinks`) |
| Worker stats endpoint | `pages/api/v1/worker/index.ts` → `getWorkerStats.ts` | a machine-readable health surface exists — extend, don't invent |
| Prisma query logging | `DEBUG=true` (`packages/prisma/index.ts`) | dev diagnosis |
| App migration status | `AppMigration` rows APPLIED/PENDING/FAILED (`schema.prisma:296-308`) | backfill observability done right |
| Supervisor restarts | `apps/worker/index.ts:6-10` logs exit code/signal | restarts are visible but not counted/alerted |
| Deploy/rollback | Docker images (`release-container.yml`); migrate-then-start ordering (`playwright-tests.yml` mirrors it) | rollback = previous image; DB migrations roll forward only |

## "How would I know this broke?" per major flow

| Flow | Breakage | Today's detection | Cheapest improvement |
| --- | --- | --- | --- |
| Add link (Flow 1) | 500s / slow title fetch | user reports | route timing + error log with request id |
| Auth (Flow 2) | login failures / revocation misfires | user reports; e2e `@login` in CI | auth failure counter by reason |
| Archiving (Flow 3) | queue grows silently | `SELECT count WHERE lastPreserved IS NULL` (manual); worker logs if you're watching | expose queue depth + oldest-pending-age on the existing stats route; alert on age > 1h |
| Search (Flow 4) | stale/missing results | "N left" indexing log | DB-vs-index count reconciliation on the stats route |
| File serving (Flow 5) | 401s/missing files | user reports | 4xx/5xx counters per route |
| RSS (Flow 7) | feed silently failing | per-feed error log | `lastBuildDate` staleness query (already in schema — `RssSubscription:252`) |

The pattern in the right column: **this system's observability upgrades are mostly SQL** — because state lives in rows, "is it healthy" is a query. Frame it that way in interviews; it shows you match the observability strategy to the architecture instead of reciting "add Datadog."

## Error handling as an operations concern

Errors are strings to users (`{response: "..."}`) and `console.log` to operators (`archiveHandler.ts:199-202`). Missing: aggregation (Sentry-shape), request ids for correlation, and structured logs (the color-coded `\x1b[34m` console strings are for humans watching a terminal — fine for self-host, insufficient for cloud). The 3-month plan's step 5 ([critique](../03-architecture-and-patterns/06-architecture-critique.md)) sequences this.

Drill: write the five SQL statements that would constitute a "morning health check" dashboard for a self-hosted instance (queue depth, oldest pending, failure-marked links last 24h [`"unavailable"` set recently], stale feeds, index lag). Self-grade — Strong: each query runs against real columns you can name without opening the schema.
