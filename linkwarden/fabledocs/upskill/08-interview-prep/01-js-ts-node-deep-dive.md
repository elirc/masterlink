# JS / TS / Node Deep-Dive Cards

16 cards. Format per the Interview Question Card template; practice = say it aloud in ~90 seconds, citing the anchor.

## Q1: Walk me through the event loop. When does it bite you in real code?

Round: JS/TS deep-dive
What it's really testing: do you understand *why* async code interleaves, not just the diagram.
Repo anchor: `apps/worker/workers/linkProcessing.ts:29-84` — an infinite `while(true)` loop that stays healthy **only because** every iteration awaits (`delay(interval)`, Prisma calls), yielding to the loop so timers/IO fire.
Junior answer sounds like: call stack, task queue, microtasks, setTimeout.
Mid-level answer adds: an `await`-less `while(true)` would starve the process — the worker's loops are cooperative; `delay()` (a promisified setTimeout) is what keeps five concurrent loops (`worker.ts:11-20`) interleaving in one thread.
Senior answer includes: CPU-bound work (jimp image processing, HTML parsing) still blocks everything — that's why heavy lifting is pushed into Playwright (a separate process); when the standard answer is wrong: "just use async" doesn't help CPU-bound tasks — worker_threads or a process boundary does.
Likely follow-ups: microtask vs macrotask ordering; what happens to timers during a long synchronous block.
Practice drill: explain why all six loops in `worker.ts` can share one thread, in 60 seconds.

## Q2: Promise.all vs Promise.allSettled vs Promise.race — when has each been the right call?

Round: JS/TS deep-dive
What it's really testing: failure-mode thinking.
Repo anchor: all three in production here — `allSettled` for batch archiving where one bad link mustn't kill siblings (`linkProcessing.ts:71-72`); `race` for timing out a Playwright session (`archiveHandler.ts:110-198`); `all` for RSS feeds where each item self-catches (`rssPolling.ts:19-36`).
Junior: definitions.
Mid: matches each to its failure semantics — `all` rejects fast (fine when errors are pre-caught), `allSettled` guarantees full iteration, `race` gives you deadlines.
Senior: `race` leaks the loser — the timed-out Playwright work keeps running until context close; pairing race with `AbortController` (`archiveHandler.ts:63-75`) and noting only `handleMonolith` honors the signal (`:189`) shows you read code, not blogs. Also: unbounded `Promise.all` over N feeds is a thundering herd (`rssPolling.ts` — flag `p-limit` as the fix).
Likely follow-ups: implement `allSettled` from `all`; add concurrency limiting.
Practice drill: whiteboard a `withTimeout(promise, ms, signal)` helper that actually aborts.

## Q3: What's a closure? Show me a real bug it causes.

