# Prompt constitution — reference

Full detail behind the SKILL.md summary. Read the section you need.

## Contents

- Capability-profile table
- Picking the specific architecture pattern
- Multi-agent designs
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
| Delegation / synthesis | Can decompose on the fly, write precise worker briefs, and reconcile what comes back | Can execute a scoped brief but can't reliably plan, brief others, or merge conflicting results | Decides whether this executor may be a lead at all (§0a). Weak end → worker only; the author pre-writes every worker brief and the fan-out / reconciliation moves to deterministic code or the human |

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

## Multi-agent designs

**Gate — check this before choosing parallelization (beyond trivial fact-finding) or orchestrator-workers.** Multi-agent pays off on breadth-first work that splits into genuinely independent threads — Anthropic's research system beat a single agent by 90% on its research eval — but at roughly 15× the tokens of a chat, and it fails on work where agents need shared context or make interdependent decisions (most coding). Go multi-agent only if all three hold: the threads are independent, the task's value justifies the cost, and writes to any shared output can stay single-threaded (one agent writes; others only read and report). Otherwise stay single-agent and use compaction plus a progress file for length.

**Who leads.** The lead's job — decompose, brief, reconcile, judge returns — is the hardest synthesis work in the design. A weak executor is a worker, never the lead. If the only available lead would be weak, the authoring agent does the decomposition now, in writing: the design becomes fixed-brief sectioning, and the fan-out and reconciliation are run by deterministic code or the human operator, not by the model. A strong-lead / cheap-worker split is the good version of this.

**What the documents must contain** (fills §0a in `template.md`; one lead document plus one standalone brief per worker type):

- **Spawn limits enforced outside the prompt where possible.** "Workers may not spawn workers" and a max worker count belong in the harness or orchestration code if the environment allows it; a prompt rule is the fallback, and §0a should say which one applies. Early multi-agent systems spawned 50 subagents for simple queries — don't leave the count to the lead's discretion.
- **Effort-scaling table.** Concrete buckets tied to the task's own cases, e.g. simple lookup = 1 agent / 3–10 tool calls; comparison = 2–4 workers / 10–15 calls each; complex = more workers with divided responsibilities. This is what stops both over- and under-investment.
- **Precise worker briefs.** Each carries the overall goal (restated, not referenced), its own objective, what's out of scope (what the other workers own), exact tools/sources, the output format, and the boundaries. Vague delegation is the main cause of duplicated work and gaps.
- **Shared spec.** Whatever the combined result's coherence depends on — style, schema, naming, definitions — copied verbatim into every worker brief. Different territory isn't the same as independent decisions (see above).
- **Return contract: file handoff plus a budgeted summary.** Workers write full output to an exact path and return only the path plus a short summary (e.g. under 2,000 tokens) that includes the basis — what was checked, how confident. This keeps the lead's context clean and avoids information degrading as it passes through the lead.
- **Explicit reconciliation step** as its own line item, with the exact comparison to run, before anything is merged.
- **Checking returns.** A clean worker summary reads like a settled fact; it isn't. State exactly what the lead checks per return (open the file, re-run the check, read the cited source) and for which returns. For high-stakes pipelines, use an independent reviewer agent between steps rather than the lead grading its own workers.
- **Blocked-worker protocol.** A fixed token the worker returns when stuck, and what the lead does on seeing it (§5) — not "use your judgment."

## Section-by-section instantiation guidance

The document (`template.md`) has nine sections, §0 through §8, plus §0a for multi-agent designs (see "Multi-agent designs" above; delete it otherwise). For each, the question isn't "what's the right policy" in the abstract — it's "how literally does this need to be spelled out given the profile."

**0. Role and architecture** — State the chosen pattern and role as a fixed fact, not a question the executor re-derives. A weak executor shouldn't be handed the six-pattern picker and asked to choose; that choice belongs in this document, made once, by the (capable) agent authoring the brief.

**1. Goal and progress** — State the goal in one to three sentences, kept in every re-injection if the executor's context is thin. If a progress file is needed, give its exact path and the exact moment to write to it ("after each completed step, before starting the next") — not "keep notes as you go."

