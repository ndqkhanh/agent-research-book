# 114 — Workflows vs Agents: When Each Wins, and the Decision Framework Production Teams Use

**Sources.** Ozdemir, *Building Agentic AI*, Chapter 4 (When Should You Use Workflows Versus Agents?); Albada, *Building Applications with AI Agents*, Chapter 5 (Orchestration); Anthropic's *Building Effective Agents* essay (Schluntz & Zhang, 2024); LangChain's *State of AI Agents* report; the canonical workflow vs agent dichotomy from operations research.

**One-line definition.** A *workflow* is a pre-defined directed graph of LLM calls and tools that the harness executes deterministically; an *agent* is an LLM that *chooses* the next call dynamically based on observations — and the decision between them is not "agents are newer so use agents" but a four-axis trade-off (predictability, cost, error-recovery, breadth) where workflows win on three axes and agents win on the fourth.

## Why this matters

The default 2024 framing was "agents replace workflows." The default 2026 framing is more sober: workflows and agents are complementary, and the most successful production systems are *workflow-first with agentic escapes*. Anthropic's own essay codified this: most production "agents" should be workflows; agentic dynamism should be reserved for the 5–10% of tasks where the next step truly cannot be precomputed.

For agent builders, choosing wrongly is expensive in two ways. **Choosing agents when workflows would do** burns 5–10× the tokens, adds nondeterminism that breaks integration tests, and makes debugging an interpretive exercise instead of a deterministic one. **Choosing workflows when agents are needed** ships a brittle pipeline that breaks when the input shape varies and requires constant re-engineering as cases proliferate. The discipline is knowing which side of the line you are on.

This chapter is the decision framework: four axes, six concrete patterns, the workflow-first heuristic, and the explicit list of cases where agents earn their cost.

## Problem it solves

Five common mis-categorisations:

1. **"It uses an LLM, so it's an agent."** No. A function that calls GPT-4 to summarise a document is a workflow node, not an agent.
2. **"It has tool calls, so it's an agent."** No. Tool calls inside a fixed sequence (search, then summarise, then email) are still a workflow.
3. **"It has multiple LLM calls, so it's an agent."** No. Sequential LLM calls where each prompt is determined by the previous output's *value* but not its *shape* are still a workflow.
4. **"It has a loop, so it's an agent."** Often, but not always. A loop with deterministic exit conditions and fixed body is a workflow with a `while`.
5. **"It's prompt-driven, so it's an agent."** Prompts are the unit of LLM communication regardless of architecture.

The defining property of agents is *dynamic next-step selection by the LLM*, not LLM use, tool use, or iteration.

## Core idea in one paragraph

A workflow is a graph: nodes are LLM calls or tools, edges are deterministic data flow. The harness executes the graph; the LLM contributes content but not control flow. An agent is an LLM in a loop that *chooses its own control flow* — at each step, the LLM decides what to do next based on what it has observed, and the harness executes whatever the LLM picks. Workflows win on predictability (the next step is known), cost (no exploration tokens), error recovery (deterministic retries), and observability (the graph is the spec). Agents win on *breadth*: tasks whose shape varies enough that you cannot enumerate the graph in advance. The decision rule is therefore: **enumerate the task. If you can list every possible execution path in a flowchart, build a workflow. If you cannot, build an agent — but constrain it to the smallest dynamic region that the workflow cannot cover.**

## Mechanism (step by step)

### 1. The defining distinction — control flow

Workflow: control flow is in *code*. The harness decides what to run next.
```python
docs = retrieve(query)
summaries = [summarize(d) for d in docs]
result = synthesize(summaries)
```

Agent: control flow is in the *LLM*. The harness executes whatever the LLM picks.
```python
while not done:
    action = llm.next_action(state)
    obs = execute(action)
    state.update(action, obs)
    done = state.terminated
```

This is the only structurally meaningful distinction. Everything else (tools, prompts, memory) appears in both.

### 2. Four-axis trade-off

| Axis | Workflow | Agent |
|---|---|---|
| **Predictability** | High — execution path is fixed | Low — path varies with input |
| **Cost** | Low — no exploration tokens | 3–10× higher — agent tries paths |
| **Error recovery** | Easy — retry the failing node | Hard — agent may take a wrong path on retry |
| **Breadth** | Narrow — covers enumerable cases | Broad — covers what you didn't enumerate |

