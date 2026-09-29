# Boundaries and Layers

A **boundary** is where one component's responsibility ends and another's begins; a layer is healthy when you can state what it *must not* do. This repo's layers, with their contracts:

| Layer | Lives in | Owns | Must NOT |
| --- | --- | --- | --- |
| UI components | `apps/web/components`, `pages/*.tsx` | rendering, form state, i18n | call `fetch` directly; contain permission rules |
| Client data layer | `packages/router/` | queries, mutations, cache coherence, optimistic logic | import platform APIs (web or RN) |
| HTTP routes | `apps/web/pages/api/**` | method dispatch, query-string coercion, status writing, env guards | contain business logic |
| Controllers | `apps/web/lib/api/controllers/**` | validation, authorization, business rules, persistence orchestration | touch `req`/`res` |
| Shared domain lib | `packages/lib/` | schemas, SSRF, search client, mail, pure utils | know about HTTP or React |
| Data | `packages/prisma/` | schema, migrations, client | — (it's the foundation) |
| Worker | `apps/worker/` | everything async: preservation, indexing, AI, RSS, emails | serve requests; trust user input without re-checking (it re-checks SSRF — `archiveHandler.ts:32-42`) |
| Filesystem | `packages/filesystem/` | S3-vs-local storage | leak backend specifics upward |

## Good boundaries (study these)

- **Controllers return `{response, status}`, never touch res** — `postLink.ts` has zero HTTP imports; that's why `[linkId].test.ts` can test route logic cheaply.
- **Platform injection at the data layer** — `packages/router/links.tsx:383-393` takes `toast/Alert/t` as parameters; the package stays runtime-agnostic for RN reuse.
- **Storage behind verbs** — callers say `createFile({filePath, data})` (`postLink.ts` → `createFolder`); no S3 types escape `packages/filesystem`.
- **The worker/web boundary is the database schema itself** — no RPC, no shared memory; the `Link` row is the message. Clean, but it means *schema changes are protocol changes* between two deployables (deploy-order matters).

## Boundary leaks (real, verifiable)

1. **Routes doing type coercion by hand** — every route re-implements `Number(req.query.…)` (`links/index.ts:15-28`, `search/index.ts:15-28`). Query-string parsing is boundary work, but duplicated and unvalidated (NaN passes through). A shared zod query schema would put the boundary in one place.
2. **`verifyUser` writes HTTP responses from lib code** (`verifyUser.ts:20-23`) — auth logic and response formatting fused; the cost shows up as the parallel `isAuthenticatedRequest.ts` (same checks, different output contract) which has already drifted (its subscription check differs — compare `verifyUser.ts:60-70` with `isAuthenticatedRequest.ts:44-46`). *Divergence between duplicated gates is how auth bugs are born.*
3. **Demo-mode env checks sprinkled through routes** (`links/index.ts:33-37, 44-48, 61-65`) — a cross-cutting policy implemented as copy-paste; belongs in one wrapper.
4. **Client cache helpers know server response shapes as `any`** (`links.tsx:162-191`) — the client/server contract exists only in convention; `LinkIncludingShortenedCollectionAndTags` (`packages/types/global.ts`) is asserted, not verified. tRPC/OpenAPI codegen is the systemic alternative; a hand-written shared type used consistently is the cheap one.
5. **Tags owned by collection owner, enforced in controllers** (`postLink.ts:120-137`, `updateLinkById.ts:120-131`) — an ownership rule that lives in two controllers rather than the schema or one domain function; a third write path could silently break the invariant.

## Transferable: how to find leaks in any repo

Ask of every file: "what does this import that its layer shouldn't know about?" (grep a controller for `next`; grep a shared package for `react-native`) and "what knowledge appears in ≥2 layers?" (coercion, env flags, ownership rules above). Leaks aren't sins — each has a cost/benefit; the skill is *naming* the cost. Interview phrasing: "the boundary I'd tighten first is X because its failure mode is Y."

Drill: `packages/router/config.tsx` and `worker.tsx` exist in the client data layer. Read one and decide: does anything in it belong server-side? Self-grade — Strong: you evaluate it against the "must not" column above and cite a line either way.
