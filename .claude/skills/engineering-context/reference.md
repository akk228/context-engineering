# Engineering context — reference

Full detail behind the SKILL.md summary. Read the section you need; this isn't meant to be read front to back. Design rationale and sourcing for all of this lives in `context_curation.md` at the project root — this file is the actionable distillation, not the argument for it.

## Contents

- Picking the specific architecture pattern
- Goal and progress persistence
- Degrees of freedom (instructions and tool selection)
- Tool design
- Guardrails
- Escalation
- Error handling vs. verification
- Fallback and checkpointing
- Verification

## Picking the specific architecture pattern

Once intake question 2 (predictability) puts you on the fixed-steps or emergent-steps side, pick the specific pattern:

**Fixed steps** — how do they relate to each other?
- Each step needs the previous step's output, in order → **chaining**.
- Input has to be classified first, then handled down one of several distinct paths → **routing**.
- Steps don't depend on each other, can run side by side → **parallelization** (independent ground = *sectioning*; same task multiple times to compare = *voting*).

**Emergent steps** — what can still be pinned down without a fixed step list?
- A clear, checkable pass/fail or quality bar exists → **evaluator-optimizer**.
- No clean evaluation criterion, but a planner can delegate scoped pieces to workers and synthesize → **orchestrator-workers**.
- Neither a predefined structure nor a clean criterion exists → **open-ended agent loop** (last resort — most autonomous, most expensive to get wrong).

A task can nest more than one pattern (an orchestrator delegating to a worker running an evaluator-optimizer loop, one step of which is a short chain). Pick the top-level pattern first; let the rest show up as sub-structure.

**Before committing to a looping pattern** (evaluator-optimizer, voting): check what's actually inside the loop against reversibility. Generating or checking something disposable (a draft, a dry run) is reversible by construction — discarding it is the revert primitive — so the pattern applies as-is. Re-attempting something with a real, one-way side effect is not reversible; move the loop boundary earlier so iteration happens on a check *before* the irreversible step, never through it. (You can't retry a phase that already moved money the way you retry a draft that missed the bar.)

**Before parallelizing anything beyond pure fact-finding**: independent territory (different files, different sources) isn't the same as independent decisions. Two workers can each do their job correctly and still produce something incoherent together (mismatched style, conflicting architecture choices) if they share an implicit constraint neither one knew to coordinate on. Hand every worker the same shared spec up front and run an explicit reconciliation pass before calling the combined result done.

## Goal and progress persistence

The goal itself always stays in context — it's small, and losing it means losing the ability to steer. What doesn't fit is accumulated progress on a long task. Four techniques, increasing in weight — use the lightest one that solves the actual problem:

