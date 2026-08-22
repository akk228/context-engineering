# Prompt constitution — reference

Full detail behind the SKILL.md summary. Read the section you need. Design rationale and sourcing for the underlying constitution lives in `context_curation.md` at the project root; this file is about turning that constitution into a document for a *different, possibly weaker* executor, not about self-governance.

## Contents

- Capability-profile table
- Picking the specific architecture pattern
- Section-by-section instantiation guidance
- Eval methodology

## Capability-profile table

Each signal from Step 1 shifts how concretely the document has to spell things out. When a signal is unknown, assume the weaker end.

| Signal | Strong end | Weak end | What it changes |
|---|---|---|---|
| Instruction-following | Follows nuanced, conditional instructions | Follows short, literal, unconditional instructions | §2 freedom level defaults lower; break conditionals into separate flat rules rather than one nested one |
| Context window / memory | Can hold a long brief plus reread a progress file | Needs everything short, re-injected each step | §1 progress persistence needs an explicit re-injection point, not "reread as needed"; trim the brief itself aggressively |
| Tool access | Can run tests/checkers, hit APIs | Text in, text out only | §8 verification can't lean on tool output — needs a checklist the executor can self-apply mechanically, and §4/§5 guardrails can't be tool-shape constraints, only harder-to-ignore natural-language ones |
| Mid-task human access | Can pause and ask | Fire-and-forget once launched | §5 escalation can't assume synchronous delivery — needs a literal stop condition and an explicit "do nothing further, output X" instruction, not "ask the user" |
| Self-assessment reliability | Honestly reports uncertainty/failure | Reports confident completion regardless of actual state | §8 must not rely on self-report at all; prefer checklists the executor fills in with specific evidence per item, not a bare "done" |
| Initiative | Notices when something's off and raises it | Only acts on what's explicitly stated | §6 error handling and §5 escalation triggers must be enumerated explicitly — "notice an impasse and escalate" isn't reliable, list the concrete conditions |

A profile is a bundle of these, not one dial — a model can have decent instruction-following but zero tool access, or vice versa. Instantiate each constitution section against the specific signals that actually bear on it, not a single aggregate "weak/strong" score.

## Picking the specific architecture pattern

**Fixed steps** — favor when the executor's synthesis is unreliable, since fixed steps need no runtime judgment about what to do next.
- Each step needs the previous step's output, in order → **chaining**.
- Input has to be classified first, then handled down one of several distinct paths → **routing**.
- Steps don't depend on each other → **parallelization** (sectioning or voting) — only if the executor (or a separate reconciliation step) can actually combine results coherently; a weak executor doing its own synthesis on parallel output is a common silent-failure point.

**Emergent steps** — only reach for these if the profile supports them:
- A clear, checkable pass/fail exists and the executor can apply it mechanically → **evaluator-optimizer**. Prefer this over the two below when available — the loop bound by an external check is the most forgiving emergent pattern for a weak executor.
- No clean criterion, but the executor can plan and delegate → **orchestrator-workers**. Requires real synthesis competence; verify the profile actually supports it before choosing this.
- Neither — **open-ended agent loop**. Avoid by default for a weak executor; this is the pattern with the least structural protection against drift, and the one that most assumes the executor's own judgment is trustworthy moment to moment.

A task can nest more than one pattern (an orchestrator delegating to a worker running an evaluator-optimizer loop, one step of which is a short chain). Pick the top-level pattern first; let the rest show up as sub-structure — and state the nesting explicitly in §0 of the document, since a weak executor won't reliably infer it.

**Before committing to a looping pattern** (evaluator-optimizer, voting): check what's actually inside the loop against reversibility. Generating or checking something disposable (a draft, a dry run) is reversible by construction — discarding it is the revert primitive — so the pattern applies as-is. Re-attempting something with a real, one-way side effect is not reversible; move the loop boundary earlier so iteration happens on a check *before* the irreversible step, never through it. This matters more for a weak executor, not less — it's less likely to notice on its own that a "just retry" instinct has wandered into unsafe territory.

**Before parallelizing anything beyond pure fact-finding**: independent territory (different files, different sources) isn't the same as independent decisions. Two workers can each do their job correctly and still produce something incoherent together (mismatched style, conflicting architecture choices) if they share an implicit constraint neither one knew to coordinate on. If the executor profile can't be trusted to run its own reconciliation pass, write the shared spec directly into each worker's instructions in the document and add an explicit reconciliation step as its own line item rather than assuming it'll happen implicitly.

