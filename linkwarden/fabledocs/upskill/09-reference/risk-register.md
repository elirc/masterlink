# Risk Register

Consolidated from all modules. Confidence: **confirmed** = behavior read directly in code; **hypothesis** = plausible from code shape, not demonstrated. None of these are demonstrated exploits/bugs; treat every row as a starting point, not an accusation.

| # | Risk | Evidence (anchors) | Impact | Likelihood | Suggested test | Suggested fix | Confidence |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | CI enforces almost nothing (unit tests, lint, worker typecheck all absent) | `.github/workflows/playwright-tests.yml:45` (only `@login`) | regressions ship | high (structural) | n/a | add vitest + typecheck jobs ([Ticket 1](../06-contribution-practice/01-good-first-tickets.md)) | confirmed |
| 2 | `getPermission({linkId})` ignores userId; callers must remember the check | `getPermission.ts:14-26` | IDOR if one caller forgets | medium | grep-audit + per-endpoint two-user tests | resolver returns relationship verdict, not raw rows | confirmed (design) |
| 3 | Auth-gate duplication with drifted subscription semantics | `verifyUser.ts:60-70` vs `isAuthenticatedRequest.ts:44-46` | inconsistent access decisions | medium | table-test both against same fixtures | unify (Kata 1) | confirmed divergence |
| 4 | Archive failures permanent (no retries) | `archiveHandler.ts:203-224` | transient failure = never preserved | high (by design) | poison-URL + transient-failure soak | bounded retries (M3) | confirmed |
| 5 | No multi-worker safety (no row locking on batch claim) | `getLinkBatchFairly.ts` (absence) | double-processing at scale-out | low today | run 2 workers, observe overlap | SKIP LOCKED claims (P1) | confirmed absence |
| 6 | Membership revocation → stale Meili authz fields until incidental reindex | `linkIndexing.ts:133-143`; no reindex trigger on membership change | none user-visible (DB re-check guards) but latent if any path skips re-check | medium | revoke + search integration test | reindex on membership mutation (M9) | confirmed staleness; impact hypothesis |
| 7 | Search deletes: index cleanup unverified | `deleteLinksById` unread vs index docs | ghost results (id re-check filters them → cosmetic) | unknown | delete + direct Meili query | delete-by-id post-commit (M7) | hypothesis |
| 8 | Unbounded RSS fan-out | `rssPolling.ts:19-36` | resource spike on large instances | low self-host / medium cloud | load test w/ 1k feeds | concurrency cap ([Ticket 13](../06-contribution-practice/01-good-first-tickets.md)) | confirmed shape |
| 9 | Scheduler N+1 (per-user queries per tick) | `getLinkBatchFairly.ts:87-96,122-128` | DB chatter at many-user scale | low-medium | query-count assertion | window-function rewrite (Kata 7) | confirmed |
| 10 | Per-request revocation+user queries | `verifyToken.ts:24-33`, `verifyUser.ts:27-35` | latency/DB load at scale | low self-host | route timing | short-TTL cache keyed by jti | confirmed cost; impact hypothesis |
| 11 | Query params coerced without validation (NaN paths) | `links/index.ts:15-28`, `search/index.ts:15-28` | benign defaults today; latent | low | fuzz query params | zod query schemas ([Ticket 15](../06-contribution-practice/01-good-first-tickets.md)) | confirmed |
| 12 | 401-used-for-403 across controllers | `updateLinkById.ts:92-101` et al. | client debugging pain; contract | certain | n/a | error codes first (M5), status later | confirmed |
| 13 | Error contract = prose strings, client translates by key | `postLink.ts:18-25`, `links.tsx:522` | copy change breaks clients | medium | contract test | additive `code` field (M5) | confirmed |
| 14 | `ALLOW_PRIVATE_NETWORK_ACCESS=true` disables SSRF wholesale | `ssrf.ts:319-324` | self-hoster foot-gun | low | n/a | docs warning + narrower allowlist option | confirmed |
| 15 | Upload MIME allowlist trusts client-declared type | `[linkId].ts:219-231` | polyglot file storage (served with stored content-type) | low-medium | upload spoofed-MIME test | content sniffing (file-type lib) | hypothesis (mimetype source = formidable) |
| 16 | Webhook route signature verification unverified by this audit | `pages/api/v1/webhook/index.ts` (unread) | forged billing events if Stripe webhook unverified | unknown | read + forge test | audit task in [05/05](../05-quality-engineering/05-security-checklist.md) | hypothesis |
| 17 | Duplicate-check TOCTOU (no unique constraint backing) | `postLink.ts:47-67` | rare duplicate rows despite setting | low | concurrent-insert test | normalized-url unique index (Kata 6) | confirmed race window |
| 18 | Conditional `useSession()` in shared hook (rules-of-hooks by convention) | `links.tsx:65-70` | breaks if `auth` prop ever varies at runtime | low | lint audit | split hooks (Kata idea) | confirmed |
| 19 | Worker crash-loop without backoff/alerting | `apps/worker/index.ts:3-16` (5s flat respawn) | silent thrash on persistent failure | medium | kill-loop test | backoff + restart counter metric | confirmed |
| 20 | Wayback fire-and-forget unhandled-rejection risk | `archiveHandler.ts:118-120` | worker crash (then respawn) per bad call | unknown | reject `sendToWayback` in test | `.catch` at callsite | hypothesis (internals unread) |
