# Senior Build Projects

Six projects, 2 days–4 weeks. Each is realistic for this product (things a maintainer plausibly wants) and exercises the full senior planning surface. Do at most two; do them completely.

---

## P1: Multi-worker safe archiving (1-2 weeks)
Problem: the pipeline assumes exactly one worker ([critique risk 5](../03-architecture-and-patterns/06-architecture-critique.md)); scaling out double-processes links.
Product value: cloud/self-host horizontal scaling; faster bulk imports.
Design checklist: claim semantics (`FOR UPDATE SKIP LOCKED` via `$queryRaw`, or a `claimedAt/claimedBy` column with expiry — compare both in your note); fairness preserved across workers; browser-per-worker capacity math; crash-mid-claim recovery (claim expiry).
Architecture decisions: keep DB-as-queue (don't introduce Redis — justify); claim TTL vs heartbeat.
Likely files: `getLinkBatchFairly.ts` (claim query), `archiveHandler.ts` (release/expiry), schema (+2 columns), worker env (`WORKER_ID`).
Migration plan: additive columns; old single worker ignores them; enable claiming behind env flag.
Test plan: two workers against one test DB, assert disjoint batches; crash-simulation (kill mid-claim, assert reclaim after TTL).
Security plan: none new. Performance plan: EXPLAIN the claim query; partial index from [05/04 hypothesis 4](../05-quality-engineering/04-performance-thinking.md).
Rollout/rollback: flag off = current behavior; rollback = flag flip.
Open questions: does fairness move to per-worker or global (shared `lastPickedAt` still works — argue it)?
Stretch: per-domain politeness (don't hammer one site from N workers).
Interview story potential: a genuine distributed-systems story at approachable scale — "made a single-consumer queue safe for N consumers."

## P2: Public REST API v2 with error codes, pagination, and OpenAPI (2-4 weeks)
Problem: v1's contract is ad-hoc (string errors, 401-for-403, cursor/offset ambiguity); API-key users deserve a documented surface.
Design checklist: error envelope; cursor semantics unified; OpenAPI spec generated or hand-written + CI-validated; deprecation policy for v1 (`DISABLE_DEPRECATED_ROUTES` precedent exists — `getLinks.ts:5-10`).
Likely files: `pages/api/v2/**` (dashboard v2 already began the pattern), shared controllers reused, `packages/types`.
Test plan: contract tests against the spec; v1 untouched-by-construction (new routes only).
Rollout: v2 additive; v1 sunset comms much later.
Interview story potential: "designed and shipped an API version migration" — the API-evolution story.

## P3: Notification/event system (email digests + webhooks out) (2-3 weeks)
Problem: nothing tells users "your import finished / 12 links failed to archive."
Design checklist: event taxonomy; an `Event` table written transactionally with state changes (the **outbox pattern** — finally a real use for it here); digest worker loop (follows existing loop pattern `worker.ts`); outbound webhooks with signature + retries.
Security: webhook URLs are user-supplied server-side fetch targets → **must** go through `safeFetch` (you already know why).
Test plan: outbox write-with-state-change atomicity; delivery retry/backoff unit tests.
Interview story potential: outbox + at-least-once delivery + SSRF-aware webhooks — three senior interview topics in one project.

## P4: Import pipeline hardening (1-2 weeks)
Problem: imports (`controllers/migration/import*.ts`, 5 formats) run on the request path; large files hit `IMPORT_LIMIT` and timeouts; partial failures are opaque.
Design checklist: move parse+insert to the worker (an `ImportJob` row — the AppMigration pattern generalized); progress reporting (job status polled by UI); per-item error report artifact.
Likely files: new worker loop; `migration/index.ts` route becomes job-creator; UI progress in `ImportDropdown.tsx`.
Test plan: the existing `importFromHTMLFile.test.ts` shows the parser seam — extend per format; job-lifecycle integration test.
Rollout: flag; small files keep sync path initially.
Interview story potential: "moved a long-running operation off the request path with progress UX" — extremely common interview scenario, and you'll have really done it.

## P5: Permission system upgrade: roles + audit (2-4 weeks)
Problem: three booleans can't express admin/viewer; no audit trail of grants ([data-model notes](../03-architecture-and-patterns/02-data-model-and-persistence.md)).
Design checklist: role enum mapped onto existing booleans (VIEWER=000, EDITOR=110…) for backward compat; expand-migrate-contract plan for `UsersAndCollections`; `PermissionAudit` append-only table; central `can(user, verb, resource)` helper replacing distributed checks *incrementally* (one controller per PR).
Test plan: the full matrix suite becomes the regression net **before** refactoring (tests-first is the whole point of this project).
Rollback: booleans remain source of truth until the final contract step; every earlier step reversible.
Interview story potential: "migrated a live permission model without breaking existing grants" — the strongest possible authz story.

## P6: Self-host operations kit (2-4 days)
Problem: self-hosters debug blind ([observability notes](../05-quality-engineering/06-observability-and-operations.md)).
Scope: boot config summary (worker + web), health/readiness endpoints, the five-query health dashboard on the admin page, structured JSON log option (`LOG_FORMAT=json`), docs page.
Likely files: `worker.ts`, `getWorkerStats.ts`, `pages/admin/`, docs.
Test plan: snapshot the config summary against env permutations.
Interview story potential: modest but broad — feeds "how do you operate what you ship" across every other story.

---

Choosing guidance: P1 if you want backend depth; P4 if you want the most transferable full-stack story; P5 only after several mid tickets (it touches everything). All six require the design note reviewed (by a mentor, a peer, or written-then-critiqued-by-you-a-week-later) before code.
