# Maintainer Communication

OSS maintainers are volunteers drowning in notifications. Every interaction should arrive pre-digested: what you need, what you already did, smallest possible ask. The same skills transfer 1:1 to asking seniors at work.

## Asking for help without outsourcing thinking

Formula: goal → what I tried (with evidence) → where I'm stuck → my current best guess → specific question.

> **Bad:** "The worker doesn't archive my links, help?"
> **Good:** "Self-hosted, worker running. New links keep `lastPreserved: null`. Worker logs never print 'Processing N links', so getLinkBatchFairly returns empty. I have `STRIPE_SECRET_KEY` unset but `NEXT_PUBLIC_EMAIL_PROVIDER=true`, and my user's `emailVerified` is null — reading getLinkBatchFairly.ts:72-76 I think that filter excludes me. Is unverified-email exclusion intended for self-host, or should I file a docs issue?"

The good version demonstrates the [debugging method](../05-quality-engineering/03-systematic-debugging.md) *and* is answerable in one sentence.

## Bug reports

Title = symptom + scope ("Archive GET returns 401 for public collections when logged out"). Body: version/commit, env mode (Docker? Meili on? Stripe on? — this repo's behavior forks on env more than most; say your flags), minimal repro steps, expected vs actual, and if you can: the anchor ("resolveAccessibleArchive.ts:38-42 suggests public access should pass — am I misreading?"). Never "it's broken"; never a fix demand.

## Proposing a feature

Lead with the problem, not the solution: user story, current workaround, evidence others want it (issue links), *then* a sketch, then explicitly: "happy to implement if the direction sounds right — would you want an RFC first?" Asking for direction before code is the highest-acceptance-rate move in OSS.

## Responding to review

- Every comment gets a response: fixed (link commit), pushback (with reasoning + willingness to defer), or question.
- Batch pushes; don't force re-review per nit.
- Disagree once, well: "I chose X because [evidence]; Y also works and I'll switch if you prefer — your call." Then actually let it be their call. Relitigating costs you the *next* PR's goodwill.
- Thank reviewers for catches, specifically: "good catch on the owner arm — added the test."

## Triage etiquette (when you start helping others)

Reproduce before confirming; label uncertainty ("can't repro on Docker+Postgres 16, can you share your storage backend?"); close-with-kindness template: what was tried, why it's not actionable, what would reopen it.

## Templates (fill-in)

**Question:** Context (1 line) / Tried (2-3 bullets w/ anchors) / Best guess / Question (1 line, answerable).
**Bug:** Version+env flags / Repro (numbered) / Expected vs actual / Suspected anchor (optional).
**Feature:** Problem+who has it / Workaround today / Proposal sketch / "RFC or PR?" ask.
**Review response:** Per-thread: Fixed@commit | Pushback+reason+defer-offer | Clarifying question.

Drill: write the actual issue text for the finding in mid-ticket M7 (search deletion propagation) *as if you'd confirmed the gap* — then the alternate version *as if you couldn't confirm*. The difference in hedging language between those two is a skill interviewers hear instantly in behavioral rounds.