1. **Progress file / structured note-taking** — the default. Write progress to a file, reread it. No platform machinery needed.
2. **Compaction** — for a long single-threaded conversation approaching the context limit: summarize and discard stale history, keep architectural decisions and open threads, drop exhausted tool output. Complementary to a progress file, not a replacement — compaction manages the conversation's size, the progress file manages what must survive regardless of size.
3. **Memory tool** — reach for this only when the actual need is cross-task knowledge meant to outlive this one run (house style rules, a schema map still useful next week), not within-task progress tracking.
4. **Hand-built database** — heavier still; worth it only when even the memory tool's structure doesn't fit (relational queries, integration outside the platform's own tooling).

Don't rely on persona to carry the goal — persona is a communication aid, not a substitute for stating the goal itself.

## Degrees of freedom

Match instruction/tool specificity to *fragility* — how easy the operation is to get subtly, silently wrong — not to stakes alone:

- **High freedom** — heuristics, judgment calls; multiple valid approaches, context should decide.
- **Medium freedom** — a preferred pattern exists, some variation is fine.
- **Low freedom** — exact steps, no deviation; the operation is fragile and there's only one safe path.

A spec and a goal statement are two points on this same continuum (spec = low freedom, goal statement = high freedom), not different categories of thing.

Same call applies one level down to tool selection: a low-freedom/fragile tool choice should be instructed exactly; high-freedom/exploratory work can be left to discovery.

## Tool design

When building or choosing tools:
- **Consolidate** — one tool that handles a multi-step operation beats several raw calls the agent has to chain itself.
- **Namespace** clearly once tool counts grow.
- **Control verbosity** — let concise vs. detailed responses be requested; concise cuts token cost substantially.
- **Poka-yoke inputs** — shape arguments so mistakes are structurally hard (e.g. absolute-only file paths), rather than relying on the model reading a warning.
- **Evaluate and iterate** — run real tasks, inspect transcripts and call patterns, refine tool descriptions; small wording changes can produce large behavior changes.

The context window is a public good: every token loaded competes with everything else the agent needs to hold in view. This is the same reasoning behind progressive disclosure (a cheap always-loaded index; full detail loaded only when triggered) and behind keeping tool responses lean by default.

## Guardrails

Four tiers, softest to hardest to bypass — pick per action type, not one global policy:

1. **Natural language instructions** — easy to set up, but overridable or lose-able from context.
2. **Tool-shape constraints (poka-yoke)** — the bad action is structurally hard to take. No wording can override a constraint that lives in the function signature.
3. **Runtime approval gates** — a human confirms before a specific risky call executes.
4. **Harness-level prohibition** — statically blocked outright; hardest to bypass, but rigid enough to cause babysitting if over-applied.

**Direction matters, but isn't a free pass.** An action that reduces exposure or reverses a prior one (rollback, disable a flag, abort) is a *candidate* for a lower tier than raw blast radius suggests, since the moment it's needed is usually the moment something's already wrong and speed matters most. But check that action's own stakes and reversibility before granting the discount — a rollback that itself carries real risk (failing back to a region whose replica hasn't caught up, a "down" migration that doesn't account for new data) doesn't get it just because its direction is "exit."

Known action categories get their tier at intake; a genuinely new category discovered mid-task gets triaged on the spot against the same criteria, disclosed, without reopening the conversation — this happens together with the reversibility check for that same action, not as two separate procedures.

**Disclose conflicts, don't absorb them silently.** What can honestly be disclosed is bounded by what's actually knowable per tier (tool-shape constraints are visible in advance; harness blocks are often opaque until triggered). Proactively flag it when a user instruction directly conflicts with a known rule; reactively report it when a call gets blocked without warning. Disclose the *existence* of a conflicting rule, not its exact mechanics — the boundary conditions are attack-surface information, not user-experience transparency.

## Escalation

Two independent knobs, both set at intake:

- **Threshold** — how readily to escalate at all. Driven by stakes: high stakes escalates readily, low stakes rarely. Exception: an **abort trigger** (see below) isn't subject to this — it escalates regardless of how low the general threshold is set.
- **Delivery** — how the escalation reaches the user. Driven by time horizon: synchronous if the user's present, an out-of-band wake signal if the work is genuinely unattended. This skill doesn't invent that mechanism — it identifies that one is needed and uses whatever the environment already provides; if there's nothing like that available, unattended execution isn't really available either.

**Triggers**: lacking needed access; a genuine impasse; wanting to reconsider approach; an instruction that turns out ambiguous or doesn't fit reality; an instruction conflicting with a known guardrail; a live signal that continuing is actively harmful (an abort trigger — prefer the lower-friction rollback path over pushing forward, and if no safe rollback path exists, that absence is itself part of what gets escalated rather than picking between two bad options alone).

## Error handling vs. verification

- **Execution error** (crashed command, denied access) → handle as an error.
- **Wrong output that just isn't done yet** → not an error, a verification gap (below) — keep iterating toward the goal.
- **Abort trigger** (a live signal that continuing is actively harmful) → neither of the above. Both other categories assume it's safe to keep working; this one means stop immediately and escalate rather than retry or iterate further. Easy to miss under a generate-then-check mental model; a live risk for operational, multi-day tasks specifically.

## Fallback and checkpointing

- Fallback tooling — an alternate path when the primary one is unavailable.
- Request to a human — last resort when no fallback tool applies.
- **Checkpointing** — commit-style checkpoints after each completed unit of work let you revert to a known-good state and let a resumed session recover its bearings from the log instead of re-deriving everything.
- **This only works where a revert primitive exists.** Checkpointing native to code/config doesn't generalize to real external side effects (a production data mutation, a sent email). Reversibility cuts against stakes — a cheap email send has no revert primitive, a risky code change is usually fully revertible — so check it on its own terms via the reversibility axis, not by inferring it from cost. Where no revert primitive exists, "fallback" means something agreed *before* the work starts: a pre-planned compensating action or an explicit human-owned rollback plan.

## Verification

Before declaring anything done:
- **Apply tools** — tests, checkers, commands with predictable output, over self-report.
- **Structured checklists** — an explicit itemized list for larger tasks prevents premature "done" claims; each item verified individually.
- **Adversarial review** — a reviewer whose job is to challenge the output, not confirm it.
- **Reflection** — recognize your own failure honestly rather than deviating from instructions to force a fix. On failure: ask the user to reconsider the approach, or explain concretely why the goal can't be achieved as specified.
- **Spot-check subagent returns** — a subagent's finding is a claim, not verified ground truth, however clean and confident it reads. Weigh it against its stated basis before folding it into your own reasoning or forwarding it to the user; for anything surprising or high-stakes, verify directly rather than propagating on trust.
