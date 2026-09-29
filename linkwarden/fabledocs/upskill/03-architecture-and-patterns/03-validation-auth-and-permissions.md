# Validation, Auth, and Permissions

The security spine in one page. Vocabulary: **authentication** (who are you) vs **authorization** (what may you do) vs **validation** (is this input well-formed) — three different questions answered in three different places here.

## The validation map

| Layer | Mechanism | Example | Trust level |
| --- | --- | --- | --- |
| Client form | shared zod schema | `NewLinkModal.tsx:88-96` | UX only |
| Client mutation | ad-hoc (`new URL`) | `links.tsx:398-404` | UX only |
| Route | manual query coercion | `links/index.ts:15-28` (`Number(...)`, NaN passes) | weak — *known gap* |
| Controller | zod `safeParse` on body | `postLink.ts:16-25` | **the contract** |
| Business rules | code | duplicates `:47-67`, capacity `:69-76` | policy |
| DB | constraints | `@@unique([name, ownerId])`, FKs, cascades | last line |
| File uploads | formidable limits + MIME allowlist + size re-check | `archives/[linkId].ts:193-231` | defense in depth |

## The authentication chain

`verifyToken.ts` (JWT decode → manual `exp` check → **jti revocation lookup** `:24-33`) → `verifyUser.ts` (user exists, has username, email verified if enabled `:49-58`, subscribed if Stripe `:60-70`). Session tokens and API keys share the `AccessToken` table (`schema.prisma:234-246`). Full trace: [Flow 2](../01-codebase-cartography/05-key-flows.md).

## The authorization model

Subjects: owner / member (3 capability booleans) / anonymous. Objects: collections (and everything transitively inside).

| Action | Rule | Enforced at |
| --- | --- | --- |
| Read links | owner OR member (any flags) | every list query re-states it: `getLinks.ts:100-109`, `searchLinks.ts:99-114` |
| Read archives | owner OR member OR `isPublic` | `resolveAccessibleArchive.ts:30-45` |
| Create link | owner OR member.canCreate | `setCollection.ts:25-36` |
| Update link | owner OR member.canUpdate | `updateLinkById.ts:97-101` |
| Move link across collections | **owner only** | `updateLinkById.ts:88-96` |
| Pin link | any member | `updateLinkById.ts:36-65` |
| Upload archive file | owner OR member.canCreate | `[linkId].ts:29-38,151-160` |

Note the *shape*: authorization is **distributed** — each controller re-implements its rule using `getPermission`. There is no central policy engine. Consequences: (a) every new endpoint is a chance to forget; (b) the rules are greppable and explicit. The sharp edge: `getPermission({linkId})` ignores `userId` (`getPermission.ts:14-26`) — it returns relationship *data*, and the caller applies the rule. See Annotation Drill 2.

## What a junior misses vs what a senior checks

| Junior misses | Senior checks |
| --- | --- |
| client validation isn't security | every mutation's controller starts with `safeParse` — grep for exceptions |
| 200-vs-error responses tell attackers things | duplicate check leaks existence only within your own collections (`postLink.ts:53-60`) — scoped correctly |
| list endpoints need isolation too, not just detail routes | ownership filter present in *every* findMany where (the `OR: [{ownerId},{members some}]` idiom) |
| "signed = trusted" | decoded JWTs get shape/scope re-validation (`createPreservedFormatUrl.ts:26-44`) |
| authz once at entry is enough | re-checks at time-of-use: SSRF re-validated in worker (`archiveHandler.ts:32-42`); target collection re-checked on move (`updateLinkById.ts:67-86`) |
| status code semantics | 401 used for authz denials repo-wide (should be 403) — an API-contract decision, fix carefully ([Ticket 8](../06-contribution-practice/01-good-first-tickets.md)) |

## IDOR test you should be able to write from memory

Stranger requests `GET /api/v1/archives/{someoneElsesLinkId}?format=0` → expect 401; member of the collection → 200; anonymous + collection `isPublic` → 200. The existing `[linkId].test.ts` + `resolveAccessibleArchive.test.ts` encode this — read them, then write the equivalent matrix for one *untested* endpoint (e.g. `GET /api/v1/links/[id]`). That exercise is [08/03 Q5](../08-interview-prep/03-api-and-data-modeling-questions.md)'s practice drill and the single highest-value security habit for a mid-level engineer.

Interview angle: this whole page is the answer bank for "how do you secure a multi-user API?" — lead with the isolation idiom, the distributed-vs-central tradeoff, and one sharp edge you'd guard with tests.
