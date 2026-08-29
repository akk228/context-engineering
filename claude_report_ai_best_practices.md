# Context Engineering for AI Agents: Failure Modes, Best Practices, and Practical Tips

## TL;DR
- **Context is a finite resource with diminishing returns, and managing it — not prompt wording or model choice — is now the primary determinant of agent reliability.** Every frontier model exhibits "context rot" (measurable degradation as input grows, per Chroma's 18-model study), errors compound super-linearly over long horizons (METR: near-100% success under ~4 minutes of human-equivalent work, under 10% over ~4 hours), and the fixes are architectural: compaction, structured note-taking/external memory, just-in-time retrieval, sub-agent isolation, and verification loops.
- **The field has converged on a shared toolkit** — Anthropic's compaction/note-taking/sub-agent triad, LangChain's write-select-compress-isolate framework, Drew Breunig's four failure modes (poisoning, distraction, confusion, clash) and six fixes, and Cognition's "share context, avoid conflicting decisions" — even as Anthropic and Cognition disagree on when to use multiple agents.
- **For your three priority use cases, accuracy comes from disciplined context curation, not speed:** long sessions need checkpointing + compaction + external memory files + keeping errors visible; discovery/research needs orchestrator-worker fan-out with isolated sub-agent contexts and "start wide, then narrow"; orchestration needs deterministic control flow around LLM decisions, single-threaded writes, and adversarial verification between steps.

---

## Key Findings

1. **"Context engineering" has displaced "prompt engineering" as the core discipline.** Anthropic defines it as "the set of strategies for curating and maintaining the optimal set of tokens (information) during LLM inference." Cognition's Walden Yan calls it "effectively the #1 job of engineers building AI agents." This is a general guideline for the *practice* of context engineering, not a description of a specific feature built into Anthropic's harness/product (e.g. Claude Code). It's a principle meant to be applied by anyone designing agent systems, regardless of framework: find "the smallest possible set of high-signal tokens that maximize the likelihood of some desired outcome."

2. **Context rot is universal and mechanistic.** Chroma's "Context Rot: How Increasing Input Tokens Impacts LLM Performance," published July 14, 2025 by Kelly Hong, Anton Troynikov & Jeff Huber, tested 18 frontier models (GPT-4.1, Claude 4, Gemini 2.5, Qwen3) and found "models do not use their context uniformly; instead, their performance grows increasingly unreliable as input length grows" — even on trivial tasks. The cause is the transformer's n² attention: every token attends to every other, so attention gets "stretched thin," and models are trained mostly on shorter sequences. Anthropic frames this as a finite "attention budget."

3. **Long-horizon failure is an execution problem, not a reasoning problem.** METR's time-horizon study (Kwa et al., "Measuring AI Ability to Complete Long Software Tasks," arXiv:2503.14499) found that "the length of tasks…that generalist frontier model agents can complete autonomously with 50% reliability has been doubling approximately every 7 months for the last 6 years" (a doubling time of ~212 days from 2019 to early 2025; task length correlates with success at R²≈0.83). Sinha et al., "The Illusion of Diminishing Returns: Measuring Long Horizon Execution in LLMs" (arXiv:2509.09677, NeurIPS 2025) shows per-step accuracy actually *rises in error* as tasks lengthen due to "self-conditioning": "models become more likely to make mistakes when the context contains their errors from prior turns. Self-conditioning does not reduce by just scaling the model size. In contrast, recent thinking models do not self-condition."

4. **The taxonomy of context failure is stable.** Drew Breunig's four modes — poisoning, distraction, confusion, clash — are cited by essentially every vendor. His six fixes (RAG, tool loadout, quarantine, pruning, summarization, offloading) map onto LangChain's four strategies (write, select, compress, isolate).

5. **Multi-agent architectures are powerful but contested.** Per Anthropic's June 2025 "How we built our multi-agent research system," a Claude Opus 4 lead + Claude Sonnet 4 subagents "outperformed single-agent Claude Opus 4 by 90.2% on our internal research eval," with token usage explaining 80% of performance variance — but multi-agent systems used ~15× the tokens of chat and only win on parallelizable, breadth-first tasks. Cognition argues the opposite for most work: keep writes single-threaded because parallel agents make conflicting implicit decisions.

6. **Prompt injection / the "lethal trifecta" is the dominant security failure.** Simon Willison's framework: an agent with (1) access to private data, (2) exposure to untrusted content, and (3) an external-communication/exfiltration channel is exploitable regardless of model alignment. Real incidents hit Writer.com, GitLab Duo, and ChatGPT Operator.

---

## Details

# REPORT 1 — Issues of Agents and How to Overcome Them

### Layer A — Harness / Framework Design Failures

**A1. Context-window overflow and naive accumulation.** Agent loops append every tool result and message; long-running tasks blow past the window, balloon cost/latency, and degrade performance. Anthropic's Claude Code runs "auto-compact" at ~95% of the window. *Mitigations:* compaction (summarize + reinitialize), tool-result clearing (a launched Claude Developer Platform feature — "once a tool has been called deep in the message history, why would the agent need to see the raw result again?"), and message trimming (hard-coded heuristics like dropping oldest messages, or trained pruners like Provence).

**A2. Bloated tool sets / poor tool orchestration.** Anthropic: "One of the most common failure modes we see is bloated tool sets that cover too much functionality or lead to ambiguous decision points about which tool to use. If a human engineer can't definitively say which tool should be used in a given situation, an AI agent can't be expected to do better." Breunig notes that beyond ~20 tools some models get confused. *Mitigations:* "tool loadout" (enable only task-relevant tools), RAG over tool descriptions (shown in one paper to improve tool-selection accuracy 3-fold; LangGraph Bigtool applies semantic search over tools), minimal viable tool sets with non-overlapping, unambiguous descriptions.

**A3. Memory systems that are opaque or unreliable.** Auto-generated long-term memories (ChatGPT, Cursor, Windsurf) can inject wrong or unwanted context — Simon Willison's example of ChatGPT injecting his location into a requested image. Breunig warns against "black box memory systems; agents need transparent, auditable, and steerable memory to be safe for production." *Mitigations:* file-based memory the agent (and human) can read (Claude Code's CLAUDE.md; Anthropic's memory tool beta); gate what gets *written* to memory, not just what's read.

**A4. Framework over-abstraction.** Practitioner consensus (and Anthropic's "Building Effective Agents"): "the most successful implementations use simple, composable patterns rather than complex frameworks." Heavy frameworks obscure what actually enters the context window. *Mitigation:* "do the simplest thing that works"; use low-level orchestration (LangGraph exposes nodes/state so you control context at each step) or raw API loops before reaching for multi-agent frameworks.

### Layer B — Underlying Model Failures

**B1. Context rot / attention dilution.** (See Key Finding 2.) Degradation is non-uniform: Chroma found needle-question semantic similarity, distractors, and even haystack *structure* matter — models did *better* on shuffled haystacks than on logically coherent ones. For 1M-token models, a clearly observable effect often begins around 300K–400K tokens. *Mitigation:* treat context as finite; keep high-signal tokens only; just-in-time retrieval over upfront dumping.

**B2. Error compounding / self-conditioning over long horizons.** Even 99% per-step accuracy decays to ~36.6% over 100 steps geometrically; empirically it's worse than geometric because per-step accuracy *degrades* as the trajectory lengthens (self-conditioning; Sinha et al. 2025). METR: near-100% success on ~4-min tasks, <10% on ~4-hr tasks. *Mitigations:* decompose into short sub-tasks with clean contexts; verification/checkpointing; "thinking"/extended reasoning (shown to fix self-conditioning — thinking models "do not self-condition").

**B3. Context poisoning.** A hallucination or bad tool output enters context and is then referenced as truth on every subsequent turn. The DeepMind Gemini 2.5 report documented a Pokémon-playing agent that hallucinated game state, poisoning its goals and producing "nonsensical strategies and repeated behaviors in pursuit of a goal that cannot be met." *Mitigations:* validate before writing to persistent state; sub-agent quarantine; explicit conflict-resolution policy in the system prompt.

**B4. Context distraction and confusion.** Distraction: the context grows so long the model over-focuses on accumulated history and neglects training knowledge (leaning on past actions, repeating them). Confusion: superfluous content the model feels compelled to use produces low-quality output. *Mitigations:* summarization, pruning, tool loadout, recitation of goals.

**B5. Context clash.** New information/tools conflict with earlier context — acute with third-party MCP tools whose descriptions clash with your prompt. Laban et al. (2025): models "commit early to a reading and struggle to let go," fumbling tasks when information arrives muddled across turns that they solve cleanly in one pass. *Mitigations:* isolation/quarantine, single-writer discipline, conflict-resolution policies.

**B6. Poor tool-call selection, sycophancy, hallucination.** Bad tool descriptions "send agents down completely wrong paths." Sycophancy and hallucination compound in loops. *Mitigations:* Anthropic's tool-testing agent (rewrites flawed tool descriptions → 40% faster task completion for later agents); LLM-as-judge and human review; keeping errors visible so the model updates its prior (Manus).

### Layer C — Deployed Application / Product Failures

**C1. Prompt injection / lethal trifecta.** (See Key Finding 6.) *Mitigations:* remove one leg of the trifecta; architectural separation (dual-LLM / CaMeL pattern from Google DeepMind); Meta's "Agents Rule of Two" (satisfy at most two of: untrusted input, sensitive-system access, external state change); sandboxing/sanitizing untrusted content. Critically, filters alone do not work: Nasr et al., "The Attacker Moves Second: Stronger Adaptive Attacks Bypass Defenses Against LLM Jailbreaks and Prompt Injections" (arXiv:2510.09023, Oct 2025; authors incl. Carlini and Tramèr) found adaptive attacks "bypass 12 recent defenses…with attack success rate above 90% for most; importantly, the majority of defenses originally reported near-zero attack success rate," and a 500+-participant human red-teaming competition "achieve[d] 100% ASR across all scenarios." Architectural constraints beat filters.

**C2. Poor state management / errors compounding in production.** Anthropic: "Agents are stateful and errors compound… minor system failures can be catastrophic." *Mitigations:* durable execution that resumes from the failure point (not restart), retry logic, regular checkpoints, "rainbow deployments" to avoid breaking in-flight agents, full production tracing.

**C3. Orchestration failures between multi-agent systems.** Anthropic's early system spawned 50 subagents for simple queries, searched endlessly for nonexistent sources, and had subagents "distracting each other with excessive updates"; one subagent investigated the 2021 automotive chip crisis while two others duplicated work on 2025 supply chains. Cognition's Flappy Bird example: parallel subagents build a Super Mario-style background and a mismatched bird. *Mitigations:* detailed delegation (objective, output format, tool guidance, boundaries), effort-scaling rules, single-threaded writes, share full agent traces not just messages.

**C4. The "game of telephone" in hand-offs.** Information degrades passing through a coordinator. *Mitigation:* Anthropic's pattern — subagents write outputs to a filesystem and pass lightweight references back, bypassing the coordinator for large artifacts.

---

# REPORT 2 — Best Practices for Context Engineering

### 1. System prompt design — the "right altitude"
Anthropic: aim for the "Goldilocks zone" between brittle hardcoded if-else logic and vague high-level guidance. "Specific enough to guide behavior effectively, yet flexible enough to provide the model with strong heuristics." Organize into distinct sections (`<background_information>`, `<instructions>`, `## Tool guidance`, `## Output description`) with XML tags or Markdown headers. Strive for the *minimal set of information that fully outlines expected behavior* — minimal ≠ short. Start minimal with the best model, then add instructions based on observed failure modes.

### 2. Tool design and curation
Tools are "the contract between agents and their information/action space." Make them self-contained, robust to error, token-efficient, and unambiguous. Curate a minimal viable set; prune overlapping tools. Use tool loadout / RAG-over-tools when the catalog is large.

### 3. Few-shot examples
"Examples are the 'pictures' worth a thousand words." Do NOT stuff a laundry list of edge cases. Curate "diverse, canonical examples" that portray expected behavior.

### 4. Retrieval strategy — RAG vs long-context vs hybrid, and just-in-time
The field is shifting from upfront embedding-based retrieval to "just-in-time" strategies: keep lightweight identifiers (file paths, queries, links) and load data at runtime via tools. Claude Code uses glob/grep to navigate just-in-time, avoiding stale indexes. Metadata (folder names, timestamps, `test_utils.py` vs `src/core_logic/`) provides signal. Enables "progressive disclosure." Trade-off: runtime exploration is slower. The *hybrid* strategy — some data upfront (CLAUDE.md), rest explored just-in-time — suits less-dynamic domains like legal/finance. RAG is "very much alive": a focused RAG injection beats a full-document dump on retrieval tasks. Anthropic's multi-agent researcher uses dynamic multi-step search rather than static RAG chunk retrieval.

### 5. Memory architectures — working vs persistent
- **Short-term / working memory:** LangGraph thread-scoped state via checkpointing; scratchpads (a state field or a file).
- **Long-term / persistent memory:** across sessions — profiles, rules files, or collections (LangGraph store, LangMem). Memory types: episodic (few-shot examples), procedural (instructions), semantic (facts). Selection from large memory collections uses embeddings or knowledge graphs and remains hard.

### 6. Compaction and summarization
Anthropic's compaction: summarize a near-full window, reinitialize with the summary + the five most recently accessed files. "The art of compaction lies in the selection of what to keep versus what to discard." Tune the compaction prompt: maximize recall first, then improve precision. Safest low-touch form: tool-result clearing. Summarization can be recursive or hierarchical, applied at tool-call boundaries or agent-agent hand-offs. Cognition fine-tuned a dedicated model for compression — "hard to get right."

### 7. Structured note-taking / external memory (offloading)
Agent writes notes to persistent storage outside the window, pulled back later. Claude Code's to-do list; a NOTES.md; Claude playing Pokémon maintained tallies "for the last 1,234 steps I've been training my Pokémon in Route 1." Manus's todo.md is rewritten continuously as *recitation* — pushing the goal into recent attention to fight "lost-in-the-middle."

### 8. Multi-agent isolation / sub-agent architectures
Isolation splits context across sub-agents with their own windows. Anthropic's orchestrator-worker: lead agent holds a high-level plan; subagents explore with tens of thousands of tokens each but return 1,000–2,000-token distilled summaries. "Separation of concerns." Also achievable via sandboxed CodeAgents (HuggingFace) and via a typed state object that isolates fields from the LLM until needed.

### 9. The failure taxonomies (mental models)
- **Breunig's four modes:** poisoning, distraction, confusion, clash. Useful axis: poisoning/distraction are problems of *what stays*, confusion of *what enters*, clash of *what coexists*.
- **LangChain's four strategies:** Write (save outside window), Select (pull in the right info), Compress (keep only needed tokens), Isolate (split across contexts). Karpathy's framing: the LLM is a CPU, its context window is RAM; context engineering is the OS curating RAM.
- **Cognition's two principles:** (1) Share context, and share full agent traces, not just individual messages; (2) Actions carry implicit decisions, and conflicting decisions carry bad results.

### 10. Manus production principles
Design around the KV-cache (stable prompt prefixes, append-only context, deterministic serialization — a ~10× cost difference cached vs uncached); mask tool logits instead of adding/removing tools (avoids cache invalidation); use the filesystem as unbounded external context; recite goals; keep errors in context so the model learns.

---

# REPORT 3 — Practical Tips and Tricks

### Use Case 1 — Long agent sessions maximizing ACCURACY

1. **Checkpoint and commit constantly.** Anthropic uses durable execution that resumes from the failure point plus retry logic and regular checkpoints. In Claude Code, commit at least once per task step / roughly hourly. Treat every session as disposable — never let a long session be your only record.
2. **Use the "Document & Clear" pattern.** Dump decisions/progress to files, then clear context. Context loss is "the failure mode that hits hardest."
3. **Externalize state to memory before the window fills.** Anthropic's LeadResearcher "saves its plan to Memory to persist the context, since if the context window exceeds 200,000 tokens it will be truncated."
4. **Compact deliberately, not reflexively.** Preserve architectural decisions, unresolved bugs, implementation details; discard redundant tool outputs. Keep the plan file re-readable from the workspace.
5. **Recite the goal.** Maintain and continuously rewrite a todo.md so the objective stays in recent attention (Manus).
6. **Keep errors visible.** Leaving failed actions and stack traces in context lets the model update its prior away from repeating mistakes (Manus). But note self-conditioning: if errors accumulate unchecked, per-step accuracy falls — so pair error-visibility with periodic clean-context restarts, or use a thinking model (which does not self-condition).
7. **Delegate to sub-agents to preserve the main thread.** Run noisy work (log parsing, HTML scrapes, DB dumps) inside a sub-agent that returns only the distilled answer (quarantine). The lead never sees raw bytes.
8. **Turn on extended thinking** for planning steps — it improves instruction-following and mitigates self-conditioning.
9. **Instrument token usage / KV-cache hit rate** — you can't optimize what you don't measure (LangSmith tracing; Manus's #1 metric).

### Use Case 2 — Accurate discovery / research sessions

1. **Use the orchestrator-worker pattern for breadth.** Lead agent plans and spawns 3–5 subagents with isolated contexts, each chasing an independent thread; synthesize with a separate citation pass. 90.2% better than single-agent on Anthropic's research eval — but only for parallelizable, breadth-first questions, and at ~15× token cost.
2. **Scale effort to query complexity.** Embed explicit rules: simple fact-finding = 1 agent, 3–10 tool calls; comparisons = 2–4 subagents, 10–15 calls each; complex research = 10+ subagents with divided responsibilities. Prevents both under- and over-investment.
3. **Delegate precisely.** Give each subagent an objective, output format, tool/source guidance, and clear boundaries — or they duplicate work and leave gaps.
4. **Start wide, then narrow.** Mirror expert researchers: broad queries first, evaluate, then drill down. Agents default to overly long, specific queries that return few results.
5. **Avoid premature conclusions.** Use the OODA loop (observe, orient, decide, act) with interleaved thinking after each tool result to evaluate quality and gaps before the next query. Guard against early commitment (context clash / stickiness).
6. **Elicit accurately, avoid hallucination.** Prefer primary sources — Anthropic's testers found early agents chose SEO content farms over authoritative PDFs/blogs until source-quality heuristics were added. Use an LLM-as-judge rubric: factual accuracy, citation accuracy, completeness, source quality, tool efficiency.
7. **Have subagents write to a filesystem** and pass references back to minimize the "game of telephone."
8. **In plan mode (Claude Code): explore read-only first, then plan, then implement.** "Skip explore and plan and Claude jumps straight to code, which is exactly when it builds the wrong thing confidently."

### Use Case 3 — Better workflow orchestration

1. **Pick the simplest pattern that fits** (Anthropic's five):
   - **Prompt chaining** — sequential steps with validation gates between.
   - **Routing** — classify input, dispatch to specialized handlers.
   - **Parallelization** — sectioning (split) or voting (consensus).
   - **Orchestrator-workers** — dynamic decomposition at runtime.
   - **Evaluator-optimizer** — generator + separate critic loops until a quality threshold passes.
2. **Distinguish workflows from agents.** Workflows = LLMs/tools on predefined code paths (predictable, controllable); agents = LLMs directing their own process (flexible, less predictable). Use deterministic code for high-frequency, low-complexity work.
3. **Deterministic control flow around LLM decisions.** Cognition enforced "subagents can't spawn their own subagents" in the orchestration layer, not by prompt begging. Combine AI adaptability with deterministic safeguards (retries, checkpoints, guardrails).
4. **Keep writes single-threaded** (Cognition's current stance): additional agents should "contribute intelligence rather than actions." Rule out architectures that violate: share context; avoid conflicting decisions.
5. **Insert adversarial verification between steps.** Zhang et al., "On the Resilience of LLM-Based Multi-Agent Collaboration with Faulty Agents" (ICML 2025), added an "Inspector, an additional agent to review and correct messages, recovering up to 96.4% [of] errors made by faulty agents"; a hierarchical A→(B↔C) structure showed the lowest performance drop (5.5% vs. 10.5% and 23.7% for other structures). Independence of the verifier is the design requirement.
6. **Use state machines / logit masking** to constrain the action space by phase (Manus) without invalidating the KV-cache.
7. **Human-in-the-loop checkpoints** at discrete state changes; evaluate end-state, not every intermediate step, for state-mutating agents.
8. **Error recovery:** resume from failure point; let the model know a tool is failing and adapt; use rainbow deployments; full tracing for non-deterministic debugging.
9. **Handoffs:** share full agent traces, not just final messages; compress at agent-agent boundaries (Cognition fine-tuned a model for this).
10. **When NOT to go multi-agent:** tasks needing shared context or with many inter-agent dependencies (most coding). Anthropic: "domains that require all agents to share the same context or involve many dependencies between agents are not a good fit for multi-agent systems today."

---

## Recommendations

**Stage 1 — Baseline (any agent project).** Start single-agent, single-threaded, with the best available model and a minimal, right-altitude system prompt. Add observability/token tracing (LangSmith or equivalent) and a small eval set (~20 real queries) before optimizing. Benchmark: if a prompt tweak moves success 30%→80% on 20 cases, you don't yet need more evals.

**Stage 2 — Add context hygiene when sessions lengthen.** Introduce, in order: (1) external memory/scratchpad files, (2) compaction with a tuned prompt, (3) tool loadout / pruning if you have >~20 tools or ambiguous tool choice, (4) just-in-time retrieval. Threshold to act: token usage climbing toward ~50% of window on routine turns, or accuracy visibly decaying mid-session (context rot — watch around 300K–400K tokens on 1M-token models, but often much earlier).

**Stage 3 — Isolate and verify for accuracy-critical work.** For research/discovery, adopt orchestrator-worker fan-out with isolated subagent contexts and effort-scaling rules. For any long-horizon or high-stakes pipeline, insert independent adversarial verification between steps and checkpoint durably. Threshold to add multi-agent: the task decomposes into genuinely independent parallel threads AND its value justifies ~15× token cost. If agents share context or have many dependencies, stay single-threaded.

**Stage 4 — Security before production.** Audit for the lethal trifecta; if all three legs are present, remove one (usually gate the exfiltration channel or sandbox untrusted content) or apply the Agents Rule of Two. Do not rely on injection filters alone (Nasr et al. found ~100% human bypass of example-based defenses). Add human approval for irreversible actions; back up irreplaceable files before granting write access.

**What would change these recommendations:** If models continue improving, expect to need *less* prescriptive engineering (Anthropic: "smarter models require less prescriptive engineering"). If cross-agent context-passing gets solved (Cognition's prediction), multi-agent becomes viable for more (incl. coding). If context rot is architecturally solved, the just-in-time/compaction emphasis relaxes — but treat that as a forecast, not a plan.

## Caveats
- **Vendor perspective.** Anthropic sources naturally reflect Claude/Claude Code; LangChain sources promote LangGraph/LangSmith. The underlying principles are framework-agnostic (write/select/compress/isolate apply to any loop) but tooling claims are self-interested.
- **The multi-agent disagreement is genuine and unresolved.** Anthropic (multi-agent wins on research) and Cognition ("don't build multi-agents") published near-simultaneously; both agree context engineering is paramount. The reconciliation — multi-agent for breadth-first parallel research, single-threaded for dependent work like coding — is the current best synthesis, not settled fact.
- **Quantitative figures carry conditions.** The 90.2% improvement is Anthropic's *internal* eval; "token usage explains 80% of variance" was specifically on BrowseComp; the 15×/4× token multipliers are Anthropic's data. METR's numbers are for software tasks and its extrapolations ("automating month-long tasks within 5 years") are explicitly forecasts, not measurements.
- **Some sources are secondary.** Where primary posts (Anthropic, Cognition, LangChain, Chroma, Manus, Breunig, METR, and the arXiv papers cited) were available they were used directly; a few operational figures were confirmed via aggregation and should be double-checked against the primary PDFs for high-stakes use.
- **Fast-moving field.** Model versions, features (memory tool, context editing), and even framework APIs cited here are from 2024–2026 and will shift.