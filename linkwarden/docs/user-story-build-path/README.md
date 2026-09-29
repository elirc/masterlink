# User Story Build Path

This suite is a progressive set of real Linkwarden feature tickets. Use it like a tech lead's backlog: pick the next story, inspect the named files, implement in a branch, verify acceptance criteria, and write down what you learned.

Difficulty progression:

- Stories 1-3 are Easy: small UI/text/display changes in existing components or pages.
- Stories 4-5 are Medium: UI features that read existing data from hooks and components.
- Stories 6-7 are Medium: new endpoint or state path work.
- Stories 8-9 are Hard: full-stack work touching frontend, backend, and persistence.
- Story 10 is Expert: design-level work with caching/performance implications.

A story is done when every acceptance criterion is independently verifiable, the UI behaves across the relevant pages, the API contract still matches the router hook, and any risky behavior has a focused test. Use existing patterns: React Query hooks in `packages/router`, Zod schemas in `packages/lib/schemaValidation.ts`, API routes under `apps/web/pages/api`, controllers under `apps/web/lib/api/controllers`, and Prisma models in `packages/prisma/schema.prisma` [packages/router/links.tsx:394-548](../../packages/router/links.tsx#L394), [packages/lib/schemaValidation.ts:125-179](../../packages/lib/schemaValidation.ts#L125), [apps/web/pages/api/v1/links/index.ts:9-71](../../apps/web/pages/api/v1/links/index.ts#L9), [apps/web/lib/api/controllers/links/postLink.ts:12-166](../../apps/web/lib/api/controllers/links/postLink.ts#L12), [packages/prisma/schema.prisma:166-198](../../packages/prisma/schema.prisma#L166).

How to use Claude or ChatGPT without getting the solution handed to you:

```text
I'm working on Story X in Linkwarden. I'm stuck on Y. Here is what I've tried: Z.
Don't give me the solution. Ask me questions that help me figure it out.
Please require me to cite the relevant file and line range before I make changes.
```
