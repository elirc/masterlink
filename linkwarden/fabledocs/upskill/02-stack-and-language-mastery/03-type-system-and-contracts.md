# Type System and Contracts (TypeScript)

## Mental model

Types are compile-time claims that erase at runtime. Value flows *into* the typed world through boundaries (HTTP, DB, env, caches) — each crossing needs either a runtime check (zod, type guard) or an assertion (trust). A codebase's type safety = the honesty of its crossings, not its annotation density.

## The crossings in this repo, graded

| Crossing | Mechanism | Grade |
| --- | --- | --- |
| HTTP body → controller | zod `safeParse` + `z.infer` types (`schemaValidation.ts:125-148` → `postLink.ts:16-27`) | strong — single source of truth |
| DB → code | Prisma generated types (`@linkwarden/prisma/client`), incl. enum reuse in zod (`z.enum(AiTaggingMethod)`, `schemaValidation.ts:85`) | strong — schema-derived |
| Query string → route | `Number(req.query.x)` casts (`links/index.ts:15-28`) | weak — NaN passes, `as string` asserts |
| React Query cache → components | asserted types (`data.data.links as LinkIncludingShortenedCollectionAndTags[]`, `links.tsx:95`); helpers take `oldData: any` (`:162-191`) | weak — the honest gap |
| Signed JWT → handler | structural type guard *after* signature check (`createPreservedFormatUrl.ts:26-44`) | exemplary — copy this |
| `unknown` handling | `extractTagsFromQueryData(data: unknown)` narrowing (`links.tsx:118-131`) | good specimen of narrowing |
| env → code | raw `process.env` reads everywhere | weak — no boot validation |

Study the contrast rows: the repo is strongest exactly where it wrote runtime validators and weakest where it asserts. That correlation is the whole lesson.

## Concepts with repo anchors

- **Inference & `z.infer`**: types derived from values (`PostLinkSchemaType`, `schemaValidation.ts:148`) — change the schema, every consumer re-checks.
- **Narrowing**: control-flow analysis (`typeof token === "string"` splits `JWT | string`, `verifyToken.ts` consumers like `verifyUser.ts:20-23`; `Array.isArray`/`in` checks `links.tsx:119-126`).
- **Discriminated unions**: `ResolveAccessibleArchiveResult` (`resolveAccessibleArchive.ts:20-28`) — status literal discriminates the payload; the compiler enforces handling both arms. The `{response,status}` controller convention would benefit from the same treatment ([Ticket 11](../06-contribution-practice/01-good-first-tickets.md)).
- **Type guards**: `isPreservedFormatToken` (`createPreservedFormatUrl.ts:26-44`) and `isArchivalTag` — predicates returning `token is T`.
- **Generics**: sparse here; the cache helpers are where a generic `updateInfiniteData<T>` would pay off — a good kata ([06/04](../06-contribution-practice/04-refactor-and-design-katas.md)).
- **`satisfies`**: not used in the read files; know it anyway — it checks without widening, ideal for config objects.

## Sharp edges checklist

☐ `as` assertions at IO boundaries = unchecked trust (grep `as ` in `packages/router`) ☐ string error channels (`JWT | string`) lose exhaustiveness — prefer tagged unions ☐ env-dependent schema factories change contracts per deployment (`PostUserSchema()`, `schemaValidation.ts:36-58`) ☐ `any` propagates silently — one `any` cache helper untypes everything downstream.

## Drills

1. Rewrite `verifyToken`'s return as `{ok: true, token: JWT} | {ok: false, reason: string}` on paper; list every caller line that must change (`verifyUser.ts:18-23`, `[linkId].ts:105-106`, …).
2. Write the type guard replacing one `oldData: any` helper; what does it *cost* per call?
3. Design `Env` zod schema for the worker; where's the one place it should run?

## Interview angle

[08/01 Q6](../08-interview-prep/01-js-ts-node-deep-dive.md) (`unknown` vs `any` with these exact anchors), Q7 (zod/TS relationship), and the deep-cut differentiator: "signature-valid ≠ shape-valid" from the JWT type guard — few candidates have a real example of validating *signed* data; you do.
