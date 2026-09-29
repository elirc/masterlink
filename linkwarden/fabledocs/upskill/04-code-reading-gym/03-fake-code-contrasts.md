# Fake-Code Contrasts

Eight bad-vs-better pairs. **All snippets are fake** (labeled per house rules); each mirrors a real pattern in this repo so you can check the "better" shape against production code.

## Contrast 1: Missing ownership filter (the IDOR maker)

```ts
// Illustrative fake code: not from this repo — BAD
const links = await prisma.link.findMany({
  where: { collectionId: Number(req.query.collectionId) },
});
```
```ts
// Illustrative fake code: not from this repo — BETTER
const links = await prisma.link.findMany({
  where: {
    collectionId,
    collection: { OR: [{ ownerId: userId }, { members: { some: { userId } } }] },
  },
});
```
Real anchor: the idiom every list query repeats — `getLinks.ts:100-109`. The bad version passes every functional test with one user. That's why isolation needs *two-user tests*, not review vigilance alone.

## Contrast 2: Coupling UI shape to DB shape

```ts
// Illustrative fake code: not from this repo — BAD
// component renders prisma row directly; adds DB-only fields to props
<LinkCard createdById={link.createdById} indexVersion={link.indexVersion} />
```
```ts
// Illustrative fake code: not from this repo — BETTER
// a view type at the boundary; DB bookkeeping never reaches components
type LinkView = { id: number; name: string; url?: string; previewUrl?: string };
```
Real anchor: partially followed — `LinkIncludingShortenedCollectionAndTags` (`packages/types/global.ts`) is a deliberate API shape, but it still carries worker bookkeeping into the client. Cost shows when renaming a column ripples into JSX.

## Contrast 3: Validating with `if` chains instead of a schema

```ts
// Illustrative fake code: not from this repo — BAD
if (!body.url || body.url.length > 2048 || !body.url.startsWith("http"))
  return res.status(400).json({ error: "bad url" });
```
```ts
// Illustrative fake code: not from this repo — BETTER
const parsed = PostLinkSchema.safeParse(body);
if (!parsed.success) return { status: 400, response: firstIssue(parsed) };
```
Real anchor: `postLink.ts:16-25`. The if-chain drifts from the client's copy and misses fields silently; the schema is shared and exhaustive.

## Contrast 4: N+1 in a loop

```ts
// Illustrative fake code: not from this repo — BAD
for (const link of links) {
  const tags = await prisma.tag.findMany({ where: { links: { some: { id: link.id } } } });
}
```
```ts
// Illustrative fake code: not from this repo — BETTER
const links = await prisma.link.findMany({ where: …, include: { tags: true } });
```
Real anchors: the repo mostly does it right (`include` everywhere, e.g. `getLinkBatchFairly.ts:151-157`) — but the scheduler itself queries per-user in a loop (`:87-96`), an *intentional-ish* N+1 worth debating ([08/03 Q10](../08-interview-prep/03-api-and-data-modeling-questions.md)).

## Contrast 5: Optimistic update without rollback

```ts
// Illustrative fake code: not from this repo — BAD
onMutate: (link) => {
  queryClient.setQueryData(["links"], (old) => [fakeRow(link), ...old]);
} // error? the ghost row lives forever
```
```ts
// Illustrative fake code: not from this repo — BETTER
onMutate: snapshot → patch → return context; onError: restore from context
```
Real anchor: the full ceremony at `links.tsx:426-534`. The bad version works in demos and corrupts caches in production — the difference is only visible on failure paths.

## Contrast 6: Swallowed errors in background jobs

```ts
// Illustrative fake code: not from this repo — BAD
try { await archive(link); } catch {} // keep the loop alive!
```
```ts
// Illustrative fake code: not from this repo — BETTER
catch (error) { log(link.id, error); markAttempt(link); if (!browser.isConnected?.()) await restartBrowser(); }
```
Real anchor: `linkProcessing.ts:58-68` — logs *and* reacts (browser health check). Silent catch keeps the loop alive and guarantees you learn about the failure from users. Note the repo's remaining gap: logs aren't metrics.

## Contrast 7: Side effect before the decision is final

```ts
// Illustrative fake code: not from this repo — BAD
await deleteFiles(link);          // destructive first
const ok = await checkPermission(user, link);   // decide second
if (!ok) return 403;              // ...files already gone
```
```ts
// Illustrative fake code: not from this repo — BETTER
authorize → validate → THEN mutate, destructive-last, ideally recoverable
```
Real anchor: `updateLinkById.ts:133-144` deletes old files *after* auth but inside the URL-change branch and before the DB update — mostly right order, still non-atomic (discussed in [08/03 Q12](../08-interview-prep/03-api-and-data-modeling-questions.md)). The bad version above is the shape to never write; the repo shows the defensible middle.

## Contrast 8: Public contract change hidden in a "cleanup"

```ts
// Illustrative fake code: not from this repo — BAD (diff excerpt)
- return res.status(401).json({ response: "Collection is not accessible." });
+ return res.status(403).json({ error: { code: "FORBIDDEN" } });  // in a "tidy responses" PR
```
```ts
// Illustrative fake code: not from this repo — BETTER
+ return res.status(401).json({ response: "Collection is not accessible.",
+   code: "COLLECTION_NOT_ACCESSIBLE" });  // additive; flip status in a major version
```
Real anchor: the strings clients (and the mobile app, and API-key scripts) already depend on — `updateLinkById.ts:92-101`, client `t(error.message)` coupling `links.tsx:522`. "Correct" changes to public contracts are still breaking changes; sequence them.

---

Drill: for each contrast, name the test that would catch the bad version (isolation two-user test; snapshot type test; schema round-trip; query-count assertion; failure-path cache test; log/metric assertion; ordering test; contract/pact test). Self-grade — Strong: 6+ named without peeking at this line.
