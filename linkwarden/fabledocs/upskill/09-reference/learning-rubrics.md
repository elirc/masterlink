# Learning Rubrics

Observable behaviors, not vibes. Grade yourself per skill: can you *do the thing listed*, today, without notes? The interview-ready column is what a strong mid-level candidate demonstrates in a loop.

| Skill | Junior (observable) | Mid (observable) | Senior (observable) | Interview-ready when… |
| --- | --- | --- | --- | --- |
| Codebase navigation | finds a named file; explains one route | traces UI→DB unaided ([Flow 1](../01-codebase-cartography/05-key-flows.md)); knows where each concern lives | predicts where a change ripples (schema→worker→index→mobile) | you narrate a full flow in 3 min with 3+ anchors |
| JS/Node async | writes working async/await | picks all/allSettled/race by failure semantics; explains the worker's loops | designs timeout+abort+supervision stacks; spots leaked racers | [08/01 Q1-Q5](../08-interview-prep/01-js-ts-node-deep-dive.md) at mid bar aloud |
| TypeScript | annotates functions | writes guards/narrowing; explains `unknown` vs `any` with the cache-helper example | designs contracts (discriminated unions, z.infer single-source) | you can defend where `any` is acceptable and where it's a hazard |
| React/data layer | builds a component with useState/useEffect | explains React Query cache model; implements optimistic update w/ rollback | designs key taxonomy + invalidation strategy across mutations | [08/02 Q2+Q4](../08-interview-prep/02-frontend-framework-questions.md) with the four required pieces |
| API design | builds CRUD endpoints | states validation layers, authn/authz split, pagination tradeoffs | evolves contracts (versioning, additive codes, deprecation) | you critique this repo's API for 5 min with fixes sequenced |
| Data modeling | designs tables for a feature | designs join-table permissions, indexes matching queries; expand/contract migrations | judges consistency boundaries; plans zero-downtime changes | [08/03 Q4+Q7](../08-interview-prep/03-api-and-data-modeling-questions.md) with worked plans |
| Async systems | writes a cron job | explains DB-as-queue, idempotency markers, at-least/at-most-once per effect | designs claims/locking, retries with poison-pill defense, backpressure | you can whiteboard [Flow 3](../01-codebase-cartography/05-key-flows.md) + its scaling story |
| Security | hashes passwords, uses ORM params | applies the isolation idiom; writes two-user IDOR tests; explains SSRF layers | threat-models a feature (Kata 8's token coupling); designs containment (origin isolation) | the [05/05 checklist](../05-quality-engineering/05-security-checklist.md) from memory, 80% |
| Testing | writes happy-path tests | picks the layer per behavior; tests where-clauses; uses seams over mocks | designs a strategy for an inherited codebase (pyramid-down plan) | you can say what NOT to test and why |
| Debugging | reproduces + printf | bisects by system boundary; states cheapest probe first | converts incidents into guardrails; distinguishes leak-vs-peak class questions | two timed rounds ([08/05](../08-interview-prep/05-debugging-and-code-review-rounds.md)) at Pass+ |
| Code review | catches syntax/logic slips | catches missing authz, half-write hazards, contract changes; kind specific comments | reviews for "will it stay correct"; sequences safe alternatives | Katas 5–8 headline findings caught |
| Communication | asks answerable questions | writes 5-section PRs; design notes before cross-layer work | writes RFCs a reader could reject; steel-mans alternatives | your PR description teaches the reviewer something |

## Self-assessment protocol

Monthly: pick 4 rows, do each row's "interview-ready when" test cold, score honestly (miss/partial/hit). Two misses in a row on the same row = that's your next fortnight's focus module. The mapping: navigation→01, async/TS→02, API/data/async-systems→03, review→04+07, testing/debugging/security→05, communication→07, everything-under-pressure→08.

Honesty guard: if you haven't *done* the observable behavior (written the test, run the timed round), the cell is a miss regardless of how confident the reading felt. Reading is the input; behaviors are the evidence.
