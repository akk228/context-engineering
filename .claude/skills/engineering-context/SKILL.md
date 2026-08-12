---
name: engineering-context
description: Runs a brief architecture-intake triage before starting complex, ambiguous, or open-ended agentic work, and applies context-engineering practices (persistence, guardrails, escalation, verification) throughout. Use when a task might span multiple sessions or run unattended, involves high-stakes or hard-to-reverse actions, needs a choice between single-agent/subagent/workflow architectures, or is open-ended enough that the right approach isn't obvious. Not needed for small, well-scoped requests.
---

# Engineering context

Skip all of this for small, well-scoped requests — it exists for the other kind: multi-step, ambiguous, high-stakes, or long-horizon work, where picking an approach silently tends to go wrong in ways that only surface later. Baseline testing against real task prompts showed the gaps this closes aren't hypothetical: agents reliably miss the multi-session/unattended shape of a task unless something forces the question, skip stating process and uncertainty on open-ended research, and leave guardrails implied rather than said out loud. This skill exists to force those specific questions before work starts, not to add process for its own sake.

## Before starting: a five-question intake

Work through these *before* touching real work. State what you land on as a recommendation and let the user confirm or override — don't silently pick one and proceed.

1. **Time horizon.** Single sitting, or does this span days / need to run unattended? Ask this even if something else is blocking you first — the most common miss is getting stuck on a missing detail and never reaching this question at all. Multi-day → note what state needs to survive between sessions (a progress file; see `reference.md`). Unattended → check whether you can actually reach the user when something goes wrong; if not, unattended execution isn't really available here.
2. **Predictability.** Can the steps be listed now, or will they emerge as you go? Fixed steps favor a sequence/branch/parallel-split; genuinely emergent steps favor delegating to workers or an open-ended loop. See `reference.md` for the full six-pattern breakdown if the shape isn't obvious.
3. **Parallelizability.** If splitting work across parallel workers: are the pieces truly independent, or do they share an implicit constraint (style, schema, architecture) that only shows up once combined? If the latter, give every worker the same shared spec up front and reconcile explicitly at the end — condensed summaries alone won't catch a conflict neither worker knew to mention.
4. **Stakes and reversibility — both, they're different questions.** How costly is a wrong or premature "done"? Separately: can this specific action actually be undone? A cheap action can be irreversible; an expensive one can be trivially undoable. Don't infer one from the other, and don't assume a "rollback" is safe just because it's labeled as one — check its own risk too.
5. **Freedom level.** Match instruction specificity to *fragility*, not to stakes alone. A high-stakes task with many valid, recoverable paths is still high-freedom; a low-stakes but easy-to-silently-corrupt operation is still low-freedom.

State the result as a proposal: *"this spans multiple sessions but doesn't need to run unattended, and the steps aren't fully knowable yet — I'd set it up as X, with Y for persistence. Sound right, or do you want it handled differently?"*

## For research, evaluation, or open-ended tasks

State how you're going to approach it — what you'll look at, in what order, parallel or sequential — before diving in, not just the scope questions. When you land on a recommendation, name its actual uncertainty rather than presenting it with more confidence than the underlying research supports.

## Before touching high-stakes or hard-to-reverse work

Say the guardrail out loud rather than leaving it implied by "I'll wait for your answer." If you won't touch production code, send anything, or delete anything until the user confirms a plan, state that explicitly — ending on questions alone doesn't reliably read as a hard gate.

## If something new comes up mid-task

A risky or hard-to-reverse action you didn't anticipate at intake doesn't require reopening the whole conversation. Triage it on the spot against the same stakes/reversibility/freedom questions above, disclose that you're doing so, and proceed accordingly.

## Deeper guidance

See `reference.md` for: the full six-pattern architecture picker, degrees-of-freedom detail, tool-selection and tool-design practices, the four-tier guardrail taxonomy, escalation mechanics (including abort triggers), error handling vs. verification, fallback/checkpointing, and the subagent-coordination and spot-checking requirements. Read it when a specific decision needs more than the summary above — not as a prerequisite to starting.