**2. Instructions** — Match instruction specificity to *fragility* (how easy the operation is to get subtly, silently wrong), not to stakes alone: high freedom (heuristics, judgment calls) where multiple approaches are valid; medium freedom (a preferred pattern, some variation tolerated); low freedom (an exact script, no deviation) where the operation is fragile and only one safe path exists. A high-stakes task with many valid, recoverable approaches can still be high freedom; a low-stakes but easy-to-silently-corrupt operation should still be low freedom. Then write it at whatever concreteness the instruction-following signal supports — for a weak executor, prefer several short unconditional rules over one nuanced conditional one, even if that's more verbose, since it's more likely to drop a clause than misapply a short flat rule.

**3. Tooling** — List exact tool names/calls if the choice is fragile at all; don't leave discovery to an executor whose tool-access signal is uncertain. If the executor is text-only, this section states what it *cannot* do and what to output instead (e.g. "propose the command, do not run it").

**4. Guardrails** — Four tiers, softest to hardest to bypass: (1) natural-language instructions — easy to set up but overridable or lose-able from context; (2) tool-shape constraints (poka-yoke) — the bad action is structurally hard to take, e.g. a destructive op requires a confirmation parameter; (3) runtime approval gates — a human confirms before a specific risky call executes; (4) harness-level prohibition — the action is statically blocked outright. Pick per action type, scaled to blast radius, not one global policy. Weight down for a weak executor: natural-language guardrails are the tier most easily lost, so lean on whatever tool-shape or runtime-gate constraints the environment actually supports instead of trusting phrasing alone. Where only natural language is available (text-only executor), state the guardrail as an unconditional, unmissable rule near the top of the document, not buried in prose.

**5. Escalation** — Two independent things to set: threshold (how readily to escalate at all — high-stakes work should escalate readily, low-stakes rarely) and delivery (how the escalation actually reaches someone). For a weak or unattended executor, delivery can't assume synchronous access unless the profile confirms otherwise: give a literal stop instruction and an exact output format for flagging the block (e.g. a fixed token or header a human/orchestrator will grep for), not "ask the user." List concrete trigger conditions rather than "recognize an impasse" — a weak executor's unreliable sense of when it's stuck is exactly the thing this section exists to substitute for.

**6. Error handling** — Enumerate the specific error conditions expected for this task and the exact response to each (retry once then stop; stop immediately; skip and log) rather than a general "handle errors sensibly." Keep this distinct from verification (§8): an execution failure (crashed command, denied access) belongs here; output that's just not correct yet belongs to verification instead.

**7. Fallback** — State the exact checkpoint mechanism if one applies (commit after each unit, exact command) so a bad state can be reverted instead of compounded. This only works where a revert primitive actually exists — a code change is usually fully revertible regardless of its stakes, while a sent email or an external API call often isn't, regardless of how cheap it was. Where no revert primitive exists for an action type, state the exact compensating action agreed before the work starts, rather than leaving "figure out a fallback if this fails" to the executor.

**8. Verification** — Prefer machine-checkable methods the executor can run and paste output from over any self-report, especially where the self-assessment-reliability signal is weak. Give a literal itemized checklist with a required evidence field per item ("what you ran / what it returned"), not a single "confirm this is done" instruction.

## Eval methodology

Write a short scenario in the shape of the task at hand: a one-paragraph prompt that plausibly triggers the specific failure modes the brief is meant to close (e.g. for high-stakes code work, whether it checkpoints and verifies before declaring done; for open-ended research, whether it states its approach and flags uncertainty; for unattended ops work, whether it notices the need for persistence and a reachable escalation path at all).

To run it:
1. Spawn a stand-in executor with the drafted brief as its *only* instructions — no access to this conversation and nothing beyond what's in the document. Use the weakest available real model as the stand-in (e.g. `Agent` with `model: "haiku"`); if no weaker model is available, use the strongest available model but explicitly instruct it to behave with low initiative and no judgment calls beyond what's written, to approximate the target profile.
2. Give it the adapted scenario as its task.
3. Read the resulting transcript against the specific profile weaknesses the brief targeted — not general quality. Did it stop at the escalation trigger instead of guessing past it? Did it use the specified verification method instead of just asserting completion? Did it stay inside the stated freedom level instead of improvising on the fragile step?
4. Any gap found here means the corresponding section in `template.md`'s instantiation wasn't concrete enough — tighten it and rerun, rather than patching the executor's behavior after the fact (there is no "after the fact" — this isn't a session you can correct mid-task).
