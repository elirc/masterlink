# Testing Strategy

## What exists (verified inventory)

| Layer | Tooling | Files | Run by CI? |
| --- | --- | --- | --- |
| Unit/integration (node) | vitest (`vitest.config.mts` — excludes e2e, alias `@`→apps/web) | 8 files: ssrf, resolveAccessibleArchive, createPreservedFormatUrl, preserved token/view, archives route, config route, importFromHTMLFile | **No** |
| E2E | Playwright (`apps/web/playwright.config.ts`, fixtures in `e2e/fixtures/`) | login spec + dashboard/public setup | Yes — grep `@login` only (`playwright-tests.yml:45`) |
| Types | `next build` (web); `tsc --noEmit` script (worker) | — | web yes (via build); worker no |

Read the coverage against the [risk register](../09-reference/risk-register.md): the tested code (archive access, preserved tokens, SSRF) is the *security-critical* code — someone triaged deliberately. The untested-but-complex code (worker scheduling, search, optimistic caching, permission matrix in updateLinkById) is where your contribution energy goes.

## What belongs at each layer (transferable rubric, repo-tuned)

- **Unit**: pure logic — CIDR math (`ssrf.test.ts` is the exemplar), token parsing, normalization, round-robin allocation (after extraction — [Ticket 7](../06-contribution-practice/01-good-first-tickets.md)). Fast, no infra, run always.
- **Integration (node)**: controllers with a test DB or well-designed seams — the archives tests show the house style: call the handler/controller, assert `{status, response}`. Auth/authz matrices live here (cheapest place to prove isolation).
- **E2E**: user-visible flows across web+API+DB — login (exists), link creation ([Ticket 14](../06-contribution-practice/01-good-first-tickets.md)). Few, stable, tagged (`@login` grep pattern scales to `@links` etc.).
- **Do NOT test**: Playwright's rendering itself; Prisma's query correctness; Meili's ranking; next-auth internals. Test *your* configuration of them at the boundary (e.g., "does my where-clause exclude strangers," not "does Postgres filter").

## Fixtures, isolation, and flake defenses (as practiced here)

E2E fixtures compose page objects (`e2e/fixtures/index.ts`, `login-page.ts`, `registration-page.ts`) with per-suite setup files (`tests/global/setup.dashboard.ts`); CI provisions a real Postgres 16 service and a **template database** naming scheme (`TEST_POSTGRES_DATABASE_TEMPLATE` env in `playwright-tests.yml:20`) — the standard trick for fast per-test DB resets. Unit tests stay hermetic via injected seams (`ssrf.ts:259-262`'s `lookup` param — design for this, don't mock modules).

Time and randomness: the codebase uses `Date.now()`/`new Date()` inline (e.g. `getLinkBatchFairly.ts:62-64` trial-window math). Tests for those need vitest fake timers (`vi.useFakeTimers`) or an injected clock — prefer injection when you touch the code anyway.

## The strategy you'd write for this repo (interview-ready summary)

"Push the pyramid down: extract pure logic from the worker and cover it with unit tests; add controller-level authz matrix tests (two-user IDOR suites) since authorization is distributed; keep e2e to a handful of tagged smoke flows; wire vitest + worker typecheck into CI first because untested code that CI never sees isn't a safety net, it's decoration." — That's a complete, defensible answer to "how would you improve testing on a codebase you inherit."
