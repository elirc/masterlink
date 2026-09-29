# Annotation Drills

For each excerpt (all real code — open the file at the anchor): annotate **inputs, outputs, dependencies, invariants, side effects, failure modes** before reading the "what to find" notes. Grade yourself with the rubric at the bottom.

## Drill 1: `verifyToken` — `apps/web/lib/api/verifyToken.ts:9-36`
What to find: input is the *request* (token implicit in cookie/header); output is a union `JWT | string` where string = error message (a smell worth noting — errors as values with no discriminant); dependency: DB on every call; invariant: a revoked `jti` never passes; failure mode: DB down = throw (unhandled here — who catches?); side effects: none (read-only).

## Drill 2: `getPermission` linkId branch — `apps/web/lib/api/getPermission.ts:14-26`
What to find: **the userId parameter is unused in this branch** — it returns the link's collection with members regardless of caller; the *caller* must apply the authz rule. Invariant this file does NOT provide: "result ≠ null ⇒ caller authorized" (false for linkId!). Contrast the collectionId branch `:27-36` where the OR-clause does filter by user. This asymmetry is the sharpest edge in the repo — a perfect annotation exercise.

## Drill 3: duplicate-URL check — `postLink.ts:47-67`
What to find: inputs: user's `preventDuplicateLinks` flag + url; normalization produces *two* candidate URLs (with/without www) after trailing-slash trim; scope: only collections the user **owns** (member collections excluded — is that intended? label "investigate"); failure mode: TOCTOU race with concurrent inserts (no unique constraint backs it); output: early 409.

## Drill 4: optimistic link assembly — `packages/router/links.tsx:458-505`
What to find: temp id `-Date.now()` (negative avoids PK collision); fabricated fields (`preview: ""`, dates from client clock); dependencies: reads *four* other caches (collections, tags, user) to fake a realistic row; invariant: every cache patched here must be restored in `onError:521-534`; failure mode: fields the server computes (title-derived name) will differ → visible correction on success.

## Drill 5: fair-batch user selection — `getLinkBatchFairly.ts:45-81`
What to find: eligibility = has unprocessed links AND (no Stripe configured OR active sub OR in trial) AND (email verified if provider on); ordering `lastPickedAt` nulls-first = never-served users win; `take: maxBatchLinks` bounds the user set; side effect elsewhere (`:159-163`) stamps served users; failure mode: env flags silently change who gets served (Debugging Round 1).

## Drill 6: the `finally` block — `archiveHandler.ts:203-230`
What to find: runs on success AND failure AND timeout; re-reads the link (why? because handlers updated columns concurrently); converts remaining nulls to `"unavailable"`; stamps `lastPreserved` unconditionally = **no retries, ever** (the pipeline's central design decision); `indexVersion: null` = re-index trigger (couples archiving to search); if the link was deleted mid-archive: removes files instead (`:225-227`) — a delete/archive race handled deliberately; context closed with error swallowed `:229`.

## Drill 7: Meili ids → Prisma re-check — `searchLinks.ts:93-120`
What to find: input: index hits (ids only); the where re-asserts ownership/membership — the **authorization invariant** lives here, not in the index; dependency: two stores whose consistency is eventual; failure mode: stale index = missing/extra ids, extra ids get filtered (safe direction), missing stay missing until reindex; note `id: { in: meiliIds }` destroys Meili's relevance ordering unless re-sorted (*investigate* the rest of the function).

## Drill 8: signed preserved-format token — `createPreservedFormatUrl.ts:26-44` (the type guard)
What to find: input: decoded JWT (signature already valid!); the guard *still* checks scope, types, suffix-consistency — treating a signed token as untrusted input; invariant: a session JWT can never pass (no `scope`); failure mode prevented: token-confusion attacks (same secret signs both token types); output: type-narrowed token or null.

---

## Self-grading rubric (per drill)

- **Basic:** you identified inputs/outputs and one side effect.
- **Solid:** you named the invariant the code maintains *and* one failure mode, with the line that handles (or fails to handle) it.
- **Strong:** you also articulated what a caller could do wrong (misuse of the contract — Drill 2 is the acid test) and what test would pin the invariant.

If you scored Basic on Drills 2, 6, or 8 — reread [key flows](../01-codebase-cartography/05-key-flows.md) 5–6 before moving on; those three encode the repo's core security/reliability judgment.
