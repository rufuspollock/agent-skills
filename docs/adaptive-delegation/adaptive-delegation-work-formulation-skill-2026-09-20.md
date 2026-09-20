# Adaptive Delegation / Work Formulation Skill

## Purpose

Design a lightweight meta-skill that takes a rough statement of work and turns it into delegation-ready work for an AI agent with the minimum necessary human involvement.

The aim is not merely to improve prompts. It is to create a reliable formulation process that decides:

1. what problem is actually being solved;
2. which problem-solving grammar is appropriate;
3. how much analysis or planning the task deserves;
4. which judgments genuinely require the human;
5. when the work is sufficiently specified to run mostly unattended.

## Situation

AI systems are increasingly capable of both helping to plan work and executing substantial tasks autonomously.

There is often far more work available than a person can execute personally, so the potential value of AI comes from parallelism: formulate work once, delegate it, and let multiple agents execute concurrently.

## Complication

Much of the available work is not delegation-ready.

In practice, many AI sessions mix formulation and execution. The human starts with a fuzzy intention, the agent begins acting, and the human then has to remain present to answer questions, correct assumptions, redirect the approach, or make judgments that should have been surfaced earlier.

This means:

- work is under-specified before execution begins;
- human attention remains coupled to execution;
- agents cannot reliably be left running;
- the user may be overloaded with work while still lacking a queue of tasks that can actually be delegated;
- the potential parallelism of autonomous agents is therefore lost.

A related operational problem occurs when an executing agent becomes blocked. Rather than leaving the blockage buried inside a chat session, the system should externalize it as a small, structured issue that can be surfaced to the human and resolved independently.

## Question

How can we create a lightweight, reliable process that turns fuzzy intentions into well-specified work that AI agents can execute mostly unattended?

More specifically:

> Given a rough description of a task or problem, how should an AI determine what kind of formulation process is needed, how deep that process should go, and what questions or judgments it needs from the human before autonomous execution?

## Working hypothesis

The main bottleneck is often formulation rather than execution.

A useful system therefore needs a meta-delegation layer that:

- diagnoses the task;
- selects an appropriate problem-solving grammar;
- selects an appropriate depth of analysis;
- performs as much formulation itself as possible;
- asks the human only for high-value judgments or missing constraints;
- produces an explicit execution contract;
- and only then hands the task to an executing agent.

The central design goal is to reach a state of:

> **safe to leave running**

rather than merely producing a better prompt.

## Core design concept

Treat the skill as a router over two dimensions:

### 1. Problem-solving grammar

The system chooses a method suited to the shape of the work.

Candidate grammars include:

- **SCQH — Situation / Complication / Question / Hypothesis**  
  Useful as a general front door for clarifying why a problem exists and what question is actually being answered.

- **Issue trees / hypothesis-driven problem solving**  
  Appropriate for analytical, strategic, diagnostic, and research problems that need decomposition into mutually intelligible subquestions.

- **Natural Planning Model (Getting Things Done)**  
  Useful for bounded practical projects: purpose and principles → outcome vision → brainstorming → organizing → next actions.

- **Shape Up / shaping**  
  Useful for product and software feature work where the task is to define appetite, boundaries, risks, rabbit holes, and a shaped solution before implementation.

- **Superpowers-style brainstorming / design / implementation planning**  
  Useful for code-level implementation once the product or feature problem is sufficiently framed.

- **Logical Thinking Process / Theory of Constraints**  
  Useful for difficult systemic or causal problems where undesirable effects, conflicts, assumptions, and causal structure need to be surfaced explicitly. This is comparatively heavyweight.

- **Decision analysis**  
  Useful where the task is fundamentally a choice among alternatives rather than a research problem: define criteria, constraints, uncertainties, trade-offs, and what information could materially change the decision.

- **Direct execution**  
  For small, obvious, low-risk tasks where further formulation would cost more than it saves.

These should not be treated as mutually exclusive. For example, a large software initiative might use:

SCQH → issue tree → shaping → Superpowers design → implementation plan.

### 2. Depth / intensity

The system should also choose how much formulation is warranted.

A possible scale:

- **Level 0 — Execute directly**  
  Intent and output are obvious; low cost of error.

- **Level 1 — Confirm**  
  Agent proposes its interpretation and intended output; human gives a quick yes/no/correction.

- **Level 2 — Lightweight formulation**  
  A few high-value questions, then a short plan and execution contract.

- **Level 3 — Structured planning**  
  Explicit framing, decomposition, assumptions, options, risks, and work plan.

