# Architectural Cartographer

This suite is a top-down map of Linkwarden as a system. It is for learning how the codebase thinks before changing it: the web app is a Next.js Pages Router app with a custom `_app` wrapper, React Query data fetching, NextAuth session/auth handling, API routes under `apps/web/pages/api`, Prisma persistence through `packages/prisma/schema.prisma`, shared client API hooks in `packages/router`, and a separate worker process for preservation/indexing jobs [apps/web/pages/_app.tsx:18-24](../../apps/web/pages/_app.tsx#L18), [apps/web/pages/api/v1/links/index.ts:9-71](../../apps/web/pages/api/v1/links/index.ts#L9), [packages/prisma/schema.prisma:5-8](../../packages/prisma/schema.prisma#L5), [apps/worker/worker.ts:8-20](../../apps/worker/worker.ts#L8).

Read in this order:

1. `00-reading-map.md` for the mental model, top files, data flows, and review red flags.
2. `01-junior-engineer.md` for setup, folder orientation, entry points, TypeScript, components, and route anatomy.
3. `02-mid-level-engineer.md` for architecture maps, API contracts, state ownership, and an end-to-end feature trace.
4. `03-senior-engineer.md` for critique, risk review, security/performance notes, test strategy, and ownership planning.
5. `04-reference-suite.md` when you are actively reviewing, debugging, interviewing, or planning a change.

How to work through checkpoints: answer each question out loud, then compare against the included self-grade. Strong answers should cite files and line ranges, because this codebase rewards proof over memory. Treat a saved link like a preserved artifact moving through a cataloging desk: the UI labels it, the router hook hands it to the intake counter, the API checks identity and permissions, Prisma files it into the archive catalog, and React Query updates the visible shelf [apps/web/components/ModalContent/NewLinkModal.tsx:88-100](../../apps/web/components/ModalContent/NewLinkModal.tsx#L88), [packages/router/links.tsx:394-424](../../packages/router/links.tsx#L394), [apps/web/lib/api/controllers/links/postLink.ts:104-166](../../apps/web/lib/api/controllers/links/postLink.ts#L104).

Code blocks in this suite are condensed and annotated, not verbatim: the code is reformatted, teaching comments were added, and some lines are left out (for example, the `PostLinkSchema` block in `02-mid-level-engineer.md` adds end-of-line comments and collapses the multi-line `collection` field from `packages/lib/schemaValidation.ts`). Open the cited file and range before quoting it. (Note added 2026-10-06.)
