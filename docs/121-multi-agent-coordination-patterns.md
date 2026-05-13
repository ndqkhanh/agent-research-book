# 121 — Multi-Agent Coordination Patterns: Beyond Orchestrator-Worker — Blackboard, Contract-Net, Marketplace, Peer-Mesh, Hierarchical

**Sources.** Arsanjani & Bustos, *Agentic Architectural Patterns*, Chapter 5 (Multi-Agent Coordination Patterns) — the canonical catalog including Hierarchical (orchestrator+specialists), Blackboard Knowledge Hub, Contract-Net Marketplace; Lakshmanan & Hapke, *Generative AI Design Patterns*, Pattern 23 (Multiagent Collaboration); plus the foundational multi-agent systems literature (Davis & Smith 1983 on contract net, Engelmore & Morgan 1988 on blackboard, MetaGPT, AutoGen, CrewAI).

**One-line definition.** Multi-agent coordination has six named patterns — *hierarchical orchestrator-specialist*, *blackboard knowledge hub*, *contract-net marketplace*, *peer-mesh / debate*, *role-based pipeline*, *supervisor with self-revision* — each with different control-flow semantics, communication overhead, and failure modes; choosing the right pattern is the difference between a multi-agent system that compounds capability and one that introduces only latency, cost, and diversity collapse ([98-diversity-collapse-mas](98-diversity-collapse-mas.md)).

## Why this chapter matters

By 2026 the question "should this be multi-agent?" is usually a wrong question; the right question is "*which* multi-agent pattern fits this problem?" Most multi-agent systems shipped as "let multiple agents talk and figure it out" produce worse results than a single-agent baseline at higher cost. The literature on diversity collapse ([98-diversity-collapse-mas](98-diversity-collapse-mas.md)) and the empirical evidence from Memento (whose top-1 GAIA result came from *single*-agent + planner-executor + memory) make this point sharply.

For agent builders, naming the patterns is a clarifying move. Each pattern has a specific control-flow shape, specific kinds of problems it solves, and specific failure modes. The team that says "we're using a hierarchical orchestrator-specialist pattern with five specialists and a meta-orchestrator above" is having a different conversation than the team that says "we have a multi-agent system" — and the first conversation is debuggable.

This chapter is the canonical six-pattern catalog with the trade-offs and the matching guidance for when each pattern earns its complexity.

## Problem it solves

Five concrete failures in unnamed-pattern multi-agent systems:

1. **Free-for-all chat.** Multiple LLM agents chat with each other; outputs degrade to mush. Diversity collapse.
2. **Orchestrator overload.** A single orchestrator drives 20 specialists; orchestrator's prompt becomes the bottleneck.
3. **Lost responsibility.** No agent is accountable for the final output; quality is an averaging artifact.
4. **Communication blow-up.** N² messaging between N agents; cost and latency both quadratic.
5. **No termination.** Agents keep talking; no one signals "we're done."

Each is solved by picking a coordination pattern with explicit control-flow semantics.

## Core idea in one paragraph

Multi-agent coordination is the *control-flow problem* at the team level: how do agents decide what to do, who does what, and when to stop? Six patterns name distinct answers. **Hierarchical (orchestrator-specialist)** has one orchestrator that delegates and aggregates; specialists work in isolation. **Blackboard** has a shared workspace where agents publish observations and pull tasks; coordination is implicit through the shared state. **Contract-net** has tasks announced as bids; agents compete for assignment. **Peer-mesh / debate** has agents directly communicate, often in role-asymmetric setups (debater + judge). **Role-based pipeline** has a fixed sequence of agents (analyst → architect → engineer → reviewer); each consumes the previous one's output. **Supervisor with self-revision** has one agent + a meta-supervisor that reviews and asks for revisions. The patterns differ on three axes: who decides (centralised orchestrator vs distributed market vs implicit blackboard), how they communicate (direct messages, shared state, sequential pipes), and how they terminate (orchestrator decision, market clearing, fixed pipeline length, supervisor satisfaction). Match pattern to problem; never let the architecture be implicit.

