# AI Work Orchestration — Overall Vision

## Vision

The goal is to make it possible to operate many AI agents in parallel without having to manage their individual chat sessions.

The fundamental object should be **work**, not the agent.

A persistent work graph should show what exists, what is ready, what is running, what is blocked, what needs human attention, and what has completed. Agent sessions are temporary execution contexts attached to nodes in that graph.

## Three subsystems

### 1. Work graph and orchestration

Maintain a persistent graph of work items including:

- objective and deliverable;
- parent/child relationships and dependencies;
- status;
- priority;
- assigned agent/session;
- artifacts produced;
- blockers;
- next action;
- whether human attention is required.

The system should make it easy to answer:

- What work exists?
- What is ready to run?
- What is currently running?
- What has finished?
- What is blocked?
- What needs me?
- What becomes runnable if I resolve a blocker?

### 2. Meta-planning and work formulation

Most rough intentions are not immediately delegation-ready.

A formulation layer should take a fuzzy intention and decide:

- what problem is actually being solved;
- what problem-solving grammar to use;
- how much planning is justified;
- which questions genuinely require human judgment;
- and when the work is sufficiently specified to execute unattended.

This is the current design focus and is described in the separate **Adaptive Delegation / Work Formulation Skill** artifact.

### 3. Autonomous execution and exception handling

Once work is well specified, agents should execute independently.

They should:

- record progress and outputs against the work item;
- create sub-work where necessary;
- respect predefined escalation rules;
- externalize blockers instead of burying them in chat;
- surface concise requests in a centralized Needs Attention queue;
- resume automatically when blockers are resolved.

## Overall loop

**Capture → formulate → decompose → schedule → execute → surface exceptions → resolve → resume → complete**

The aspiration is a system in which the human spends most of their time on:

- choosing goals;
- supplying scarce judgment;
- resolving genuine ambiguities;
- reviewing consequential outputs;

rather than continuously supervising execution.
