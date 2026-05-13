# 111 — Magentic-One & Magentic-UI: Generalist Multi-Agent Systems

**Paper.** Adam Fourney, Gagan Bansal, Hussein Mozannar, Cheng Tan, Eduardo Salinas, Erkang Zhu, Friederike Niedtner, Grace Proebsting, Griffin Bassman, Jack Gerrits, Jacob Alber, Peter Chang, Ricky Loynd, Robert West, Victor Dibia, Ahmed Awadallah, Ece Kamar, Rafah Hosn, Saleema Amershi — *Magentic-One: A Generalist Multi-Agent System for Solving Complex Tasks* — arXiv:2411.04468 — Microsoft Research — Nov 2024. Companion: *Magentic-UI: Towards Human-in-the-loop Agentic Systems* (Microsoft Research, July 2025).

**One-line definition.** Magentic-One is **AutoGen's reference multi-agent system**: an **Orchestrator** plans, dispatches, and re-plans, while four specialist agents (**WebSurfer**, **FileSurfer**, **Coder**, **ComputerTerminal**) execute. Magentic-UI extends it with a **human-in-the-loop interface**, making the orchestration visible and steerable in real time.

## Why this paper matters

Where the classic multi-agent paradigm is **role-based collaboration** (engineer + tester + designer, à la [20-metagpt-role-based-workflows](20-metagpt-role-based-workflows.md), [92-chatdev](92-chatdev.md)), Magentic-One is **task-based delegation** with a single orchestrator and tool-specialist workers — closer in spirit to [02-subagent-delegation](02-subagent-delegation.md). It is the cleanest published reference architecture for **generalist** (vs domain-bounded) agentic systems and, crucially, achieves competitive performance on **GAIA**, **AssistantBench**, and **WebArena** with a small, principled cast of agents rather than a large simulated company.

## Problem it solves

1. **Monolithic agents stretch context windows** trying to be planner, browser, coder, and shell user simultaneously.
2. **Generalist tasks span surfaces** — a real GAIA task can require web browsing, file inspection, code execution, and shell commands. One agent has to be all of those, badly.
3. **Re-planning under failure** is awkward in linear ReAct loops; an explicit Orchestrator that owns a plan and revises it cleanly addresses this.
4. **Human oversight at scale.** Magentic-UI's contribution: surface the Orchestrator's plan and let humans intervene at any node, without breaking the underlying autonomy.

## Core idea in one paragraph

The **Orchestrator** receives a task and produces a *high-level plan* + *task ledger* (facts, assumptions, open questions). At each step it selects which specialist agent to invoke, hands them the relevant slice, and reads back results. Specialists are narrow:

- **WebSurfer** — controls a Chromium via a web-action API.
- **FileSurfer** — reads local files (PDFs, spreadsheets, text).
- **Coder** — writes/analyzes code without executing it.
- **ComputerTerminal** — executes code and shell commands in a sandbox.

The Orchestrator updates the ledger, identifies the next sub-task, and continues. On stall or repeated failure, it **re-plans** by editing the ledger or the plan itself. Built on AutoGen's `autogen-core`, Magentic-One is model-agnostic — defaults are GPT-4o for all roles, with experiments using o1-preview for the Orchestrator + Coder.

## Mechanism (step by step)

### (a) The Orchestrator's outer loop

```text
1. Receive task.
2. Build initial plan + ledger (facts, guesses, plan steps, current step).
3. While not done:
   a. Identify the next sub-task and the appropriate specialist.
   b. Dispatch sub-task; await response.
   c. Update ledger with new facts/assumptions.
   d. Check for stall: if no progress in K turns, re-plan or escalate.
4. Synthesize final answer.
```

The ledger is the persistent state object — closer to [12-todo-scratchpad-state](12-todo-scratchpad-state.md) than to a hidden chat history.

### (b) Specialist contracts

Each specialist exposes a small action space and returns structured observations. The contracts mean the Orchestrator can reason about *what* to delegate without reasoning about *how* the specialist works internally — a clean Liskov-like substitution principle for agents.

### (c) Re-planning trigger

Magentic-One's re-plan is triggered by:

- **Stall detection**: same observation repeated, or progress not measurable.
- **Specialist failure**: a sub-task fails K consecutive times.
- **Plan invalidation**: a fact in the ledger contradicts a plan step.

Re-plan is structured: the Orchestrator emits a new plan + ledger, which becomes the live spec.

### (d) Magentic-UI: the human surface

Magentic-UI shows the live plan, ledger, and per-specialist actions. Humans can:

- **Edit the plan** at any time.
- **Pause / resume** specialists.
- **Override** a specialist's output.
- **Take over** a specialist's surface (e.g., drive the browser for a few steps, then hand back).

This is operationally close to LangGraph's interrupt model ([110-langgraph](110-langgraph.md)), but specifically designed for the Orchestrator + specialist topology.

### (e) AutoGenBench evaluation

The paper introduces **AutoGenBench**, an OSS tool for repeatable agentic-benchmark runs that controls for LLM nondeterminism (multiple seeds), isolates side effects, and produces variance estimates. The benchmarks themselves (GAIA, WebArena, AssistantBench) are not new; the *evaluation discipline* is.

## Empirical results

Headline results from the paper (point estimates with reported intervals — see paper for full tables):