Round: JS/TS deep-dive
What it's really testing: whether you connect the concept to stale-value bugs.
Repo anchor: `apps/web/components/ModalContent/NewLinkModal.tsx:63-82` — the `useEffect` with `[]` deps closes over `initial` and `collections` from first render; it works because it *wants* mount-time values, but add a dependency on live data and the stale closure bug appears. Also `linkProcessing.ts:18-27`: `restartBrowser` mutates the outer `browser` binding — closure as deliberate shared state.
Junior: "a function remembering outer variables."
Mid: the React stale-closure connection — effects/handlers capture the render they were created in; deps arrays exist to refresh captures.
Senior: closures as an ownership tool (the worker's `browser` mutation is fine because one loop owns it; the same trick across concurrent consumers would race).
Likely follow-ups: the classic `for (var i...)` loop; why hooks lint rules exist.
Practice drill: find what `useFetchLinks` (`packages/router/links.tsx:62-104`) closes over and what invalidates it.

## Q4: How do you handle errors in async Node code? Critique a real strategy.

Round: JS/TS deep-dive
What it's really testing: judgment beyond try/catch syntax.
Repo anchor: three tiers in the worker — per-link try/catch that logs and continues (`linkProcessing.ts:58-68`), `finally` that guarantees state convergence (`archiveHandler.ts:203-230`), supervisor respawn for anything uncaught (`apps/worker/index.ts:3-16`).
Junior: try/catch and `.catch()`.
Mid: describes tiered handling — recover where you have context, contain at batch level, restart at process level; notes controllers return `{status, response}` instead of throwing (`postLink.ts:18-25`), so the route layer never needs try/catch for expected failures.
Senior: critiques it — errors are swallowed into `console.log` with no error tracking (**observability** gap); the error contract to clients is a bare string, so clients can't program against failures; `catch {}` on browser close (`linkProcessing.ts:23`) is fine but should be commented.
Likely follow-ups: unhandled rejection behavior in modern Node; error-cause chains.
Practice drill: trace one thrown error from `handleReadability` to its final resting place.

## Q5: How do you put a timeout on an operation that doesn't support one?

Round: JS/TS deep-dive
What it's really testing: async control-flow fluency.
Repo anchor: `archiveHandler.ts:63-75` — a promise that `setTimeout`-rejects and fires `abortController.abort()`, raced at `:110-198`, timer cleared in `finally:203-207`.
Junior: `Promise.race` with a setTimeout.
Mid: adds the two hygiene rules this code demonstrates — clear the timer (else it holds the process/leaks) and signal cancellation to the loser via AbortSignal.
Senior: cancellation is cooperative in JS — the raced work must *check* the signal; here most preservation handlers don't, so the timeout protects the loop's progress but not the resources. The real backstop is closing the browser context.
Likely follow-ups: `AbortSignal.timeout()`; fetch's native signal support.
Practice drill: rewrite the block using `AbortSignal.timeout` and list what behavior changes.

## Q6: `unknown` vs `any` — where does it actually matter?

Round: JS/TS deep-dive
What it's really testing: whether your TS is load-bearing or decorative.
Repo anchor: `packages/router/links.tsx:118-131` — `extractTagsFromQueryData(data: unknown)` narrows step-by-step before use (good); contrast the `oldData: any` in the cache helpers (`:162-191`) where type safety is abandoned exactly where bugs are most likely (hand-maintained cache shapes).
Junior: "any disables checking, unknown makes you check."
Mid: shows the narrowing pattern (`Array.isArray`, `"pages" in data`) and explains *why* cache data arrives untyped (React Query stores by string key, type is caller-asserted).
Senior: the systemic fix — a typed query-key factory / typed helpers so `["links"]` carries its data type; acknowledges the cost (generics complexity) and when `any` is the pragmatic call (rapidly-shifting internal shapes with tests elsewhere).
Likely follow-ups: type guards vs assertions; `satisfies`.
Practice drill: write the type guard that would replace one `oldData: any`.

## Q7: How does zod relate to TypeScript types? Why do you need both?

Round: JS/TS deep-dive
What it's really testing: compile-time vs runtime boundary understanding.
Repo anchor: `packages/lib/schemaValidation.ts:125-148` (`PostLinkSchema` + `z.infer` type) enforced server-side `postLink.ts:16-25` and client-side `NewLinkModal.tsx:89`.
Junior: "zod validates at runtime."
Mid: types are erased at build; every trust boundary (HTTP body, env, file) needs runtime validation; `z.infer` keeps the static type derived from the single runtime source of truth.
Senior: schema-as-contract — sharing it between client and server kills drift; flags the env-dependent schema factory (`PostUserSchema()` at `:36-58`) as a testing hazard (same code, different contract per deployment) and the error-reporting choice (first issue only) as an API-design decision.
Likely follow-ups: zod vs TypeBox/valibot; validating env vars at boot.
Practice drill: add a max-tags rule to `PostLinkSchema` on paper; where else must change?

## Q8: Explain Node's module resolution in a monorepo. How do `@linkwarden/*` imports work?

Round: JS/TS deep-dive
What it's really testing: build-system literacy — where juniors get stuck for days.
Repo anchor: root [package.json](../../../package.json) workspaces; `packages/router/package.json` (deps on sibling packages); `vitest.config.mts` aliasing `@` → `apps/web`.
Junior: node_modules lookup chain.
Mid: workspaces symlink packages into the root node_modules so `@linkwarden/lib` resolves like any dep; the web app transpiles them (Next config), the worker runs TS directly via `tsx` — two consumers, two compilation strategies, one source.
Senior: the sharp edges — duplicate React from version skew (that's what root `resolutions: react 19.1.0` prevents — critical because *hooks break* with two Reacts); `packages/*` must stay runtime-agnostic or the RN bundler chokes (why `packages/router` injects `toast`/`Alert` — `links.tsx:383-393`).
Likely follow-ups: peerDependencies; why "Invalid hook call" usually means duplicate React.
Practice drill: explain what would happen if mobile pinned react 18.

## Q9: What happens between typing a URL into "add link" and the row existing in Postgres?

Round: JS/TS deep-dive (systems fluency screener)
What it's really testing: can you narrate a full stack trip with real detail.
Repo anchor: Flow 1 in [key flows](../01-codebase-cartography/05-key-flows.md) — modal → optimistic cache → POST → verifyUser → zod → setCollection authz → duplicate/capacity checks → server-side title fetch → prisma create.
Junior: browser → API → database.
Mid: adds the two validation points, the authorization point, and the optimistic UI reconciliation.
Senior: adds the deferred side effect (`lastPreserved: null` queues archiving), the SSRF gate on the title fetch (`postLink.ts:28-30, 78-79`), and the latency critique — a slow target site slows link creation; move title-fetch to the worker to fix.
Likely follow-ups: whatever detail you mention — so only mention details you can expand.
Practice drill: 3-minute narration, then have a friend pick any step and ask "what exactly is in the request there?"

## Q10: Node streams/buffers — when have you cared?

Round: JS/TS deep-dive
What it's really testing: memory-pressure awareness.
Repo anchor: `apps/web/pages/api/v1/archives/[linkId].ts:17-22` disables Next's body parser and caps `responseLimit: "50mb"`; uploads go through formidable with `maxFileSize` (`:193-197`); reads load whole files into memory (`readFile` → `res.send(file)`, `:122-127`).
Junior: streams process data in chunks.
Mid: explains why file routes disable the JSON body parser, and that buffering whole archives in memory is fine at 10MB caps but a scaling wall.
Senior: proposes streaming (`fs.createReadStream`/S3 GetObject stream → pipe to res, plus Range support for PDFs) and quantifies: 50 concurrent 50MB responses = 2.5GB heap — that's the argument that wins the change.
Likely follow-ups: backpressure; highWaterMark.
Practice drill: sketch the streamed version of `handleGet`.

## Q11: How do you keep secrets and config sane in a Node app?

Round: JS/TS deep-dive
What it's really testing: production hygiene.
Repo anchor: `dotenv --` wrappers in root scripts; `.env.sample` as the documented surface; conditional infrastructure (`s3Client.ts`, `meilisearchClient.ts:1-10` — client is null when unconfigured); `NEXT_PUBLIC_*` inlined to the browser bundle.
Junior: "use environment variables."
Mid: explains the null-client pattern (absence of config = feature off) and the `NEXT_PUBLIC_` build-time inlining trap.
Senior: critiques — env reads are scattered (grep `process.env` returns dozens of sites), no boot-time validation, so a typo'd var fails deep at runtime; fix = one zod-validated config module. When the standard answer is wrong: secrets in env are fine until you need rotation/audit — then a secrets manager.
Likely follow-ups: 12-factor; how you'd migrate to validated config incrementally.
Practice drill: write the zod env schema for the worker's 8 most important vars.

## Q12: JWT sessions — how do you log someone out?

Round: JS/TS deep-dive / security
What it's really testing: whether you know statelessness cuts both ways.
Repo anchor: `verifyToken.ts:24-33` — every request checks the JWT's `jti` against the `AccessToken` table for `revoked: true`; the same table stores API keys (`schema.prisma:234-246`, `isSession` flag).
Junior: "delete the cookie."
Mid: cookie deletion doesn't invalidate the token elsewhere; you need denylist-by-jti (this repo) or short-lived access + refresh rotation; notes the cost — a DB read per request, which erases much of JWT's statelessness benefit.
Senior: articulates the actual tradeoff space — server sessions (Redis) vs JWT+denylist vs short-TTL+refresh, and that this repo effectively rebuilt server sessions with JWT syntax, which is defensible for a self-hosted app with one DB and no Redis.
Likely follow-ups: where to store tokens (cookie vs localStorage); CSRF implications of cookies.
Practice drill: argue both sides — "this design is redundant" vs "this design is right here" — 60 seconds each.

## Q13: bcrypt and password storage — what does this code do right?

Round: JS/TS deep-dive / security
What it's really testing: baseline security literacy.
Repo anchor: bcrypt in the credentials provider (`[...nextauth].ts` imports bcrypt at `:8`); password minimums in `schemaValidation.ts:21-22` (min 8, max 2048); reset tokens in a dedicated table with expiry (`schema.prisma:118-124`).
Junior: "hash passwords, never plaintext."
Mid: bcrypt = salted, deliberately slow, cost factor tunable; explains why max-length on password fields also matters (bcrypt cost × attacker-supplied 1MB strings = DoS).
Senior: mentions the whole lifecycle — reset tokens must be single-use + expiring + hashed at rest (*investigate:* whether this repo hashes them), timing-safe comparisons, and rate limiting on auth endpoints (absent here — worth flagging).
Likely follow-ups: bcrypt vs argon2; pepper vs salt.
Practice drill: review `pages/api/v1/auth/forgot-password.ts` against that lifecycle list.

## Q14: What is SSRF and how would you defend a URL-fetching feature?

Round: JS/TS deep-dive / security (very common for fullstack)
What it's really testing: can you think like an attacker about *your own* feature.
Repo anchor: the full stack — `packages/lib/ssrf.ts:306-329` (protocol allowlist, hostname/CIDR blocks, DNS resolution check), `safeFetch.ts:16-57` (socket pinned to vetted IPs — kills DNS rebinding), `:103-124` (per-redirect re-validation), re-check at use time `archiveHandler.ts:32-42`.
Junior: "validate the URL."
Mid: names the classic bypasses — redirects, DNS rebinding, IPv6-mapped IPv4, decimal-encoded IPs — and points to the layer here that stops each.
Senior: check-vs-use TOCTOU framing; network-level egress controls as the stronger guarantee; the `ALLOW_PRIVATE_NETWORK_ACCESS` escape hatch as a product-vs-security tension for self-hosters.
Likely follow-ups: cloud metadata endpoints (169.254.169.254 — in the block table `ssrf.ts:43`); how to test this.
Practice drill: read `ssrf.test.ts`, then recite the four layers and the attack each stops, from memory.

## Q15: Sync vs async work on the request path — how do you decide what to defer?

Round: JS/TS deep-dive / architecture
What it's really testing: latency budgeting instinct.
Repo anchor: `postLink.ts` keeps title-fetch inline (`:78-79`, seconds of user-visible latency, needed for immediate UX) but defers all preservation to the worker via `lastPreserved: null`; Wayback submission is fire-and-forget (`archiveHandler.ts:118-120`).
Junior: "slow things go in background jobs."
Mid: the actual criteria — does the response *need* the result? is failure tolerable later? is it idempotent to retry? — applied to the three examples above.
Senior: notes the middle option this repo skips (respond first, enrich via follow-up fetch/websocket) and the observability requirement for anything deferred: if it's async, you need a way to see it's stuck (`countUnprocessedBillableLinks` exists for exactly that, `linkProcessing.ts:74`).
Likely follow-ups: exactly-once vs at-least-once expectations for the deferred work.
Practice drill: argue for moving title-fetch to the worker; then argue against; pick a side.

## Q16: How would you test Node code that touches the network/filesystem?

Round: JS/TS deep-dive / testing
What it's really testing: seams and dependency injection.
Repo anchor: `ssrf.ts:259-262` accepts a `lookup: HostnameLookup` parameter — DNS injected as a function, so `ssrf.test.ts` runs hermetically; contrast module-level singletons (`s3Client.ts`, `meiliClient`) that resist injection.
Junior: "mock it with jest/vitest."
Mid: prefers designed seams (parameters, factories) over module-mocking magic; points at the `lookup` parameter as the house example of testable design.
Senior: layers the strategy — pure logic extracted and unit-tested (the CIDR math in ssrf.ts), integration tests against real Postgres (CI does this for e2e — `playwright-tests.yml` provisions Postgres 16), and the judgment call of what *not* to test (Playwright rendering itself).
Likely follow-ups: test doubles taxonomy; when module mocks are fine.
Practice drill: design (don't write) the test for `getLinkBatchFairly` — what needs injecting? What's the assertion?
