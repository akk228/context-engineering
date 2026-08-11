# Context Engineering

## Goal

The result of this work is meant to help to curate better prompts that are aimed to be put together as a skill for Automated Prompt Optimization.

The result is meant to be a skill that helps an engineer work together with an agent to create a better initial prompt and guide the engineer through a process of complex task execution.

**Problem statement**

-   Create a workflow that achieves the best results based on specific initial requirements.
-   Create a workflow that overcomes various agent constraints and flaws, such as:
    *   Finite context window
    *   Context pollution
    *   False/unnecessary context propagation
    *   Hallucinations

**Outcome**

1. A skill that helps write better prompts and follow context engineering instead of simple prompting.
2. A framework that improves over time through deliberate iteration — not an agent that rewrites its own rules mid-task, but a skill whose instructions get refined between sessions as real failures surface (see §2's evaluation-driven amendment mechanism).

## Task variety

The agentic workflow is heavily dependent on the type of problem being solved. Rather than inventing our own taxonomy, we adopt Anthropic's, since it maps cleanly onto real architectural decisions ([Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)):

-   **Prompt chaining** — fixed sequence of LLM calls, each processing the last one's output. Use when the task decomposes cleanly into ordered subtasks.
-   **Routing** — classify the input, send it down a specialized path. Use when inputs fall into distinct categories that are better handled separately.
-   **Parallelization** — run several LLM calls at once, either by *sectioning* (independent subtasks) or *voting* (same task, multiple attempts, compare). Use for speed or for higher confidence via multiple perspectives.
-   **Orchestrator–workers** — a central agent decomposes the task on the fly and delegates to workers, then synthesizes. Use when the subtasks can't be predicted ahead of time.
-   **Evaluator–optimizer** — one agent generates, another critiques, looped until it passes. Use when you have a clear evaluation criterion and iteration measurably helps.
-   **Open-ended agent loop** — no predefined path; the agent plans, acts, observes, and adapts across an unknown number of steps. Use only when the task genuinely can't be hardcoded — it's the most autonomous and most expensive-to-get-wrong option.

Which of these applies to a given task is decided during the **[architecture intake](#0-architecture-intake)** step below, not guessed at in advance.

## Agent's anatomy

The agent is:

1. An interface to an underlying model.
2. An agentic loop delivering the agentic reasoning.
3. A set of tools it can employ.
4. A user's servant :)

Outlining an agent's anatomy helps us understand how to prompt an efficient agentic workflow.

## Prompt constitution

Below is what we'll call a prompt constitution. Some of these constituents overlap; the separation is there for separation of concerns, not because each is fully independent.

0.  Architecture intake.
1.  Goal definition or problem statement.
2.  Execution instructions.
3.  Tooling.
4.  Guardrails.
5.  Escalation policy.
6.  Error handling.
7.  Fallback mechanisms.
8.  Verification.

[NOTE]: we couldn't care less about these constituents when we're just shooting shit to see what sticks, or when the request has an extremely small scope. We're here for the bigger fish: complex workflow orchestration.

### 0. Architecture intake

Before any execution prompt gets written, the agent runs a short triage against the actual task and **proposes** an architecture out loud — it doesn't silently pick one. The user confirms or overrides. This is what gives the workflow agility: the shape of the solution is negotiated per-task instead of fixed by the skill in advance.

**Triage axes:**

| Axis | Question the agent asks itself | Pulls toward |
|---|---|---|
| Time horizon | Single sitting, or does this span days / unattended stretches? | Multi-day → persisted state + a reconstructable agent identity |
| Predictability | Can the steps be enumerated now, or do they emerge as work proceeds? | Fixed steps → workflow pattern (chaining/routing/parallel); emergent → open-ended agent loop |
| Parallelizability | Do subtasks explore independent territory that would otherwise pollute one context? | Independent exploration → subagents, condensed return |
| Stakes / reversibility | How costly is a wrong or premature "done"? | High stakes → *default* toward low instruction-freedom, tighter verification, tighter guardrails (refined per-action later, not final here) |
| Attention decoupling | Does the user need to walk away while it runs, or just resume a paused chat later? | True unattended execution → separate front-end/orchestrator split; "just don't forget" → persisted state alone is enough |

The output is a stated recommendation, e.g.: *"This spans multiple sessions but doesn't need to run unattended, and the steps aren't fully knowable yet — I'd suggest a single persistent agent identity that reloads a progress file each time, with a subagent spun off only for the research-heavy part. Want me to set it up that way, or do you want tighter checkpoints given the stakes?"*

Two of the axes feed forward into later sections — but as distinct inputs answering distinct questions, not one dial wearing different hats:

-   **Stakes/reversibility** sets a *default baseline* for the instruction-freedom level (§2) and the guardrail tier (§4), and for how *readily* the agent escalates (§5's threshold). It's a starting point set once at intake, not a final answer applied uniformly — §2 checks that default against the operation's actual fragility before using it, and §4 applies it per action rather than once for the whole task.
-   **Time horizon / attention decoupling** sets whether a persisted/reconstructable identity is needed at all (this section), and separately, *how* an escalation gets delivered (§5's delivery mechanism) — a synchronous question if the user's reachable in-session, an out-of-band wake signal if the work is genuinely unattended.

"How cautious should the agent be" and "can the agent reach a human right now" are two different questions. Architecture intake is where both get an initial answer — later sections refine or apply them, they don't re-derive them from scratch.

<details><summary>Why not always run a dedicated orchestrator + user-facing agent?</summary>

It's tempting to default to a two-tier design — a lightweight user-facing agent plus a persistent orchestrator that survives across days. It genuinely earns its complexity when the task requires **true unattended execution** (attention truly decoupled from the conversation). But most "multi-day" needs are really "don't forget where we were," which a *single* reconstructable agent identity solves via persisted external state (progress file, git log) — the pattern Anthropic's long-running-harness work uses ([Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)): same role, same instructions, same tools, reloaded from disk each session.

**A concrete line for "true unattended execution":** if the agent needs to keep making progress through a stretch where no synchronous human response is possible for longer than a normal working session — overnight, over a weekend, mid-flight on a multi-hour job — attention is genuinely decoupled. If the user is just stepping away between messages within the same rough workday, that's "don't forget where we were," and persisted state on a single reconstructable identity is enough.

The two-tier split costs real complexity that a single reconstructable agent avoids:
-   A hand-off contract between the tiers has to be designed, tested, and debugged.
-   The front-end agent needs a way to query orchestrator state without re-ingesting its full history — a second context-curation problem stacked on the first, with real risk of the two views drifting apart.
-   It doesn't solve escalation on its own — an orchestrator stuck while the user is offline still needs an out-of-band wake signal, regardless of how many tiers exist.
-   Authority gets ambiguous: if the user changes the plan mid-session, does the front-end agent edit state directly, or always route through the orchestrator?

Default: one agent holds the goal and *is* the orchestrator. Delegate to subagents only for isolation/parallelism, with an explicit return-budget (e.g., "report back in under 2,000 tokens"). Stand up a genuinely separate orchestrator tier only when the attention-decoupling axis says so.
</details>

### 1. Goal definition and problem statement

1.  Requires clear declaration.
2.  Must always be part of the main context — never pushed out, never delegated to storage.

**Where progress lives vs. where the goal lives**

The goal itself stays in context — it's small, and an agent that can't recall its own goal can't steer a conversation. What *doesn't* fit in context over a long task is the accumulated progress toward that goal. That gets externalized to a persisted note/progress file the agent rereads — structured note-taking, per [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents), which treats this and Anthropic's file-based memory tool as the same lightweight technique.

*Our own addition, not from that source:* don't reach for a heavier memory-as-database system by default. A progress file is the right weight for "don't lose the thread"; a database is solving a different problem — knowledge meant to outlive any single task, which is a separate design decision from progress-tracking.

**Identifying the main agent**

-   Don't rely on persona to carry the goal — a persona is a communication aid (adjust tone/framing so the goal lands clearly), not a substitute for stating the goal itself. If the persona were dropped, the goal statement should still stand on its own.
-   Either the user or the main agent itself must communicate the goal to any subagent it spawns.

### 2. Instructions

**How ambiguous should instructions be?** Not a fixed policy — it's **degrees of freedom** ([Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)):

-   **High freedom** (heuristics, judgment calls) — when multiple approaches are valid and context should decide. "Open field."
-   **Medium freedom** (parameterized templates/pseudocode) — a preferred pattern exists but some variation is fine.
-   **Low freedom** (exact scripts, no deviation) — the operation is *fragile*, easy to get subtly and silently wrong, and there's only one safe path. "Narrow bridge, cliffs on both sides."

The source grounds this in operation **fragility**, not stakes — and the two aren't interchangeable. §0's stakes/reversibility axis gives a reasonable *default* to start from (high stakes, no other information yet → start cautious), but fragility is what should actually decide it: a high-stakes task with many valid, recoverable approaches (e.g. "reduce false positives without hurting recall") stays high freedom despite the stakes; a low-stakes but fragile, easy-to-silently-corrupt operation (a one-way data transform) stays low freedom despite the low stakes. Check fragility before inheriting the stakes default uncritically.

A spec and a goal statement aren't different categories of thing — they're two points on this same continuum (spec = low freedom, goal statement = high freedom).

**Should we rely on user instructions only, or discover together?** Same answer as §0: don't presuppose a freedom level, negotiate it. The agent proposes one based on its read of the task; the user confirms or adjusts.

**Amendment mechanism** — this splits into two different cases that shouldn't share a mechanism:

-   *Mid-task*, the agent hits an instruction that doesn't fit reality → this is an escalation event (§5), not an instructions-section concern.
-   *For the skill itself*, over time → this is evaluation-driven iteration: build evaluations before extensive documentation, observe real failures, add the missing constraint, retest. This is the concrete mechanism behind the second outcome stated at the top of this doc — improvement through deliberate, between-session iteration, not mid-task self-rewriting.

**CoT / ToT** — mostly dissolves rather than needing its own answer. Modern reasoning models handle chain-of-thought internally. Tree-of-thought's core mechanism — searching multiple paths *with backtracking* — is a genuinely different capability than the parallelization/voting pattern from the Task Variety taxonomy (generate N independent attempts, then compare); voting doesn't backtrack mid-path. For most tasks, voting is the practical substitute and this question can be dropped. If a task specifically needs exploration-with-backtracking, that's still an open design question this doc doesn't resolve.

### 3. Agent's tooling

**Instruct, ask, or let it discover?** Same freedom question as §2, applied to tool selection specifically — driven primarily by how fragile it is to pick the wrong tool (a wrong destructive-tool choice is fragile regardless of the task's overall stakes):

-   Low-freedom/fragile tool choice → instruct the exact tool.
-   High-freedom/exploratory work → let the agent discover.

Not a fourth independent decision — the same fragility-first call as §2, applied one level down.

**"How aware is the agent of its own capabilities?"** — this is what progressive disclosure via metadata is for. Every skill/tool gets a cheap always-loaded index (name + one-line description); full instructions load only once triggered. The agent always knows what it *could* reach for without paying the cost of loading everything up front ([Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)).

**Tool design itself is prompt engineering** ([Writing effective tools for agents](https://www.anthropic.com/engineering/writing-tools-for-agents), [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)):

-   **Consolidate** — one `schedule_event` beats three raw calls the agent has to chain itself.
-   **Namespace** clearly when tool counts grow (`service_resource_action`).
-   **Control verbosity** — let the agent request concise vs. detailed responses; concise cuts token cost roughly two-thirds.
-   **Poka-yoke the inputs** — shape arguments so mistakes are structurally hard (e.g. absolute-only file paths), rather than hoping the model reads a warning.
-   **Evaluate and iterate** — run real tasks, inspect transcripts and tool-call metrics, refine descriptions. Small description edits can produce large behavior changes.

### 4. Guardrails

Guardrails protect against the agent doing things it wasn't meant to do. The original two-option framing (natural language vs. harness-level prohibition) is missing two real, distinct options — the fuller taxonomy is four tiers, from softest to hardest to bypass:

1.  **Natural language instructions** — easy to set up, steerable, but can be overridden by other instructions or simply lost from context.
2.  **Tool-shape constraints (poka-yoke)** — the bad action is structurally hard to take (a destructive op requires a confirmation token as a parameter; writes require absolute paths). More granular than a blanket block, and there's no wording to override — the constraint lives in the function signature, not a sentence.
3.  **Runtime approval gates** — a human confirms before a specific risky call executes. Dynamic, per-call, not all-or-nothing.
4.  **Harness-level prohibition** — the action is statically blocked outright. Hardest to bypass, but rigid enough to cause babysitting if over-applied.

Pick the tier per action during §0 intake, scaled to that action's blast radius — not one global policy for the whole task.

**Disclosing conflicts, not absorbing them silently**

What an agent can honestly disclose about tiers 2–4 is bounded by what it can actually see: tool-shape constraints are knowable in advance from the tool's own schema; runtime gates are knowable by category but not exact boundary until triggered; harness-level blocks are often opaque until a call is actually denied. Disclosure has to follow that knowability, not pretend to a completeness the agent doesn't have:

-   **Proactive** — when a user instruction *explicitly and directly* conflicts with a rule the agent already knows about (e.g. "never ask me for confirmation on anything," when destructive ops carry a hard-coded approval gate), say so at the moment the instruction is given — don't silently accept it and let the user discover the gap mid-task.
-   **Reactive** — when a call gets blocked or denied without prior warning, report what happened and why. Don't quietly retry, work around it, or go silent.

Disclose the *existence* of a conflicting rule, not its exact mechanics. Two reasons, not one: it's the same "context is a public good" concern from §0/§3 — a full manifest of constraints at session start is noise for a conflict that may never occur — and precisely enumerating exact rule *boundaries* hands an adversarial user or injected content a map for finding the gap between rules. "I have a constraint here" is transparency; specifying the exact edge conditions that trigger it is closer to attack-surface documentation.

### 5. Escalation policy

We want an agent to avoid doing things it isn't meant to do, and to give honest output rather than force a result.

**Escalation has two independent knobs, both set during §0 intake — keep them separate:**

-   **Threshold** — how readily the agent escalates at all. Driven by stakes/reversibility: high-stakes work escalates readily, low-stakes exploratory work escalates rarely.
-   **Delivery** — how the escalation actually reaches the user. Driven by time horizon/attention decoupling: a synchronous question if the user is present in the session, an out-of-band wake signal (below) if the work is genuinely unattended.

These answer different questions ("should I stop and ask" vs. "how do I reach someone who isn't here") and can point in different directions on the same task — a high-stakes, unattended overnight job escalates *readily* (threshold) but has to do it *asynchronously* (delivery), not less often.

**Triggers for escalation:**

-   Agent lacks access it needs.
-   Agent recognizes an impasse and needs the user's input to proceed.
-   Agent wants to reconsider its approach (arguably a form of impasse).
-   An instruction turns out to be ambiguous or doesn't fit what the agent is observing (the runtime half of §2's "amendment" question).
-   An instruction conflicts with a known guardrail (§4's disclosure principle) — surfaced as a statement, not necessarily a blocking question, but the same "don't absorb it silently" instinct applies.

**Escalation across time gaps** — for genuinely unattended, multi-day work, a second agent tier does *not* solve escalation by itself. What's actually needed is an out-of-band wake mechanism (a scheduled check, a push notification) independent of how many agent tiers exist. This skill doesn't invent that mechanism — it identifies *that* one is needed during §0 intake and uses whatever the operating environment already provides for it (a scheduling primitive, a notification channel); if the environment has nothing like that, unattended execution isn't actually available and the architecture choice reverts to a synchronous, attended one. Don't conflate "we split the architecture" with "we solved escalation" — they're separate problems.

### 6. Error handling

The agent must know what to do when it hits an error — a failed command, a tool without access, and so on.

**Is a wrong output an error?** No — a result that doesn't hit the desired outcome is not a tooling/execution error, it's just the task not yet done. It's handled by the verification loop (§8), not by error-handling logic. Error handling is for *execution* failures (crashed command, denied permission); verification is for *result-quality* failures.

### 7. Fallback mechanisms

-   Fallback tooling — an alternate path when the primary one is unavailable.
-   Request to a human — the last resort when no fallback tool applies.
-   **Checkpointing as a fallback** — for longer tasks, commit-style checkpoints (e.g. git commits after each completed unit of work) let the agent revert to a known-good state instead of compounding a bad one, and let a resumed session recover its bearings by reading the log rather than re-deriving everything ([Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)).
-   **This only works where a revert primitive exists.** Git-style checkpointing is native to code and config — it doesn't generalize to work with real external side effects (a production data mutation, a third-party API call, an email sent). For those, "fallback" has to mean something agreed *before* the work starts — a pre-planned compensating action, or an explicit human-owned rollback plan — not an assumed ability to revert. Flag this gap during §0 intake rather than discovering it after something irreversible has happened.

### 8. Verification

We want the agent to be able to test the results of its own work, and to do so before declaring victory.

-   **Apply tools** — run tests, checkers, or commands with predictable, machine-checkable output wherever possible. Prefer this over the agent's self-report.
-   **Structured checklists** — for larger tasks, an explicit itemized feature/requirement list (not a vague sense of "done") prevents premature completion claims; each item gets its own verification step before being marked done.
-   **Adversarial prompting** — reviewer subagents whose job is to challenge the output, not confirm it.
-   **Reflection** — the agent should recognize its own failure to reach the goal without deviating from given instructions just to force a fix. On failure, the correct output is an honest acknowledgment: either ask the user to reconsider the approach, or explain concretely why the goal can't be achieved as specified — not a silent workaround.

## Sources

-   [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — Anthropic
-   [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) — Anthropic
-   [Writing effective tools for AI agents](https://www.anthropic.com/engineering/writing-tools-for-agents) — Anthropic
-   [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) — Anthropic
-   [Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices) — Anthropic