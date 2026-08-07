# Context Engineering

## Goal

The result of this work is meant to help to curate better prompts that are aimed to be put together as a skill for Atutomated Prompt Optimization.

The result is meant to be a skill that helps engineer to work together with an agent to create better initial prompt and guide an engineer through a process of complex task execution.

**Problem statement**

-   Create a workflow that achieves the best results based on a specific initital requirments.
-   Create a workflow that overcomes various agent's constraints and flaws, such as:
    *   Finite context window
    *   Context pollution
    *   False/unneccesary context propagation
    *   Hallucinations

**Outcome**

1. Skill that helps to write better prompts and follow context engineering instead of simple prompting.
2. Self-guiding framework that self-corrects and self-improves over time.

## Task variety

The agentic workflow is heavily dependant on the type of problem that a prompt engineer is trying to solve. The workflows that I'm going to address are aminly the ones that I'm targeting for my own purposes and not the exacerbating list.

The tasks to be addressed are:

*   Execution
*   Discovery
*   Combination(?)

## Agent's anatomy

The agent is:

1. An interface to a underlying model.
2. Agentic loop delivering the agentic reasoning.
3. A set of tools it can employ.
4. A user's servant :)



### Agent's tooling

**Problems to adress**


### Agent's tooling and skill set

*   Should we always instruct the agent to employ sepcific tools, skills?
*   How aware agent is of it's own capabilities?

**Proposition**

1. Instruct.
2. Ask.
3. Let it discover along the way.

1. **Instructing the agent**

**Pros**

-   Deterministic.
-   Structured and predictable - lets an engineer to understand what's going on better.
-   More secure.

**Cons**

-   Narrow - you know worse than an agent (likely)
-   Rigid.

2. **Asking**

**Pros**

-   Agile - combine knowledge of your own and agent's.
-   Still deterministic - you know the resulting tool set that is at your disposal.
-   Also predictable.

**Cons**

-   Rigidity -  doesn't allow changing within a cahnging scope. Meaning that is bad for discovery, but ok when engineer understands the scope well, and the task is well structured.

3. **Agent discovers itself**

**Pros**

-   Best for search, allows agent to adapt based in chnaging scope.


**Cons**

-   Less deterministic.
-   Not secure, unless extra permissions are enabled.
