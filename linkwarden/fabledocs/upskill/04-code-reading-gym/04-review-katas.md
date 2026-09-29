# Review Katas

Eight fake PRs. For each: read the intent + diff summary, write your review (blocking / important / optional), then compare. Four of these have timed interview versions in [08/05](../08-interview-prep/05-debugging-and-code-review-rounds.md). Findings must be phrased as you'd actually post them — kind, specific, anchored to repo precedent.

## Kata 1: "Public collection RSS feed"
Author intent: expose `GET /api/v1/public/collections/rss?id=` returning latest links as RSS XML.
Fake diff summary: new route file; fetches collection by id; string-concatenates XML from link names/urls; no auth (it's public!).
Files this resembles: `pages/api/v1/public/**`, `resolveAccessibleArchive.ts:38-42`.
Expected — Blocking: no `isPublic: true` filter (private data exposure); XML injection via link names (user-controlled). Important: unbounded query (no take); NaN id handling. Optional: Content-Type header, caching.
Good comment: > "Public routes elsewhere gate on `isPublic` (resolveAccessibleArchive.ts:41) — without that filter this exposes private collections by id-guessing. Can we add the filter plus a two-user test like the archives suite?"

## Kata 2: "Faster duplicate check"
Intent: simplify `postLink.ts:47-67` to one `findFirst({where:{url}})`.
Expected — Blocking: drops ownership scoping (cross-tenant existence leak + wrong behavior). Important: loses www/slash normalization. Optional: normalized-url column + unique index as the durable fix.
Lesson: where-clause edits are authz edits.

## Kata 3: "Retry failed archives"
Intent: stop marking failures as preserved (`archiveHandler.ts:203-224`).
Expected — Blocking: poison-pill loop without attempt bounds. Important: UI semantics of null-forever. Optional: the attempts-column design.
Lesson: ask what invariant the weird code protects.

## Kata 4: "Type-safe cache helpers"
Intent: replace `oldData: any` with generics in `links.tsx:162-191`; accidentally drops the not-found-prepend branch (`:182-188`).
Expected — Blocking: silent behavior change in a types-only PR. Important: shape assumptions vs dashboardData. Optional: split PR.
Lesson: refactors get behavioral review, not rubber stamps.

## Kata 5: "Add canDelete check to bulk delete"
Intent: `deleteLinksById` should verify member `canDelete` per link.
Fake diff summary: loops over linkIds, calls `getPermission({userId, linkId})` per id, filters by `members.some(m => m.userId === userId && m.canDelete)` — but forgets the **owner** case (`ownerId === userId` grants everything).
Files this resembles: `updateLinkById.ts:97-101`, `[linkId].ts:29-38`.
Expected — Blocking: owners can no longer delete their own links (the OR is `owner || member-with-flag`, never member-flag alone). Important: N+1 permission queries for bulk ops (batch by collection instead). Optional: extract a `can(userId, "delete", link)` helper — third duplication of this rule.
Good comment: > "Every other permission check pairs the member-flag with the owner shortcut (e.g. updateLinkById.ts:97). Owners without membership rows will get 401 here — needs the `ownerId === userId ||` arm plus a test for the owner path."

## Kata 6: "Cache dashboard data in localStorage"
Intent: instant dashboard on reload by hydrating React Query from localStorage.
Fake diff summary: serializes `["dashboardData"]` to localStorage on change; hydrates on boot; no user-id in the storage key; no version/invalidations.
Expected — Blocking: multi-account leakage on shared browser (previous user's links flash); stale data shown as fresh with no revalidation marker. Important: quota/size (dashboards embed links arrays); storage key needs schema version. Optional: React Query's own persister utilities do this with gcTime handling — use them.
Lesson: client caches are also **isolation** surfaces.

## Kata 7: "Configurable archive formats per collection"
Intent: add `Collection.archiveAsPDF` etc., overriding user defaults (mirrors the tag-level overrides `archiveHandler.ts:85-107`).
Fake diff summary: schema columns + UI toggle + worker reads collection flags first; migration sets all existing collections to `false`.
Expected — Blocking: the migration default silently disables archiving for every existing collection (should be null = inherit, exactly like Tag's nullable booleans `schema.prisma:206-210`). Important: precedence rules undefined (tag vs collection vs user — must be documented and tested); mobile/API clients unaware of new field (additive OK, but response shape grows). Optional: settings-resolution helper to keep the cascade in one place.
Lesson: **null-vs-false is a semantic decision** — this repo already encodes "null = inherit" on Tag; new code should follow or explicitly diverge.

## Kata 8: "Upgrade next-auth"
Intent: bump next-auth major for security fixes.
Fake diff summary: version bump; renames `getToken` import path; lockfile churn; "tests pass" (= the one login e2e).
Expected — Blocking: JWT shape/claims compatibility unverified — existing sessions and the preserved-format tokens share `NEXTAUTH_SECRET` and the encode/decode from `next-auth/jwt` (`createPreservedFormatUrl.ts:2,66-79`); an algorithm/claim change invalidates *both* session and content tokens at once. Important: 60+ provider configs compile-checked only (`[...nextauth].ts` imports) — smoke at least credentials + one OIDC; check `patches/` for patched next-auth (patch-package!). Optional: staged rollout note for self-hosters.
Lesson: dependency upgrades are **contract** changes when tokens/wire formats are involved; "tests pass" is only as strong as the suite (here: one e2e).

---

Self-grade across katas — Basic: you catch the headline blocking issue in ≥5. Solid: you also catch the second-order issue (N+1, storage key, migration default) in ≥4, and every comment cites repo precedent. Strong: your comments consistently offer the smaller safe alternative and distinguish "blocks merge" from "file a follow-up," and you caught Kata 8's shared-secret coupling.