## Mechanism (step by step)

### 1. Hierarchical (orchestrator-specialist) — the dominant production pattern

```text
[ORCHESTRATOR]
   ├── decompose task
   ├── delegate to specialist 1
   ├── delegate to specialist 2
   ├── delegate to specialist 3
   ├── aggregate results
   └── return final answer
```

- *Control flow*: orchestrator drives.
- *Communication*: orchestrator ↔ specialists; no specialist-to-specialist.
- *Termination*: orchestrator decides.
- *Strengths*: clear accountability, simple debugging, predictable cost.
- *Weaknesses*: orchestrator is the bottleneck; specialists can't share context.

This is the [02-subagent-delegation](02-subagent-delegation.md) pattern, the [42-langchain-deep-agents](42-langchain-deep-agents.md) pattern, and the dominant pattern in production agents. The Memento planner-executor split is a degenerate case: 1 orchestrator + 1 specialist.

When the specialists themselves have specialists, you get nested hierarchy — an *orchestrator-of-orchestrators* pattern. Useful for very large problems; risky for control-flow clarity.

### 2. Blackboard (knowledge hub) — implicit coordination via shared state

```text
[shared blackboard]
   ↑↓ agents read/write
[agent 1] [agent 2] [agent 3] ... [agent N]
```

- *Control flow*: each agent watches the blackboard; acts when it sees a task it can solve.
- *Communication*: implicit through state changes.
- *Termination*: when no agent has anything to do, or a meta-agent declares done.
- *Strengths*: highly parallel, agents can be added/removed, naturally extensible.
- *Weaknesses*: harder to reason about; race conditions on writes; cost of monitoring blackboard.

The classical Hearsay-II speech-understanding system was the canonical blackboard. Modern incarnations: the project's `todo.md` / scratchpad ([12-todo-scratchpad-state](12-todo-scratchpad-state.md)) acts as a single-agent blackboard; multi-agent blackboards live in collaborative-coding harnesses where multiple subagents read/write a shared workspace.

### 3. Contract-net (marketplace) — bidding for tasks

```text
[task announcer] -- broadcasts task --> [agents]
[agents] -- bid with capability + cost -->
[task announcer] -- selects winning bid -->
[winning agent] -- executes -->
```

- *Control flow*: market dynamics.
- *Communication*: announce → bid → select → execute.
- *Termination*: task completion + payment.
- *Strengths*: load-balances naturally; agents only take what they're suited to.
- *Weaknesses*: bidding overhead; agents must accurately self-assess capability; collusion-vulnerable.

Less common in 2026 production; useful when agents are heterogeneous (different capabilities, different costs) and the task distribution is varied. Some research-tool platforms use it; production agents rarely.

### 4. Peer-mesh / debate — direct communication

```text
[agent A] ↔ [agent B] ↔ [agent C]
           [judge agent]
```

- *Control flow*: agents communicate; a judge or aggregator decides.
- *Communication*: O(N²) potential, often with role asymmetry (proposer + critic).
- *Termination*: judge decision or fixed rounds.
- *Strengths*: high diversity (when roles are asymmetric); good for tasks with verifiable correctness (debate → critique → verdict).
- *Weaknesses*: diversity collapse if roles aren't enforced; quadratic communication; expensive.

Specific instances: *debate* (Du et al. 2023) has agents debate a question with a judge; *self-consistency* across multiple agents is a degenerate single-axis form. [98-diversity-collapse-mas](98-diversity-collapse-mas.md) is the literature on why this often fails.

### 5. Role-based pipeline — fixed sequence

```text
[analyst] → [architect] → [engineer] → [reviewer] → [output]
```

- *Control flow*: deterministic sequence.
- *Communication*: each agent consumes only the previous one's output.
- *Termination*: pipeline end.
- *Strengths*: easy to reason about; clear handoffs; testable.
- *Weaknesses*: fixed shape; cannot adapt to varying task structure.