Workflows win three axes; agents win one. This is why the workflow-first heuristic is correct.

### 3. The workflow-first heuristic

Anthropic's essay puts it bluntly: **start with a workflow; add agentic dynamism only where the workflow cannot cover the case**.

The decision flow:
1. **Can I enumerate the task as a flowchart?** If yes → workflow.
2. **Are there a few branches?** If yes → workflow with branches; still no agent needed.
3. **Are there many branches that depend on content I cannot anticipate?** If yes → agent for the dynamic region only.
4. **Is the dynamic region the *whole* task?** If yes → full agent; this is rare.

Most production "agents" in 2026 are workflow + small agentic regions, not full agents.

### 4. Six concrete patterns along the spectrum

| Pattern | Type | Example |
|---|---|---|
| **Linear workflow** | Workflow | Retrieve → summarise → email |
| **Branching workflow** | Workflow | Classify → route to one of N pipelines |
| **Looping workflow** | Workflow | While doc has more pages, summarise next page |
| **Workflow with verifier** | Workflow + small agent | Generate answer; verifier agent decides retry or accept |
| **Bounded agent** | Agent | Research agent with max 20 iterations and a fixed tool set |
| **Open agent** | Agent | Fully autonomous coding agent with no step budget |

The middle three (looping, verifier, bounded) cover most production needs; pure linear is too narrow, pure open is rarely justified.

### 5. When agents earn their cost

Five concrete situations:

- **Open-ended research.** Cannot enumerate which documents to fetch in advance.
- **Coding agents.** Cannot enumerate which files to read or edits to make.
- **Computer-use agents.** Cannot enumerate which buttons to click on a website.
- **Multi-step debugging.** Cannot enumerate which hypotheses to test.
- **Negotiation / dialogue agents.** Cannot enumerate the user's responses.

Common to all: the *shape* of the task is unknown until the agent encounters the input.

### 6. When workflows earn their predictability

Five concrete situations:

- **Document processing pipelines.** Read PDF → extract → classify → store.
- **Customer support deflection.** Categorise question → route to FAQ or escalate.
- **Data extraction.** Extract structured fields from invoices.
- **Content generation pipelines.** Outline → draft → edit → publish.
- **Most ETL with LLM steps.** LLM is one tool in a fixed pipeline.

Common to all: the *shape* is fixed; only the *content* varies.

### 7. The hybrid pattern — workflow-with-agentic-region

The dominant production pattern in 2026:

```text
[deterministic pre-processing]
     ↓
[deterministic retrieval]
     ↓
[AGENT — bounded, e.g. max 10 steps, fixed tool set]
     ↓
[deterministic post-processing]
     ↓
[deterministic output]
```

The agent is a *bounded region*. Inputs and outputs are typed. Cost is bounded by step budget. Inside the region, the agent has freedom; outside, the harness is in control.

This pattern is what most "Deep Research" agents (DeepSeek's, Memento, Tongyi's DeepResearch) actually implement: the dynamic region is the search-and-synthesis loop; everything before and after is pipeline.

## Empirical anchors

- **Workflows are 3–10× cheaper** for tasks both can do.
- **Agents are higher-variance**. The same input can produce different outputs across runs (without T=0 + cache control).
- **80% of "agents" in production are workflows**, by Anthropic's own counting in 2024.
- **Bounded agents win on long-tail** tasks where workflows would need 50+ branches.
- **DSPy compilation** ([93-dspy](93-dspy.md)) is workflows-as-code; useful when each LLM-step is narrow.
- **Agentic Deep Research** systems ([109-memento-results-and-harness](109-memento-results-and-harness.md), [65-deer-flow-bytedance](65-deer-flow-bytedance.md)) bound the agent region to the search loop; outside is pipeline.

## Variants and counter-arguments addressed

