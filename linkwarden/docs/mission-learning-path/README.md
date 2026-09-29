# Mission Learning Path

This suite trains code-reading skill through ordered missions. You are not passively reading; you are tracing real Linkwarden behavior through files, naming the design intent, answering questions, and self-grading against evidence from the repo.

Recommended pacing: do one junior mission per sitting until you can cite files without searching, then do mid-level missions in pairs because they connect state, API, and persistence. Senior missions should be done slowly with a scratchpad and a habit of asking, "What would break if this changed?"

How to self-grade: strong answers cite exact files and line ranges, describe ownership boundaries, and name a likely bug or regression. For example, a strong link-creation answer mentions client validation, optimistic cache update, API auth, server validation, collection permission, Prisma write, and query invalidation [apps/web/components/ModalContent/NewLinkModal.tsx:88-100](../../apps/web/components/ModalContent/NewLinkModal.tsx#L88), [packages/router/links.tsx:397-548](../../packages/router/links.tsx#L397), [apps/web/pages/api/v1/links/index.ts:9-42](../../apps/web/pages/api/v1/links/index.ts#L9), [apps/web/lib/api/controllers/links/postLink.ts:16-166](../../apps/web/lib/api/controllers/links/postLink.ts#L16).

This suite differs from normal documentation because every mission asks you to do something: trace, annotate, explain, debug, review, or design a safe change. Treat each mission like inspecting a saved link moving through the archive: find the intake form, the catalog record, the permission stamp, and the shelf where the UI displays it.

