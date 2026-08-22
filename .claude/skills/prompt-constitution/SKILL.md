---
name: prompt-constitution
description: Interviews the user to author a portable "prompt constitution" document (architecture through verification) for handing a complex task off to a different executing agent — especially a weaker model or thinner harness that can't be trusted to make its own judgment calls. Produces a written brief plus baseline eval scenarios to test whether the target executor actually follows it. Use when the user wants to write a spec/prompt/brief for another agent to execute, is delegating work to a less capable or unfamiliar model, or asks for help doing prompt/spec engineering for a complex multi-step task.
---

# Prompt constitution

This skill produces a document, not a behavior change in the current session. It's for the moment where the task itself will be executed by *some other* agent — a weaker model, a thinner harness, a contractor's automation, a future session with no memory of this one — and that executor can't be trusted to make the judgment calls a capable agent would make on its own. The brief has to make those calls for it, in writing, before handoff.

Skip this for small, well-scoped handoffs — a one-line instruction doesn't need a constitution. This exists for delegated work that's multi-step, ambiguous, high-stakes, or long-horizon, where an unreliable executor filling gaps with its own judgment is exactly the failure mode to design out.

## Step 1 — Task and executor profile

Gather two things before drafting anything:

1. **The task itself** — goal, time horizon, predictability, parallelizability, stakes, reversibility.
2. **The executor profile** — the axis that doesn't exist in self-governance mode, and the one this whole skill is organized around. Ask directly rather than assuming:
   - What model/harness will actually run this? (name it if known — a specific small model, an unfamiliar agent framework, a non-LLM automation, a junior contractor treated as an "agent")
   - What's its context window / memory like — can it hold a long brief and reread a progress file, or does it need everything short and re-injected?
   - Can it use tools, and which ones — or is it text-in/text-out only?
   - Can it reach a human mid-task if it gets stuck, or is it fire-and-forget once launched?
   - What failure modes are known or suspected — weak instruction-following, confident hallucination, poor self-assessment, no initiative to ask questions, no initiative to stop?

If the user doesn't know some of these, default to the weakest reasonable assumption for the unknowns (no reliable self-assessment, no mid-task human access, easily drifts off narrow instructions) rather than assuming competence you haven't confirmed. See `reference.md`'s capability-profile table for how each answer shifts the sections below.

State the profile back before drafting: *"Sounds like this is headed to \[X], which I'll treat as: can't self-assess reliably, no tool access beyond Y, can't ask for help once running. That pushes the brief toward low freedom everywhere and mechanical verification rather than self-report — sound right?"*

## Step 2 — Architecture, biased by capability

Run the six-pattern triage in `reference.md`, weighted by the executor profile, not just the task shape: patterns that need the executor to *synthesize* well under ambiguity — orchestrator-workers, open-ended loop — assume a competence the profile may not support. When the executor is weak, prefer collapsing toward chaining, routing, or a tight evaluator-optimizer loop with an external (tool-checkable, not self-assessed) pass/fail bar, even where a capable agent might otherwise reach for something more autonomous. State this trade explicitly to the user if it means giving up flexibility the task would otherwise want.

## Step 3 — Draft the document

Use `template.md` as the skeleton and `reference.md`'s section-by-section guidance to instantiate each part *concretely* — no heuristic left for the executor to interpret if the profile says it can't be trusted to interpret it well. The general shift, across every section, is the same: the weaker the executor, the more each judgment call gets made now, in writing, instead of delegated to the executor's own discretion at runtime.

Write the draft to a file the user can hand off wholesale — it has to stand alone with no access to this conversation.

## Step 4 — Eval the brief against a stand-in

Don't hand off a brief that's only been sanity-checked by the same capable agent that wrote it — that checks whether the brief *reads* well, not whether a weaker executor actually *follows* it. Before calling it done:

1. Adapt one short scenario from the task at hand (or draw on the pattern in `reference.md`'s eval-methodology section).
2. Run it against a real stand-in for the target executor — the cheapest/most literal model available (e.g. spawn an `Agent` with `model: "haiku"`), or the strongest model available under an explicit low-initiative persona if no weaker model is accessible, given only the drafted brief as its instructions — not this conversation's context.
3. Read the transcript for exactly the gaps the brief was designed to close: did it ask before doing the risky thing, did it stop at the stated escalation trigger, did it use the verification method specified rather than just asserting done.
4. Feed observed gaps back into the draft. Build the eval before trusting the document, not after — a brief that only reads well to the agent that wrote it hasn't actually been checked against the thing it needs to survive.

## Deeper guidance

See `reference.md` for: the capability-profile table (how each executor signal shifts freedom/guardrails/escalation/verification), the six-pattern architecture picker, section-by-section instantiation guidance for the document, and the eval-methodology detail. See `template.md` for the blank document skeleton.
