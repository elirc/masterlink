# Behavioral STAR Story Worksheets

Eight worksheets. Stories 1–3 are available to you *from studying this repo alone*; 4–8 unlock as you complete tickets/projects in [06-contribution-practice](../06-contribution-practice/README.md) — do the work first, then the story is true. Never claim authorship of Linkwarden itself.

Rehearsal check applies to all: under 2 minutes? concrete numbers/files? ends with impact?

---

## Story 1: Learning a complex codebase fast

Prompts this answers: "Tell me about a time you had to ramp up quickly" / "How do you approach an unfamiliar codebase?"
Source: this curriculum's cartography work.
Situation: wanted production-grade fullstack depth beyond CRUD apps. Task: independently map a ~15-package OSS monorepo (Next.js + background worker + React Native) well enough to contribute.
Action: systematic procedure — manifests → Prisma schema (13 models) → traced one write path end-to-end (modal → optimistic cache → API gate → controller → worker pickup) → read the async pipeline → inventoried tests/CI to see what's actually enforced.
Result: could trace 7 end-to-end flows and identify concrete gaps (e.g., zero worker test coverage; CI running only a login e2e), which became my contribution backlog.
Senior-signal details: the *procedure* is transferable and you can name it; you distinguished enforced-by-CI from claimed-by-docs.
Resume bullet: Reverse-engineered a production OSS monorepo (Next.js/Prisma/Playwright worker) into a documented architecture map and contribution backlog.

## Story 2: A technical tradeoff I can defend from real code

Prompts: "Describe a technical decision with tradeoffs" / "Tell me about a design you disagreed with."
Source: the archiving pipeline study ([Flow 3](../01-codebase-cartography/05-key-flows.md)).
Situation: Linkwarden's worker marks *failed* archives as completed (`archiveHandler.ts:203-224`) — looks like a bug. Task: decide whether I'd change it.
Action: traced the alternative — retrying failures with no attempt-tracking means permanently-dead URLs poison the queue forever; the design trades retries for guaranteed progress. Wrote up the bounded-retry design (attempts column, backoff, dead-letter state) that would buy both, with its migration cost.
Result: changed my initial judgment; documented the invariant so the next reader doesn't "fix" it blindly.
Senior-signal: steel-manning the existing design before proposing change; naming what the weird code protects.
Resume bullet: Analyzed and documented failure-handling tradeoffs (retry vs progress guarantees) in a production job pipeline, producing a bounded-retry redesign proposal.

## Story 3: Finding a security issue class (and thinking like an attacker)

Prompts: "Tell me about a time you improved security/quality" / "What's the most interesting bug you've studied?"
Source: SSRF defense study (`packages/lib/ssrf.ts`, `safeFetch.ts`).
Situation: any URL-archiving product must fetch attacker-controlled URLs. Task: audit how a real product defends this.
Action: mapped four layers — protocol/hostname/CIDR validation, DNS-resolution checking, socket-level IP pinning (defeats DNS rebinding TOCTOU), per-redirect re-validation — and matched each to the bypass it kills; verified the test suite encodes the bypasses.
Result: can now design and *test* SSRF defense from scratch; found the sharp edge worth documenting (`ALLOW_PRIVATE_NETWORK_ACCESS` global off-switch).
Senior-signal: attack-defense pairing, TOCTOU vocabulary, tests-as-spec.
Resume bullet: Audited and documented a four-layer SSRF defense (DNS pinning, redirect re-validation) in a production URL-archiving system.

## Story 4: Shipping a contribution to an unfamiliar codebase

Prompts: "Walk me through a recent PR you're proud of."
Source: complete any 2–3 tickets from [06/01-good-first-tickets.md](../06-contribution-practice/01-good-first-tickets.md) (the test-coverage ones are strongest: worker batch logic, search query builder).
Situation/Task: fill in from the ticket. Action: emphasize reading-first (anchors you studied), matching house style (`resolveAccessibleArchive.test.ts` as template), small blast radius. Result: merged PR / passing suite; cite numbers (files touched, tests added).
Senior-signal: you scoped *down*; you wrote the risk section of your own PR.
Resume bullet: Contributed tested changes to Linkwarden (OSS, ~X k stars), including first unit coverage for its background-worker scheduling logic.

## Story 5: Debugging something non-obvious

Prompts: "Hardest bug you've debugged."
Source: run Debugging Round 1 or 4 from [05-debugging-and-code-review-rounds.md](05-debugging-and-code-review-rounds.md) against a local instance until you've *actually* reproduced and diagnosed it.
Structure: symptom → bisection question you asked → cheapest probe → root cause → regression guard. The memory-growth scenario (browser contexts + buffer math) makes the strongest story.
Senior-signal: you name the probe *before* the theory; you added a guardrail, not just a fix.
Resume bullet: Diagnosed memory growth in a Playwright-based archiving worker via heap profiling; instituted batch-size × buffer capacity limits.

## Story 6: Disagreeing productively / code review conflict

Prompts: "Time you disagreed with a teammate" / "How do you give hard feedback?"
Source: run the review katas ([04/04](../04-code-reading-gym/04-review-katas.md)) with a peer or mentor playing author; or a real review exchange on your OSS PR.
Action: pattern to narrate — separate blocking (correctness/security) from preference; anchor to existing repo convention rather than taste ("public routes elsewhere gate on isPublic — see resolveAccessibleArchive"); offer the smaller safe alternative.
Result: agreement + a better diff; relationship intact.
Senior-signal: kind, specific, evidence-based language; you conceded something too.
Resume bullet: (usually not a bullet — keep this one verbal.)

## Story 7: Handling ambiguity / scoping

Prompts: "Time you had unclear requirements."
Source: do a mid-level ticket from [06/02](../06-contribution-practice/02-mid-level-feature-tickets.md) that forces a design note (e.g., archive-status API field).
Action: wrote the one-page design note first (options, chosen tradeoff, rollback), got feedback *before* code.
Result: implementation matched review expectations; no rework.
Senior-signal: you converted ambiguity into a decision document, not questions.
Resume bullet: Drove design-note-first delivery for cross-layer features (schema/API/UI) in a shared OSS codebase.

## Story 8: A mistake and what changed

Prompts: "Tell me about a failure."
Source: real — you'll generate one doing this curriculum (a test you broke, an anchor you got wrong, an "obvious fix" that violated an invariant like the finally-block one).
Structure: own it fast, quantify blast radius, name the process change (e.g., "now I ask what invariant weird code protects before deleting it — that's a checklist item for me").
Senior-signal: the process change is specific and verifiable, not "I'm more careful now."
Resume bullet: none — this one is for the room.

---

Mapping table (prompt → story): ramp-up→1; tradeoff→2; security→3; proud PR→4; hard bug→5; conflict→6; ambiguity→7; failure→8; leadership-ish ("influenced without authority")→6 or 7 framed around the design note.
