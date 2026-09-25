<!--
Blank skeleton for the handoff document. Delete this comment block before handing off —
the executor should receive only the filled-in sections below, nothing about how they
were produced. Fill every section concretely; see reference.md for how much concreteness
a given executor profile needs per section. Delete any section that's genuinely N/A for
this task rather than leaving a placeholder.
-->

# [Task name]

## 0. Role and architecture

You are [role]. The overall shape of this task is [chaining / routing / parallelization / evaluator-optimizer / orchestrator-workers / open-ended loop], meaning [one-sentence concrete description of what that means for how you should move through the work]. [If multi-agent: your role in it is lead / worker — see §0a.] Do not re-derive this — it has already been decided.

## 0a. Delegation

<!-- Only for parallelization / orchestrator-workers designs. Delete this whole section for a single-agent task. -->

**Who orchestrates:** [You / deterministic code / the human operator]. [If not you: you are a worker — follow only your worker brief and do not spawn anything.] Spawn limit: at most [N] workers, and workers may not spawn their own workers. [State whether this is enforced by the harness or only by this rule.]

**How many workers to use:**
- [Simplest case, e.g. single fact lookup] → [1 worker / do it yourself], about [N] tool calls
- [Middle case, e.g. comparison of 2–4 items] → [2–4] workers, about [N] tool calls each
- [Largest case] → [N] workers with the split below

**Worker brief** (fill in once per worker; send it verbatim):
- Goal of the overall task: [same sentence as §1]
- Your objective: [the one thread this worker owns]
- Out of scope: [what the other workers own — do not touch it]
- Tools / sources: [exact list]
- Shared spec: [the conventions every worker must follow — style, schema, naming, definitions]
- Output: write full results to `[exact path]`; return only that path plus a summary under [N] tokens stating what you checked and how confident you are
- If blocked: return `[fixed token]` plus one line saying why; do not guess past it

**Reconciliation:** after all workers return, [exact step — e.g. compare every output against the shared spec and list conflicts before merging]. Do not combine results without this step.

**Checking worker returns:** a worker's summary is a claim, not a fact. Before using it, [exact check — e.g. open the output file and confirm the cited source / re-run the stated check] for [every return / any return that is surprising or feeds a risky action].

## 1. Goal

[One to three sentences. State it plainly enough to survive being reread with no other context.]

**Progress tracking:** [Exact file path if one is used, and the exact moment to write to it. Omit this subsection if the task is short enough not to need it.]

## 2. Instructions

[Concrete steps or rules, at the freedom level decided for this task. Prefer several short unconditional rules over one nuanced conditional one if the executor's instruction-following is uncertain.]

## 3. Tooling

[Exact tools/commands to use, or an explicit statement of what the executor cannot do and should propose instead of doing directly.]

## 4. Guardrails

[Unconditional, unmissable rules about what not to do without confirmation. State the tier if it matters (e.g. "this requires human approval before running" vs. "do not do this at all").]

## 5. Escalation

Stop and [exact output format — e.g. a fixed header/token] if any of the following happen:
- [Concrete trigger 1]
- [Concrete trigger 2]
- [...]
- [If multi-agent: a worker returns the blocked token, or exceeds its scope / budget]

Do not attempt to work around a blocked step or guess past an unclear instruction — stop and flag it instead.

## 6. Error handling

- [Error condition] → [exact response: retry once then stop / stop immediately / skip and log / ...]
- [...]

## 7. Fallback

[Exact checkpoint mechanism if one applies — e.g. commit after each completed unit, with the exact command. Exact compensating action if a step has no revert primitive.]

## 8. Verification

Before declaring this done, confirm each item below and record what you actually checked:

- [ ] [Item] — evidence: [what you ran / what it returned]
- [ ] [Item] — evidence: [...]
- [...]
- [ ] [If multi-agent: each worker return checked per §0a] — evidence: [...]
- [ ] [If multi-agent: reconciliation run, conflicts found and resolved] — evidence: [...]

Do not mark this complete on the basis of your own impression that it's finished — every item needs the evidence field filled in.
