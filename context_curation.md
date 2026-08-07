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

Outlining an agent's anatomy helps us to understand how to prompt up an efficient agentic workflow.

## Prompt constitution

Below one may find a list of what I would call a prompt constitution. However, one may find some of these constituents overlapping, the separatino helps to achieve a better separation of concerns.

1.  Goal definition or ploblem statement.
2.  Execution instructions.
3.  Tooling.
4.  Guardrails.
5.  Escalation policy.
6.  Error handling.
7.  Fallback mechanisms.
8.  Verification.

[NOTE]: let's agree that we couldn't care less about these constituents of a promp in case we are just shooting shit to see what sticks or when we have a request with an extremly small scope. Rememeber, we are here for a bigger fish, which is complex workflow orchestration.

### 1. Goal definition and problem statement

1.  Require clear declaration.
2.  Must always be a part of the main context.

<details><summary>Techniques</summary>

[WARNING]: maybe the stuff below shouldn't be here, maybe it's better to include it in a separate part on agent orchestration techinques.

How to adress the main goal that you are instructing an agent to?

**1. Use sub agents**

We have the main agent that is an orchestrator. It 

-   steers the conversation based on the main goal
-   provides high-level instructions
-   provides user output
-   Takes executive descisions on the execution process


**Open questions**

*   Should we maybe use the main agent (the one we interact with) - meta-agent, and then span another agent as an orchestrator?
*   How often the orchestrator agent should inquery a human if any of its subagents have reached an impass and confidently state that they can not achieve the required goal? (falls into error handling, escalation policy)
*
**2. Use memory**

Emphasize that the goal must be stored. We must never push out the goal from the main context.

**Open questions**

Q:  How to preserve? Should we just let the main goal be a part of the orchestator's context or store it in the permanent memory.
A:  Goal is the part of main context. And orchestrator must always rememember it's main goal otherwise how would it steer a conversation.

Q:  Should we let orchestrator maintain the memory as a part of context?
A:  I don't think so. I belive an agent should just decide what kind of storage is enough for it weather it is just an internal plan or some database.

**Identify the main agent/orchestator**

-   Don't relu on persona, use "Jekyll and Hyde"
-   Explain the main agent what do you want from it. Persona should be something that just better communicates the agent its goal.
-   Either user or the main agent itself must communicate to subagents thier goal as well.
</details>

### 2. Instructions

We provision to an agent a set of instructions that it can epmploy to deterministically perform a task.

How ambiguous should instructions be?
Should we amend them? If yes, what would be the mechanism of amendment?
Should we rely on the user instructions only, or we discover them together with an agent?
Are specs for code instructions in the sense that we use them to steer agents behaviour, or is it something else?
When we define instructions are we implying a CoT or ToT, or agent decides for itself?

### 3. Agent's tooling

Let's start from outline the basic set of tools that we already know might be employed by agents.

*   Skills
*   MCP's
*   CLI
*   Sub-agent creation

**Problems to adress**

*   Should we always instruct the agent to employ sepcific tools?
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

### 4, Guardrails

Guardrails help us to protect ourselves from agent doing things that wasn't meant or instructed to do. E.g.

-   Natural language instructions.
    *   Can be prompt
    *   Can be a custom rule
-   Harness
The ways to set up guardarails may be:

**Natural language instructions**

*   Pros
    1.  Can help to steer the agent better
    2.  Easy to set up
*   Cons
    1.  Can be bypassed, bcs say, some other instructions overrides it.
    2.  Can be thorown out of context or lost there.

**Harness level prohibition**

*   Pros
    1.  Hard to bypass
*   Cons
    1.   to rigid, may lead to babysitting an agent

Are there any other guradarails?


### 5. Escalation policy

We want an agent to avoid

-   Doing stuff that it is not meant to do
-   Provide honest output

What could be the need for escalation

-   agent doesn't have access
-   agent recognized that it is at impass and it needs user's input to get further instructions
-   agent wants to reconsider approach (also an impass?)

Are there any other cases where escalation to a human make sense?