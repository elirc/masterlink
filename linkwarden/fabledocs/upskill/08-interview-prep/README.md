# 08 — Interview Prep (first-class module)

Target: **mid-level fullstack JS interviews**. Over 60% of the question cards here anchor to real Linkwarden code, so you practice answering with evidence instead of recited definitions.

## How mid-level fullstack loops are typically structured

1. **Recruiter/tech screen** (30–45 min) — one coding exercise or rapid-fire fundamentals.
2. **Technical deep-dive** (60 min) — "tell me about a system you know well," then follow-the-thread questions. *This repo is your system.*
3. **Practical coding** (60–90 min) — build/extend a small feature; tests valued.
4. **System design** (45–60 min) — mid-level bar: coherent API + data model + one scaling story, honest tradeoffs.
5. **Behavioral** (45 min) — STAR stories; collaboration, mistakes, ambiguity.

## Files

| File | Round | Cards |
| --- | --- | --- |
| [01-js-ts-node-deep-dive.md](01-js-ts-node-deep-dive.md) | screens + deep-dive | 16 |
| [02-frontend-framework-questions.md](02-frontend-framework-questions.md) | frontend | 12 |
| [03-api-and-data-modeling-questions.md](03-api-and-data-modeling-questions.md) | API/data | 12 |
| [04-system-design-from-this-repo.md](04-system-design-from-this-repo.md) | system design | 1 full walkthrough + 4 variations |
| [05-debugging-and-code-review-rounds.md](05-debugging-and-code-review-rounds.md) | practical | 4 debugging + 4 review simulations |
| [06-behavioral-star-stories.md](06-behavioral-star-stories.md) | behavioral | 8 worksheets |
| [07-two-week-cram-plan.md](07-two-week-cram-plan.md) | all | day-by-day |

## The golden rule

Every answer = **concrete example + tradeoff + failure mode**. Not: "JWTs are stateless tokens." Instead: "JWTs are stateless, which makes revocation the hard part — Linkwarden solves it by checking the token's `jti` against a revocation table on every request (`verifyToken.ts:24-33`), which buys real logout at the cost of a DB query per request. If that query became the bottleneck I'd cache revocations with a short TTL."

That sentence pattern — *what it is → how real code does it → what it costs → what I'd do under pressure* — is the mid-level bar. Add *when the standard answer is wrong* and you're signaling senior.

## Using this repo as your portfolio

You didn't write Linkwarden, and you should never imply you did. The honest frames that work:

- "I did a deep architecture study of an open-source bookmark manager…" (deep-dive round)
- "I contributed tests/features to an OSS project…" (if you do the tickets in [06-contribution-practice](../06-contribution-practice/01-good-first-tickets.md))
- "Here's how a production OSS codebase handles that…" (any conceptual question)

Interviewers consistently reward candidates who cite *specific* real systems. "The archiving worker races each job against a timeout and rotates its browser every 30 minutes" beats any textbook paragraph.