This is essentially a workflow ([114-workflows-vs-agents](114-workflows-vs-agents.md)) with LLM agents at each node. MetaGPT ([20-metagpt-role-based-workflows](20-metagpt-role-based-workflows.md), [91-metagpt-deep](91-metagpt-deep.md)) is the canonical example: SDLC roles (PM → Architect → Engineer → QA) in a fixed pipeline.

### 6. Supervisor with self-revision

```text
[agent] -- output -->
[supervisor] -- approve / revise -->
[agent] -- revised output -->
... until [supervisor] approves
```

- *Control flow*: agent + supervisor loop.
- *Communication*: agent ↔ supervisor only.
- *Termination*: supervisor approval or max revisions.
- *Strengths*: self-correcting; combines generation with verification.
- *Weaknesses*: supervisor bias (if supervisor is misaligned, the system iterates toward bad outputs).

This is the [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md) pattern at the multi-agent level. Reflexion ([14-reflexion](14-reflexion.md), [90-reflexion-deep](90-reflexion-deep.md)) is an in-agent version; multi-agent supervisor is the extension.

### 7. Choosing the pattern

| Problem shape | Pattern |
|---|---|
| Decompose, delegate, aggregate | Hierarchical |
| Many parallel tasks of varying types | Blackboard |
| Heterogeneous agents with cost/capability variation | Contract-net |
| Verifiable answer, want diversity | Peer-mesh / debate |
| Stable SDLC-like task flow | Role-based pipeline |
| Single agent + need for verification | Supervisor with self-revision |

Most production systems use hierarchical or role-based pipeline. Blackboard, contract-net, and peer-mesh are appropriate for specific niches.

### 8. Anti-patterns

- **"Let the agents talk."** Free-form chat with no role asymmetry; diversity collapses, cost rises, no clear termination.
- **Pattern-mixing.** Hierarchical orchestrator that also acts as a debate judge as a contract-net auctioneer; debugging becomes impossible.
- **No termination criterion.** Multi-agent systems without an explicit "done" signal run forever.
- **Implicit role boundaries.** "Agent 1 does coding and review and architecture" — when roles overlap, accountability dissolves.

## Empirical anchors

- **MetaGPT (role-based pipeline)** outperforms ad-hoc multi-agent on software engineering benchmarks ([91-metagpt-deep](91-metagpt-deep.md)).
- **Hierarchical** is the dominant production pattern; Memento, Claude Code's subagents, LangChain Deep Agents all use it.
- **Diversity collapse** ([98-diversity-collapse-mas](98-diversity-collapse-mas.md)) is the most-reported failure mode in undisciplined multi-agent systems.
- **Single-agent + memory** ([109-memento-results-and-harness](109-memento-results-and-harness.md)) outperforms many multi-agent setups; multi-agent isn't always the right answer.
- **Cost-multipliers** of 2–10× over single-agent are typical for well-tuned multi-agent; > 10× usually means the pattern is wrong.

## Variants and counter-arguments addressed

- **"Multi-agent is the future."** Conditional on the right pattern; otherwise it's expensive single-agent.
- **"Hierarchical is too rigid."** It's the most reliable. If your problem genuinely needs more dynamism, blackboard or peer-mesh; document the choice.
- **"Pipelines aren't really multi-agent."** Definitionally, multi-agent means multiple agents; sequence is one valid coordination shape.
- **"Debate solves diversity collapse."** Sometimes; depends on role asymmetry and judge quality.
- **"More agents = more capability."** No. Each agent adds cost and coordination overhead; capability gains diminish quickly past 3–5 agents in most patterns.

## Failure modes and limitations

