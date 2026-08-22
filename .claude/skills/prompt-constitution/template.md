<!--
Blank skeleton for the handoff document. Delete this comment block before handing off —
the executor should receive only the filled-in sections below, nothing about how they
were produced. Fill every section concretely; see reference.md for how much concreteness
a given executor profile needs per section. Delete any section that's genuinely N/A for
this task rather than leaving a placeholder.
-->

# [Task name]

## 0. Role and architecture

You are [role]. The overall shape of this task is [chaining / routing / parallelization / evaluator-optimizer / orchestrator-workers / open-ended loop], meaning [one-sentence concrete description of what that means for how you should move through the work]. Do not re-derive this — it has already been decided.

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

Do not mark this complete on the basis of your own impression that it's finished — every item needs the evidence field filled in.