## Section-by-section instantiation guidance

The document (`template.md`) has nine sections mirroring the constitution's §0–§8. For each, the question isn't "what's the right policy" (that's `context_curation.md`'s job) — it's "how literally does this need to be spelled out given the profile."

**0. Role and architecture** — State the chosen pattern and role as a fixed fact, not a question the executor re-derives. A weak executor shouldn't be handed the six-pattern picker and asked to choose; that choice belongs in this document, made once, by the (capable) agent authoring the brief.

**1. Goal and progress** — State the goal in one to three sentences, kept in every re-injection if the executor's context is thin. If a progress file is needed, give its exact path and the exact moment to write to it ("after each completed step, before starting the next") — not "keep notes as you go."

**2. Instructions** — Pick freedom level per the fragility rule (`context_curation.md` §2), then write it at whatever concreteness the instruction-following signal supports. For a weak executor, prefer several short unconditional rules over one nuanced conditional one, even if that's more verbose — a weak executor is more likely to drop a clause than misapply a short flat rule.

**3. Tooling** — List exact tool names/calls if the choice is fragile at all; don't leave discovery to an executor whose tool-access signal is uncertain. If the executor is text-only, this section states what it *cannot* do and what to output instead (e.g. "propose the command, do not run it").

**4. Guardrails** — Pick the tier per `context_curation.md` §4's four-tier taxonomy, but weight down: natural-language guardrails are the tier most easily lost by a weak executor, so lean on whatever tool-shape or runtime-gate constraints the environment supports instead of trusting phrasing alone. Where only natural language is available (text-only executor), state the guardrail as an unconditional, unmissable rule near the top of the document, not buried in prose.

**5. Escalation** — Threshold and delivery per `context_curation.md` §5, but delivery has to assume no synchronous access unless the profile confirms otherwise: give a literal stop instruction and an exact output format for flagging the block (e.g. a fixed token or header the human/orchestrator will grep for), not "ask the user." List concrete trigger conditions rather than "recognize an impasse" — a weak executor's threshold for noticing an impasse is the thing this section exists to substitute for.

**6. Error handling** — Enumerate the specific error conditions expected for this task and the exact response to each (retry once then stop; stop immediately; skip and log) rather than a general "handle errors sensibly."

**7. Fallback** — State the exact checkpoint mechanism if one applies (commit after each unit, exact command) and the exact compensating action if no revert primitive exists for an action type — per `context_curation.md` §7's reversibility check. Don't leave "figure out a fallback if this fails" to the executor.

**8. Verification** — Prefer machine-checkable methods the executor can run and paste output from over any self-report, especially where the self-assessment-reliability signal is weak. Give a literal itemized checklist with a required evidence field per item ("what you ran / what it returned"), not a single "confirm this is done" instruction.

## Eval methodology

Write a short scenario in the shape of the task at hand: a one-paragraph prompt that plausibly triggers the specific failure modes the brief is meant to close (e.g. for high-stakes code work, whether it checkpoints and verifies before declaring done; for open-ended research, whether it states its approach and flags uncertainty; for unattended ops work, whether it notices the need for persistence and a reachable escalation path at all).

To run it:
1. Spawn a stand-in executor with the drafted brief as its *only* instructions — no access to this conversation, no access to `context_curation.md`, nothing beyond what's in the document. Use the weakest available real model as the stand-in (e.g. `Agent` with `model: "haiku"`); if no weaker model is available, use the strongest available model but explicitly instruct it to behave with low initiative and no judgment calls beyond what's written, to approximate the target profile.
2. Give it the adapted scenario as its task.
3. Read the resulting transcript against the specific profile weaknesses the brief targeted — not general quality. Did it stop at the escalation trigger instead of guessing past it? Did it use the specified verification method instead of just asserting completion? Did it stay inside the stated freedom level instead of improvising on the fragile step?
4. Any gap found here means the corresponding section in `template.md`'s instantiation wasn't concrete enough — tighten it and rerun, rather than patching the executor's behavior after the fact (there is no "after the fact" — this isn't a session you can correct mid-task).