- **"Workflows can't handle uncertainty."** They can — branching plus verifier-loops handle most uncertainty. Agents are needed only when the *branches themselves* cannot be enumerated.
- **"Frameworks like LangGraph blur the line."** They allow both. The blur is in the framework, not in the architecture; the question "does the LLM choose the next step?" still applies node by node.
- **"Multi-agent systems are not agents?"** Multi-agent is *several* agents (or workflows) coordinating. Each component is itself either a workflow or an agent. The orchestration layer is usually a workflow.
- **"Aren't fully-agentic systems the future?"** Maybe. In 2026, full agents are still expensive and brittle for most production tasks. The trajectory matters; today's right answer is workflow-first.
- **"Doesn't 'workflow-first' miss the agentic upside?"** Only if you treat the choice as binary. The hybrid pattern captures the upside without the brittleness.

## Failure modes and limitations

1. **Premature agentification.** Building an agent because "it's the modern way" rather than because the task needs it. Cost overrun, debugging hell.
2. **Workflow ossification.** Building a workflow with 50 hand-coded branches when a small bounded agent would do. Maintenance burden grows linearly with cases.
3. **Unbounded agents.** Shipping an agent without step / cost budgets. One runaway agent eats a day of API quota.
4. **Mis-typed boundaries.** The hybrid pattern requires typed I/O at the agent boundary. Without typing, the deterministic post-processing breaks on unexpected agent output.
5. **Eval mismatch.** Workflows are easy to test (deterministic); agents need trajectory evaluation ([21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md)). Choosing agents and not building the eval is shipping blind.
6. **Cost surprise.** Agents cost more than expected because exploration tokens are invisible until the bill arrives. Always include the agent region's average iterations × cost-per-iteration in the budget.

## When to use, when not

**Use a workflow when** the task shape is stable, the cases are enumerable, integration tests must be deterministic, and cost is constrained. This covers most production ETL, document processing, classification, extraction.

**Use a bounded agent when** the dynamic region cannot be enumerated *but* you can wrap it with typed I/O and cost limits. This covers most research, coding, computer-use tasks.

**Use a fully-agentic system when** even the I/O shape is unknown. This is rare and usually a sign that the problem decomposition is incomplete.

## Implications for harness engineering

- **Adopt workflow-first as the default mindset.** Most agent-shaped problems in production are actually workflow-shaped. Build the workflow; promote to agent only where evidence forces it.
- **Bound every agentic region.** Step budget, cost budget, time budget, tool subset. See [01-agent-loop-architecture](01-agent-loop-architecture.md), [44-four-pillars-harness-engineering](44-four-pillars-harness-engineering.md).
- **Type the boundaries.** Pydantic at every workflow ↔ agent transition. Pair with [112-constrained-decoding](112-constrained-decoding.md) for guaranteed-valid I/O.
- **Build trajectory evaluation early.** If you're building an agentic region, you need [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md) before deployment, not after.
- **Use DSPy / LangGraph / similar for the workflow** layer; use Memento / Deep Research / Open Claw / your own loop for the agent region. See [93-dspy](93-dspy.md), [42-langchain-deep-agents](42-langchain-deep-agents.md), [108-memento-codebase-mcp](108-memento-codebase-mcp.md).
- **Document each region's contract.** "This agent has goal G, tool set T, budget B, output type O." This is the integration boundary; treat it as an API contract.
- **Re-evaluate annually.** As models improve, the workflow / agent line shifts. A region that needed an agent in 2024 might be a workflow in 2026.

The one-sentence takeaway: **workflows beat agents on three of four axes — use workflows by default, agents only for the dynamic region you cannot enumerate, and bound that region tightly.**

## See also

- [01-agent-loop-architecture](01-agent-loop-architecture.md) — the canonical agent loop.
- [02-subagent-delegation](02-subagent-delegation.md), [42-langchain-deep-agents](42-langchain-deep-agents.md) — workflow + bounded-agent patterns.
- [16-plan-and-solve](16-plan-and-solve.md) — plan-and-solve is workflow-after-planning.
- [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md), [115-evaluating-llm-systems](115-evaluating-llm-systems.md) — eval that distinguishes workflow tests from agent tests.
- [93-dspy](93-dspy.md) — DSPy is workflow-as-code; the right home for narrow LLM steps.
- [108-memento-codebase-mcp](108-memento-codebase-mcp.md) — bounded research agent inside a workflow shell.
