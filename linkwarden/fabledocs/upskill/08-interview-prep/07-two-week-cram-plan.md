# Two-Week Cram Plan

Assumes ~2.5 focused hours/day; interviews at the end of week 2. Everything references files in this curriculum. "Aloud" means actually aloud — silent review does not transfer to interviews.

## Week 1 — build the evidence base

**Day 1** — Run the app if possible ([00-fast-track](../00-fast-track.md) §1). Read [01/05-key-flows](../01-codebase-cartography/05-key-flows.md) Flows 1–2. Aloud: narrate Flow 1 in 3 minutes.
**Day 2** — Flows 3–4. Cards [01/Q1, Q2, Q9](01-js-ts-node-deep-dive.md). Aloud: "how does the archiving pipeline avoid double work?"
**Day 3** — Flows 5–7. Cards 01/Q12, Q14 (auth + SSRF — highest-yield security pair). Start [Story 1](06-behavioral-star-stories.md) worksheet.
**Day 4** — [Pattern catalog](../03-architecture-and-patterns/05-pattern-catalog.md) cards 1–7. Cards 03/Q1, Q5, Q7. Aloud: authn-vs-authz distinction with anchors.
**Day 5** — Pattern cards 8–14. Cards 03/Q2, Q6, Q8. Draft Stories 2 and 3.
**Day 6** — Frontend day: cards 02/Q1–Q6; re-read `useAddLink` (`packages/router/links.tsx:383-534`) until you can whiteboard the optimistic-update lifecycle from memory.
**Day 7 — checkpoint (self-assessment):** ☐ narrate 3 flows without notes ☐ answer Q12/Q14/Q4(03) aloud with anchors ☐ two STAR stories under 2 min ☐ explain optimistic updates with the four required pieces. Anything unchecked → repeat its day before proceeding.

## Week 2 — simulate and sharpen

**Day 8** — **Mock system design** (timed 45 min): [04-system-design-from-this-repo](04-system-design-from-this-repo.md) walkthrough, ideally with a friend as interviewer; else record yourself and review against the junior/mid/senior contrasts.
**Day 9** — **Timed debugging round 1**: Debugging Rounds 1 + 3 from [05](05-debugging-and-code-review-rounds.md), 25 + 20 min, narrating. Review misses.
**Day 10** — Remaining 02 cards (Q7–Q12) + 01 cards Q4–Q8. Rehearse Stories 1–3 + your strongest of 4–8.
**Day 11** — **Timed debugging round 2**: Rounds 2 + 4. Then Review Rounds 1–2 (write your comments, compare to expected findings).
**Day 12** — Remaining 03 cards (Q3, Q4, Q9–Q12). One system-design **variation** prompt (pick multi-tenancy). Teach-back: explain "index finds, database decides" to a non-engineer.
**Day 13** — Full dress rehearsal: 1 flow narration + 6 random cards (roll dice across files) + 1 STAR + 15-min mini-design. Fix the weakest single item only.
**Day 14 — checkpoint + rest:** ☐ system design in 35 min hitting all 6 steps ☐ 80% of cards answered at mid-level bar (tradeoff + example + failure mode) ☐ 4 STAR stories fluent ☐ you can honestly frame your repo experience ([README golden rule](README.md)). Then stop. Sleep beats one more card.

## Daily micro-habits (both weeks)

- Every answer in the pattern: *claim → repo evidence → tradeoff → what I'd do under pressure.*
- Keep one page of "anchors I actually remember" — 10 file:line facts is plenty (suggested: `verifyToken.ts:24-33`, `lastPreserved: null` predicate, `ssrf.ts` four layers, `-Date.now()` temp id, `attributesToRetrieve:["id"]`, `UsersAndCollections` booleans, `finally` stamp, `isPublic` OR-clause, 5-min preserved token, `resolutions: react 19.1.0`).
- When you can't answer a card at mid-level, don't memorize the model answer — reopen the anchor and re-derive it. Derivation sticks; recitation doesn't.
