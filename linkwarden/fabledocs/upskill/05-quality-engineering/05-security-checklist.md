# Security Checklist

Map first (what this repo does, with anchors), then the pre-merge checklist. Interview note: nearly every row is a standard security question; the anchors are your evidence bank.

| Concern | This repo's answer | Anchors | Assessment |
| --- | --- | --- | --- |
| Authentication | NextAuth (credentials bcrypt + ~60 SSO), JWT sessions with DB revocation by `jti` | `[...nextauth].ts`, `verifyToken.ts:24-33` | solid; per-request DB cost noted |
| Authorization / IDOR | per-resource checks; ownership filter idiom on every list; owner/member/public triad | `getPermission.ts`, `getLinks.ts:100-109`, `resolveAccessibleArchive.ts:30-45` | solid but **distributed** — each new endpoint is a risk; two-user tests are the guard |
| Input validation | shared zod schemas w/ max lengths at every string | `schemaValidation.ts` | strong for bodies; query params weakly coerced (`links/index.ts:15-28`) |
| SSRF | 4-layer stack: URL policy, CIDR blocks, DNS-pinned sockets, per-redirect re-validation, time-of-use re-check | `ssrf.ts:306-329`, `safeFetch.ts:16-57,103-124`, `archiveHandler.ts:32-42` | exemplary; `ALLOW_PRIVATE_NETWORK_ACCESS` is the documented footgun |
| XSS (stored) | preserved HTML quarantined to separate origin + 5-min scoped JWT; DOMPurify for readable content; React auto-escaping elsewhere | `createPreservedFormatUrl.ts`, `[linkId].ts:93-102` | strong design; verify every render of `textContent`/highlights sanitizes (*audit task below*) |
| CSRF | next-auth cookie CSRF protections; API accepts Bearer for non-browser | next-auth defaults | *investigate*: state-changing routes rely on next-auth session cookie semantics — confirm SameSite config |
| Injection (SQL) | Prisma parameterization throughout; no raw SQL found in read files | — | low risk; watch any future `$queryRaw` |
| Path traversal | file paths derived from DB ids, never user strings | `resolveAccessibleArchive.ts:70-72` | solid by construction |
| Uploads | formidable size caps, MIME allowlist, size re-check, quota | `[linkId].ts:193-231,41-55` | good; MIME is client-declared — content-sniffing not verified (*investigate*) |
| Secrets | env-based; `NEXT_PUBLIC_` inlining trap; one secret signs two token types (disambiguated by `scope`) | `createPreservedFormatUrl.ts:6,71` | acceptable; scope-check is the load-bearing part |
| Open redirects | not obviously applicable (no redirect params found in read files) | — | *investigate* auth callback URLs |
| Rate limiting | **absent** | — | gap — design in [08/03 Q11](../08-interview-prep/03-api-and-data-modeling-questions.md) |
| Webhooks | `pages/api/v1/webhook/index.ts` exists — unread | — | *audit task*: verify signature validation (esp. if Stripe webhook) |
| Dependency risk | yarn 4 lockfile, patch-package pins, Playwright/browser supply chain | `patches/` | standard; renovate/audit not evident |
| Registration abuse | `NEXT_PUBLIC_DISABLE_REGISTRATION`, whitelisted users, `DISABLE_NEW_SSO_USERS` | `schema.prisma:99-106` | good self-host controls |
| Cookies/session TTL | next-auth defaults + manual exp check | `verifyToken.ts:19-21` | fine |

## Audit tasks the table generated (do these — they're real practice)

1. Read `pages/api/v1/webhook/index.ts`: is there signature verification before any state change? Report like Ticket 8's format.
2. Grep `dangerouslySetInnerHTML` and every render of `link.textContent` / highlight text; confirm DOMPurify wraps each.
3. Read the next-auth options object (`[...nextauth].ts` bottom half — unread by this curriculum, flagged in the [verification log](../09-reference/verification-log.md)) for cookie/SameSite/callback-URL settings.

## Pre-merge security checklist (use on every PR you write here)

☐ Every new query on user data repeats the ownership/membership filter (or calls a resolver that does)
☐ Every new route calls `verifyUser`/`verifyToken` — or is deliberately public and lives under `public/` with an `isPublic` filter
☐ All new body input passes a zod schema with bounded strings; query params validated or safely defaulted
☐ Any server-side fetch of user-influenced URLs goes through `safeFetch`/`assertUrlIsSafeForServerSideFetch`
☐ Any new file path is built from DB-derived ids only
☐ Any new token/signed value gets a scope claim + shape guard on decode
☐ Errors don't leak other tenants' existence (404 vs 403 consistency for cross-tenant probes)
☐ A two-user test exists for any new authz branch
☐ No secret behind `NEXT_PUBLIC_`; no secret in logs