| Benchmark | Magentic-One (GPT-4o) | SOTA when paper released |
|-----------|-----------------------|---------------------------|
| GAIA (Levels 1–3) | competitive with SOTA | matched / approached |
| AssistantBench | competitive with SOTA | matched / approached |
| WebArena | competitive with SOTA | matched / approached |

The paper's framing is "**statistically competitive with state-of-the-art** across three benchmarks with a *single* generalist system" — not new SOTA on any individual bench. This is the right framing for a generalist architecture: it loses to specialists on each axis but covers more ground than any specialist.

## Variants and ablations

- **Orchestrator backbone**: o1-preview > GPT-4o for plan quality; trade-off is latency.
- **Coder vs ComputerTerminal split**: separating "write code" from "execute code" reduces hallucinated executions; a single agent doing both is messier.
- **WebSurfer's web-action API granularity**: broad actions (`navigate`, `extract`) vs narrow (`click`, `type`) — broad easier for the Orchestrator to dispatch, narrow more precise. Magentic-One leans broad, with narrow as fallback.

## Failure modes and limitations

1. **Orchestrator hallucinations propagate.** If the Orchestrator makes a wrong plan, all specialists do wrong work; re-plan is the only recourse, expensive in tokens.
2. **Hand-off costs.** Every Orchestrator-specialist exchange is multiple LLM calls and context translations; latency adds up.
3. **Specialist redundancy.** WebSurfer + ComputerTerminal can do overlapping things; the Orchestrator must choose, which is itself a learned skill.
4. **Adversarial web pages and prompt injection** ([22-guardrails-prompt-injection](22-guardrails-prompt-injection.md)) — WebSurfer is the entry point.
5. **No persistent memory across tasks** by default; Magentic-One is single-task, not stateful across sessions like Letta ([105-letta-stateful-agents](105-letta-stateful-agents.md)).

## When to use, when not

**Use Magentic-One when** tasks span heterogeneous surfaces (web + file + code + shell), you want a generalist baseline, and you can afford the orchestration overhead. Especially compelling for research and demos where breadth-of-capability is the headline.

**Don't reach for it** when tasks are narrow (a coding agent doesn't need WebSurfer), when latency is critical, or when you've already specialized: a single CodeAct agent ([108-openhands-codeact](108-openhands-codeact.md)) on SWE-bench-like tasks beats a Magentic-One stack tuned for breadth.

## Implications for harness engineering

- **The Orchestrator is the Plan Mode** ([03-plan-mode](03-plan-mode.md)) made into a runtime role. Worth seeing the symmetry.
- **Specialist contracts are the Permission/Hooks ([06-permission-modes](06-permission-modes.md), [05-hooks](05-hooks.md)) story** for multi-agent: each specialist's action space is what the harness gates.
- **The ledger is externalized state** ([12-todo-scratchpad-state](12-todo-scratchpad-state.md)) — making it visible to humans is what enables Magentic-UI's HITL story.
- **AutoGenBench's variance discipline** is generalizable: every agent benchmark suffers from LLM nondeterminism, and reporting variance should be standard.

## Connections to other work in this corpus

- **[02-subagent-delegation](02-subagent-delegation.md):** the topology Magentic-One operationalizes.
- **[20-metagpt-role-based-workflows](20-metagpt-role-based-workflows.md), [92-chatdev](92-chatdev.md):** role-based vs task-based multi-agent — different points in the design space.
- **[110-langgraph](110-langgraph.md):** the explicit-graph framework; Magentic-One could be expressed in LangGraph as orchestrator + specialist nodes with conditional edges.
- **[42-langchain-deep-agents](42-langchain-deep-agents.md):** related orchestrator-with-specialists pattern, OSS-distilled.
- **[42-langchain-deep-agents](42-langchain-deep-agents.md), [110-langgraph](110-langgraph.md), [109-smolagents](109-smolagents.md):** alternative framework substrates.

## Key takeaways

1. **Generalist agents are best built as orchestrator + specialists**, not as one heroic monolith.
2. **The ledger pattern** — externalized facts/plan/current-step — is the multi-agent equivalent of a TODO scratchpad.
3. **Re-plan triggers (stall, failure, contradiction)** turn re-planning from a hack into a contract.
4. **Magentic-UI** shows the right HITL surface: not "approve/reject" but **edit the plan, take the keyboard, hand back**.
5. **AutoGenBench** raises the bar on agent evaluation discipline — variance and isolation matter.

## References

- Fourney, A. et al. (2024). *Magentic-One: A Generalist Multi-Agent System for Solving Complex Tasks.* arXiv:2411.04468. https://arxiv.org/abs/2411.04468
- Microsoft Research article: https://www.microsoft.com/en-us/research/articles/magentic-one-a-generalist-multi-agent-system-for-solving-complex-tasks/
- *Magentic-UI: Towards Human-in-the-loop Agentic Systems.* Microsoft Research (July 2025): https://www.microsoft.com/en-us/research/wp-content/uploads/2025/07/magentic-ui-report.pdf
- AutoGen documentation: https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/magentic-one.html
- Code: https://github.com/microsoft/autogen (`packages/autogen-magentic-one`).
- Microsoft Agent Framework Workflows — Magentic Orchestration: https://learn.microsoft.com/en-us/agent-framework/user-guide/workflows/orchestrations/magentic
