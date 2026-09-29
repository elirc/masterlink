# Language & Runtime Model (JS/Node)

## Mental model from first principles

One thread runs your JS. The event loop alternates: run a task to completion → drain **microtasks** (promise callbacks) → check timers/IO (**macrotasks**). `await` = "suspend me, schedule my continuation as a microtask when the promise settles." Concurrency in Node is *interleaving at await points*, not parallelism — parallelism requires other processes (Playwright's browser) or worker_threads.

## Where this repo uses it well

- **Cooperative infinite loops**: six `while(true)` loops share one thread (`worker.ts:11-20`) because each iteration awaits (`delay()`, Prisma IO) — `linkProcessing.ts:29-84` is the cleanest specimen. Remove the awaits and the process starves.
- **Deliberate concurrency shapes**: parallel-with-isolation (`Promise.allSettled`, `linkProcessing.ts:71-72`), parallel-all (`rssPolling.ts:19-36`), deadline (`Promise.race` + AbortController, `archiveHandler.ts:63-75,110-198`), fire-and-forget (`sendToWayback` un-awaited, `archiveHandler.ts:118-120`).
- **Process-level resilience**: spawn/respawn supervision (`apps/worker/index.ts:3-16`), SIGINT handling (`:14`).

## Sharp edges present here

- Racing doesn't cancel: the timeout path leaves Playwright work running until context close (only `handleMonolith` gets the abort signal, `archiveHandler.ts:189`).
- Un-awaited promises: an un-caught rejection in a fire-and-forget crashes modern Node (then the supervisor restarts it — resilience masking a bug class). *Investigate `sendToWayback`'s internal catch.*
- Client clock in optimistic data (`links.tsx:494-495` `createdAt: new Date()`) — ordering glitches when clocks skew.
- CPU-bound work (jimp previews, readability parsing) blocks every loop in the worker while it runs — the single-thread tax that awaits don't fix.

## Pitfall checklist (any Node codebase)

☐ every `while(true)` awaits ☐ every raced promise is abortable or bounded ☐ every fire-and-forget has `.catch` ☐ no `JSON.parse`/crypto/image work on hot request paths ☐ timers cleared in `finally` (`archiveHandler.ts:204-206` does this).

## Drills

1. Predict the console interleaving of two ticks of `linkProcessing` + one `rssPolling` cycle; then explain why exact order is *not* guaranteed (that's the point).
2. Write `delay()` from scratch (then check `packages/lib/utils.ts`).
3. Find every un-awaited promise in `archiveHandler.ts` (there are at least two: wayback, and the `.catch(()=>{})` context close).

## Interview angle

Q1/Q2/Q4/Q5 in [08/01](../08-interview-prep/01-js-ts-node-deep-dive.md) come straight from this file: event loop with the worker as evidence; Promise combinator selection; tiered error handling; timeout construction. A fifth: "how would you parallelize CPU-bound work in Node?" — answer: process boundary (this repo already pays Playwright's process cost; jimp could move behind worker_threads if measured hot).
