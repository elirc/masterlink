# Code Review Mindset

## The five layers (review in this order; stop escalating when you hit a blocker)

1. **Does it work?** — happy path, obvious errors.
2. **Is it correct?** — edge cases, failure paths, concurrency, the *unhappy* 90%.
3. **Will it stay correct?** — tests pinning the behavior; invariants documented; will next quarter's change break it silently?
4. **Does it fit?** — matches this repo's patterns (controllers return `{status,response}`; authz idiom; zod at the edge); doesn't invent a second way to do an existing thing.
5. **Is it kind to future maintainers?** — names, comment-where-nonobvious, diff size, PR narration.

Junior reviewers live at layer 1 and nitpick layer 5's formatting. Mid-level earns the title at layers 2–3. Senior spends most words at 3–4 and *volunteers what they didn't check*.

## Repo-specific review checklist

☐ New/changed `findMany`/`findFirst` on user data: ownership filter present? (the `OR:[{ownerId},{members:{some}}]` idiom — `getLinks.ts:100-109`)
☐ New route: `verifyUser`/`verifyToken` called, or deliberately public under `public/` with `isPublic` filter?
☐ Body input through zod; string fields bounded (`schemaValidation.ts` house style)
☐ Controller stays HTTP-free; route stays logic-free
☐ Any write path: what happens if it half-completes? (transaction or documented tolerance — [08/03 Q12](../08-interview-prep/03-api-and-data-modeling-questions.md))
☐ Worker code: does it preserve the `lastPreserved`/`"unavailable"` invariants? ([Trace 5](../04-code-reading-gym/02-trace-tables.md))
☐ Cache-touching client code: every affected queryKey patched AND rolled back? (`links.tsx:426-534` as the reference)
☐ Env-conditional behavior: documented in `.env.sample`? Tested in both modes?
☐ Response strings/status codes changed: flagged as API contract change?
☐ Schema change: additive? deploy-order safe? Meili reindex needed?

## Example comments (tone calibration)

Blocking, kindly: > "This query drops the members arm of the ownership filter, so shared-collection users lose access (compare getLinks.ts:100-109). Can we add it plus a member-visibility test? Happy to pair on the test setup."

Important, non-blocking: > "This works, but it's the third place we resolve tag ownership (postLink.ts:120, updateLinkById.ts:120). Fine to ship — could you file a follow-up to extract it so they can't drift?"

Optional, clearly labeled: > "nit (feel free to ignore): `linkData` → `optimisticLink` would match the naming in links.tsx."

Asking instead of asserting (when you're <90% sure): > "Is the early return here intentional when `pinnedBy` is empty? Reading updateLinkById.ts:41-65 I'd expect members to fall through to the canUpdate check — might be missing context."

## The reviewer's meta-habits

State scope: "Reviewed logic and authz; did not run it locally." Approve with nits rather than blocking on taste. When you request changes, offer the smallest acceptable version. And review the *tests* hardest — a wrong test is worse than no test because it certifies the bug.

Drill: run Review Katas 5–8 ([04/04](../04-code-reading-gym/04-review-katas.md)) writing full comment threads in this voice, then grade against the expected findings.