- **Level 4 — Deep design / investigation**  
  Multi-stage analysis for high-value, ambiguous, irreversible, systemic, or technically difficult work.

The skill should infer the lowest level likely to produce a reliable autonomous handoff.

## Desired interaction

A rough interaction could look like this:

1. Human gives a rough intention.
2. Skill restates the apparent objective and deliverable.
3. Skill classifies the task.
4. Skill chooses a problem-solving grammar and depth.
5. Skill does whatever analysis it can without asking the human.
6. Skill asks only questions whose answers materially affect the work.
7. Skill produces a delegation-ready specification.
8. Human approves or corrects it.
9. An execution agent runs independently.
10. Execution only returns to the human for explicit exceptions.

The skill should avoid asking questions merely because information is missing. It should distinguish between:

- information it can infer safely;
- information it can research itself;
- reversible assumptions it can make explicitly;
- and judgments that genuinely require the human.

## Delegation-ready output

A handoff should normally contain:

- objective;
- desired deliverable;
- relevant context;
- constraints;
- success / done criteria;
- assumptions;
- chosen approach;
- work plan;
- resources or sources to use;
- decisions already made;
- unresolved questions, if any;
- escalation conditions;
- expected artifact or output location.

The exact schema should remain lightweight and adapt to task type.

## Appendix: blocker / attention queue

During autonomous execution, an agent should not simply pause inside its session when it needs human input.

Instead it should create a small structured blocker item — for example a Bead or issue — containing:

- task/work-item ID;
- concise blocker statement;
- why execution cannot or should not continue;
- exact decision or information needed from the human;
- relevant context sufficient to answer without reading the full session;
- recommended/default option where appropriate;
- what work will resume once resolved.

That blocker can then appear in a centralized **Needs Attention** queue.

Once resolved, the execution session should be able to resume without the human reconstructing the entire conversation.

This blocker mechanism belongs to the wider orchestration system rather than the formulation skill itself, but the formulation skill should specify escalation rules up front.

## Relationship to the wider system

This skill is one subsystem of a larger AI work-management architecture:

1. **Work graph / orchestration** — track all work items, dependencies, status, sessions, outputs, and what is ready to run.
2. **Formulation / meta-planning** — turn fuzzy intentions into delegation-ready work. This document focuses here.
3. **Autonomous execution / exception handling** — run work, record outputs, surface blockers, and resume automatically after resolution.

The key principle is to manage the **work graph**, not the fleet of agents. Agents are executors attached to work items.

## Design assignment for the next agent

Design a proof of concept for this adaptive delegation / work-formulation skill.

Do not jump directly to implementation.

### Phase 1 — Critique and discovery

1. Critique the SCQH and working hypothesis above.
2. Identify important assumptions or missing dimensions.
3. Ask the human only the highest-value questions needed to improve the design.
4. Research existing approaches to:
   - AI task formulation and specification;
   - agent routing and metareasoning;
   - delegation and management theory;
   - mission command / commander's intent;
   - issue-tree and hypothesis-driven problem solving;
   - planning frameworks;
   - software/product shaping and specification;
   - autonomous-agent escalation and human-in-the-loop patterns.
5. Distinguish existing solved components from genuinely missing functionality.

### Phase 2 — Design

Propose:

- the task-classification dimensions;
- the routing logic for choosing a grammar;
- the routing logic for choosing depth;
- the candidate grammar library;
- the minimum common output schema;
- rules for asking versus inferring;
- rules for assumptions;
- escalation / blocker semantics;
- examples across several task types;
- failure modes and safeguards.

Prefer a small composable architecture over a large monolithic prompt.

### Phase 3 — Prototype

Build the smallest useful proof of concept.

It should accept a rough task such as:

- “I may need to replace my son's iPad.”
- “We need to decide what workflow automation tool Life Itself should use.”
- “Design a substantial new software feature.”
- “Investigate why this organizational process keeps failing.”

For each, it should select an appropriate formulation strategy and depth, formulate as much as possible itself, ask only material questions, and produce a delegation-ready work package.

### Phase 4 — Evaluation

Develop a small benchmark of representative tasks and evaluate:

- human minutes spent formulating;
- number of human interruptions during execution;
- proportion of tasks reaching “safe to leave running”;
- unnecessary questions asked;
- failures caused by hidden assumptions;
- quality of final deliverable;
- whether the selected grammar/depth was appropriate.

The primary success metric is not prompt quality. It is **reduction in human attention required per successfully completed unit of work**.
