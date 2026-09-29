# Trace Tables

Fill each table yourself before reading the pre-filled rows (some rows intentionally answered as a worked example, some left `?` for you). Columns: step, file:line, value shape at that point, owner (layer), transformation, risk.

## Trace 1: UI → API — "user types a URL and hits save"

| Step | File:line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | `NewLinkModal.tsx:113-114` | `string` in controlled input | UI | keystroke → state | none |
| 2 | `NewLinkModal.tsx:89` | `PostLinkSchemaType` (unvalidated) | UI | object literal assembly | shape drift vs schema |
| 3 | `links.tsx:398-404` | same + throws on bad URL | data layer | `new URL` gate | error message is a translation key |
| 4 | `links.tsx:416` | JSON string | HTTP | serialize | ? |
| 5 | `links/index.ts:39` | `req.body` (**untyped any**) | route | none — passed raw | trust boundary crossed |
| 6 | `postLink.ts:16-27` | `PostLinkSchemaType` (validated) | controller | `safeParse` | first-issue-only errors |
| 7 | `postLink.ts:104-151` | Prisma `Link` row | DB | defaults applied (`type`, timestamps) | ? |
| 8 | back up: `links.tsx:420-424` | `data.response` asserted as link | data layer | JSON → asserted type | server/client type drift |

Your rows: fill 4 and 7's risk cells; then answer — at which single step does the *name* get chosen if the user left it blank? (`postLink.ts:81-88`.)

## Trace 2: Persistence — "what exactly is in the Link row after save (unpreservable URL)"

| Column | Value | Written at | Why |
| --- | --- | --- | --- |
| `url` | trimmed string | `postLink.ts:106` | `.trim()` — second trim after zod's |
| `name` | url itself | `:81-88` | no title fetch (unsafe URL) |
| `lastPreserved` | now | `:138-148` | opt-out of the worker queue |
| `readable/image/monolith/pdf/preview` | `"unavailable"` ×5 | `:138-148` | terminal state, UI shows "no archive" |
| `indexVersion` | null | `:146` | still gets indexed (text search works on name/url) |
| `type` | ? | ? | you: trace `:90-100` |
| `image` (second write!) | ? | `:153-162` | you: when is this non-undefined? |

## Trace 3: Auth — "one request's identity, hop by hop"

| Step | File:line | Value shape | Risk |
| --- | --- | --- | --- |
| cookie/Bearer | `getToken` inside `verifyToken.ts:12` | `JWT \| null` (`{id, jti, exp, ...}`) | header parsing by next-auth |
| exp check | `verifyToken.ts:19-21` | epoch seconds comparison | manual — why not trust getToken? (*investigate: getToken validates signature, this adds explicit expiry*) |
| revocation | `:24-33` | DB row or null | query per request |
| user load | `verifyUser.ts:27-40` | `User & {subscriptions, parentSubscription}` | second query |
| policy gates | `:42-70` | booleans | env-conditional |
| controller receives | `postLink(body, user.id)` | **just the int id** | least privilege — controllers never see the token. Good. |

Your task: mark which hops are authn vs policy, and identify what a controller *cannot* learn about the caller (answer: token metadata — by design).

## Trace 4: Error — "zod rejection travels back to a toast"

| Step | File:line | Value shape |
| --- | --- | --- |
| zod issue | `postLink.ts:18-25` | `{response: "Error: <msg> [path]", status: 400}` |
| route | `links/index.ts:40-42` | `res.status(400).json({response})` |
| mutation | `links.tsx:420-422` | `!response.ok` → `throw new Error(data.response)` |
| onError | `links.tsx:521-522` | `toast.error(t(error.message))` — message used as i18n key |
| rollback | `:526-534` | caches restored from context |

Your task: write what the user literally sees for a 2049-char URL, and name the two places that message text is coupled to (API contract + translation files) — then the fix direction (error codes, [critique #4](../03-architecture-and-patterns/06-architecture-critique.md)).

## Trace 5: Async — "a link's row state through the worker"

| T | Event | `lastPreserved` | `image` | `indexVersion` | Who wrote |
| --- | --- | --- | --- | --- | --- |
| t0 | created (safe URL) | null | null | null | `postLink` |
| t1 | batch picked | null | null | null | (read only; `lastPickedAt` on *User* changes — `getLinkBatchFairly.ts:159-163`) |
| t2 | screenshot done | null | `archives/{c}/{id}.png` | null | `handleScreenshotAndPdf` |
| t3 | job finally | **now** | path (kept) | null (reset) | `archiveHandler.ts:203-224` |
| t4 | indexer pass | now | path | `MEILI_INDEX_VERSION` | `linkIndexing.ts:156-159` |
| t5 | user edits URL | **null** | null | null | `updateLinkById.ts:157-163` — back to t0! |

Your task: add the failure column — same table when the page 404s at t2. (Key: t3 still stamps; `image` becomes `"unavailable"`.) Then say aloud what invariant holds at every T: *(`lastPreserved != null`) ⇒ every format column is non-null* — that's the system's core **invariant**, and now you've derived it rather than memorized it.