1. **Diversity collapse.** Agents converge to the same answer; multi-agent compute is wasted.
2. **Orchestrator bottleneck.** Hierarchical pattern's orchestrator becomes the constraint.
3. **Communication explosion.** Peer-mesh patterns blow up at N > 5 agents.
4. **Termination ambiguity.** No one signals "done"; system runs to budget.
5. **Role drift.** Specialists in role-based pipelines start doing each other's jobs.
6. **Coordination overhead exceeds gain.** Some problems are simply not multi-agent problems; forcing them is pure overhead.
7. **Failure cascade.** One specialist fails; orchestrator can't recover; entire system fails.
8. **Eval complexity.** Multi-agent trajectories are harder to evaluate than single-agent; building eval infrastructure first ([115-evaluating-llm-systems](115-evaluating-llm-systems.md)) is non-optional.

## When to use, when not

**Use hierarchical** when the task decomposes cleanly and the orchestrator can manage delegation.

**Use blackboard** when many agents must work in parallel on related tasks and the shared state is the natural coordination medium.

**Use contract-net** when agents are heterogeneous in capability and cost.

**Use peer-mesh / debate** when the answer is verifiable and diversity adds value.

**Use role-based pipeline** when the task has a stable SDLC-like flow.

**Use supervisor with self-revision** when verification is the bottleneck.

**Default to single-agent** when none of the above clearly applies; multi-agent is overhead, not capability, by default.

## Implications for harness engineering

- **Name the pattern explicitly.** Every multi-agent system should have a one-line declared pattern in its design doc.
- **Match infrastructure to pattern.** Hierarchical fits [02-subagent-delegation](02-subagent-delegation.md) primitives; blackboard needs shared-workspace abstractions; pipeline fits workflow frameworks.
- **Bound termination.** Step budgets, time budgets, cost budgets per agent and at the coordinator level. See [01-agent-loop-architecture](01-agent-loop-architecture.md).
- **Eval the pattern's specific failure modes.** Diversity collapse for peer-mesh, orchestrator overload for hierarchical, communication blow-up for direct mesh. Per-pattern checks.
- **Start single-agent.** Promote to multi-agent only when single-agent has demonstrably hit a ceiling.
- **Avoid pattern mixing.** Pick one; commit; debug. Mixing patterns combines the failure modes of all.
- **Multi-LoRA + role specialisation** lets you serve role-based pipelines from one base model — a 2026 production pattern. See [116-adapter-tuning-lora-peft-for-agents](116-adapter-tuning-lora-peft-for-agents.md), [117-small-language-models](117-small-language-models.md).
- **Document inter-agent contracts.** What does each agent input/output? Pydantic models at every boundary; constrained decoding ([112-constrained-decoding](112-constrained-decoding.md)) for reliability.

The one-sentence takeaway: **multi-agent coordination is six named patterns — pick one, name it, bound it, eval it; never ship "the agents talk to each other and figure it out."**

## See also

- [02-subagent-delegation](02-subagent-delegation.md), [42-langchain-deep-agents](42-langchain-deep-agents.md) — hierarchical patterns in detail.
- [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md) — the supervisor/verifier pattern.
- [14-reflexion](14-reflexion.md), [90-reflexion-deep](90-reflexion-deep.md) — single-agent self-revision; the multi-agent extension is supervisor-with-self-revision.
- [20-metagpt-role-based-workflows](20-metagpt-role-based-workflows.md), [91-metagpt-deep](91-metagpt-deep.md), [92-chatdev](92-chatdev.md) — role-based pipeline canonical examples.
- [12-todo-scratchpad-state](12-todo-scratchpad-state.md) — single-agent blackboard; multi-agent extension.
- [98-diversity-collapse-mas](98-diversity-collapse-mas.md) — the dominant failure mode of poorly-designed peer-mesh systems.
- [114-workflows-vs-agents](114-workflows-vs-agents.md) — when "multi-agent" should actually be a workflow.
- [109-memento-results-and-harness](109-memento-results-and-harness.md) — Memento's single-agent + memory beating multi-agent baselines.
