# 142 — Trajectory-Simulation / Prediction Agents: What-If Reasoning, Counterfactuals, Simulation as a Tool

**Sources.** Rothman, *Building Business-Ready Generative AI Systems*, Chapter 8 (GenAI for Trajectory Simulation and Prediction); the broader simulation-augmented AI literature (Mu Zero, world models, RL with simulation); plus the emerging "agent + simulator" pattern in operations research, supply chain, scientific discovery.

**One-line definition.** Trajectory-simulation agents combine an *LLM agent* (decision-maker, hypothesis generator, narrator) with a *simulator* (deterministic or stochastic model of a domain — supply chain, traffic flow, financial market, scientific experiment, business process) where the agent proposes interventions, the simulator evaluates them, and the agent synthesises results — turning "what would happen if?" from speculation into measurable forecast, and unlocking applications (operations planning, risk assessment, scientific discovery, business simulation) that pure-LLM agents cannot reach.

## Why this matters

LLMs are good at hypothesis generation and narrative reasoning; they are bad at quantitative prediction in complex systems. Simulators are the inverse: deterministic, quantitative, but require an operator to know what to ask. Combining the two — LLM as the *hypothesis-and-narrative* layer, simulator as the *quantitative-evaluation* layer — produces capabilities neither has alone.

For agent builders, simulation-augmented agents are an emerging frontier. Use cases: supply chain disruption forecasting, what-if pricing analysis, scientific discovery (the agent proposes experiments, the simulator runs them), business process simulation, risk assessment under multiple scenarios, autonomous-vehicle trajectory planning, climate scenario analysis.

This chapter is the architecture for hybrid agent + simulator systems — how the LLM reasons over simulator outputs, how to build simulators agents can call, and the evaluation discipline that ensures simulator-based answers are trustworthy.

## Problem it solves

Five concrete capabilities simulation-augmented agents enable:

1. **What-if analysis at scale.** "What if we delay this product launch by 2 weeks?" The agent runs simulator scenarios and synthesises results.
2. **Risk forecasting.** Run Monte Carlo simulation across uncertain inputs; agent narrates the distribution.
3. **Counterfactual reasoning.** "Would we have won if we'd lowered the price 10% in February?" Replay the scenario in simulation.
4. **Hypothesis testing in scientific discovery.** Agent proposes experiments; simulator (or actual lab) executes; results inform next hypothesis.
5. **Operations optimisation.** Find the best policy via simulation-based search; agent explains the trade-offs.

## Core idea in one paragraph

A simulator is a *callable model of a domain* that takes inputs (state, intervention, parameters) and returns outputs (results, metrics, traces). It can be deterministic (engineering simulator, supply-chain network) or stochastic (Monte Carlo, agent-based model). The LLM agent treats the simulator as a tool: poses questions, generates input scenarios, calls the simulator, interprets outputs, and either synthesises an answer or proposes follow-up scenarios. The integration pattern is iterative — the agent runs many scenarios, looks for patterns, drills into edge cases, and produces a narrative that is *grounded in simulator outputs* rather than imagined. Critically, the simulator is *cited* in the agent's response: "Under scenario X, we'd expect outcome Y with 80% probability (simulation result), versus Z under scenario X' (simulation result)." This is the [135-trustworthy-generation](135-trustworthy-generation.md) pattern extended to quantitative outputs. Cost: building good simulators is hard. Benefit: the agent's quantitative outputs become trustworthy.

## Mechanism (step by step)

### 1. The architecture

```text
[user question: "what would happen if X?"]
   ↓
[agent: hypothesis + scenario design]
   ↓
[simulator: run scenario(s)]
   ↓
[agent: interpret results, identify patterns, drill into edge cases]
   ↓
[multiple iterations]
   ↓
[final narrative answer with simulator citations]
```

Iterative: the agent doesn't just call once; it explores.

### 2. Simulator types

**Deterministic engineering simulators**: physics, supply chain, network flow, financial models. Inputs → outputs reproducibly.

**Stochastic simulators**: Monte Carlo, queueing models, epidemic models. Inputs → distribution of outputs.

**Agent-based simulators**: many simulated actors with rules; emergent behavior. Used in market simulation, traffic flow, social dynamics.

**Black-box surrogates**: ML-trained surrogates that approximate expensive simulators (e.g. neural surrogates for physics simulations).

**Real-world experiments**: in scientific contexts, the "simulator" can be an actual lab; the loop is the same but slower and more expensive.

### 3. Simulator as a tool

```python
@tool
def run_supply_chain_simulation(
    network: dict,
    disruption: dict,
    duration_days: int
) -> dict:
    """Simulate the supply chain network under a given disruption."""
    sim = SupplyChainSim(network)
    sim.apply_disruption(disruption)
    results = sim.run(days=duration_days)
    return {
        "service_level": results.service_level,
        "stockouts": results.stockouts,
        "cost_impact": results.cost,
        "time_series": results.time_series,
    }
```

Agent calls this like any tool. Output is structured. Multiple calls explore the scenario space.

### 4. Hypothesis-driven scenario design

The agent generates scenarios:

```text
User: "What if we lose our primary supplier in March?"

Agent:
  - Scenario 1: complete supplier outage, no replacement
  - Scenario 2: outage with secondary supplier ramp at 50% capacity
  - Scenario 3: outage with secondary supplier at 100% capacity, 10-day delay
  - For each: vary demand assumptions (low / median / high)
  - Total: 9 scenarios
```

LLM is good at this scenario generation; uses domain knowledge to design realistic scenarios.

### 5. Result interpretation + narrative synthesis

After running:

```text
Agent narration:
"Under complete outage (Scenario 1), we'd expect a 35% service-level drop
and $12M cost impact over 60 days. With secondary supplier at 50%
ramp (Scenario 2), service stays above 80% but at $7M cost. The
secondary supplier strategy is recommended; further analysis suggests..."

Citations:
- Scenario 1 simulator run: ID-X1
- Scenario 2 simulator run: ID-X2
...
```

Each claim is grounded in a specific simulator run — the [135-trustworthy-generation](135-trustworthy-generation.md) pattern applied to simulation.

### 6. Iterative drill-down

If a scenario shows surprising results, agent investigates:

```text
Surprising: under Scenario 3, costs are higher than Scenario 1
  → why?
  → call simulator with finer time granularity
  → cross-check with sensitivity analysis
  → narrate the explanation
```

This is [25-agentic-rag](25-agentic-rag.md) for simulation: iterative refinement until the picture is clear.

### 7. Stochastic simulation

When simulator is stochastic (Monte Carlo):

```text
Run scenario N times (e.g. N=1000)
   ↓
[agent receives distribution of outcomes]
   ↓
Narrative: "Median outcome is X with 80% CI [Y, Z]. The downside tail
shows..."
```

Agent communicates uncertainty correctly — not as "X will happen" but as "X is most likely, but Y has 10% probability."

### 8. Surrogate modelling

For expensive simulators, train ML surrogate:
- Run simulator for many input scenarios.
- Fit a neural surrogate that approximates input → output.
- Use surrogate for fast iteration; spot-check with real simulator.

Agent may not know which it's calling; the harness handles routing.

### 9. Real-experiment loop

For scientific discovery:

```text
[agent: proposes experiment]
   ↓
[lab: runs experiment (days/weeks)]
   ↓
[agent: incorporates real result]
   ↓
[propose next experiment]
```

Same loop, longer cycle time. Patterns: AlphaFold pipeline, AlphaEvolve ([85-alphaevolve](85-alphaevolve.md)).

### 10. Validation discipline

Simulator outputs need their own eval:
- **Validation set**: known scenarios with known outcomes (historical disruptions for supply chain).
- **Sensitivity analysis**: vary inputs; check outputs change reasonably.
- **Stress tests**: edge-case inputs.
- **Surrogate accuracy**: where surrogates are used, periodic check vs full simulator.

The simulator's quality is the agent's quality ceiling; validate aggressively.

## Empirical anchors

- **Supply-chain agents** with simulation reduce planning time from days to hours.
- **Risk assessment** with Monte Carlo + agent narration produces reports comparable to consultant work.
- **Scientific discovery** with hypothesis-experiment loops accelerates research; AlphaFold and similar are early examples.
- **Operations research** problems: simulation + LLM scenario design beats classical optimisation on ill-structured problems.
- **Adoption** is emerging in 2026 in supply chain, financial risk, energy planning, scientific R&D.

## Variants and counter-arguments addressed

- **"LLMs can predict directly."** They can hypothesise; they can't quantitatively predict complex systems reliably.
- **"Just use the simulator."** Simulators need operators who know what to ask. The LLM is the operator.
- **"Building simulators is hard."** Yes; that's why the existing simulator base in industries that have them (engineering, ops) is the moat.
- **"Surrogates are inaccurate."** With proper validation, they're sufficient for many use cases; spot-check.
- **"What-if analysis is fluff."** Not when it informs decisions worth millions.

## Failure modes and limitations

1. **Simulator-reality gap.** Simulator doesn't match reality; agent answers are wrong.
2. **Hallucination of simulator results.** Agent makes up numbers when simulator output is unclear; verify ([135-trustworthy-generation](135-trustworthy-generation.md)).
3. **Scenario gaps.** Agent doesn't generate the scenario that matters; user expertise still required.
4. **Compute cost.** Running many simulations is expensive.
5. **Stochastic interpretation errors.** Agent treats Monte Carlo medians as certainties.
6. **Cycle time.** Real experiments take days/weeks; agent's iteration speed bound by reality.
7. **Domain expertise gap.** Without domain context, agent's scenarios are naive.
8. **Eval gap.** No standard benchmark for simulation agents; bespoke eval per domain.

## When to use, when not

**Use simulation-augmented agents when** you have a working simulator (or can build one), the questions are quantitative what-if, and decisions benefit from scenario-based analysis.

**Skip when** the domain doesn't have a credible simulator and building one is impractical.

**Add real-experiment loops** for scientific contexts; cycle time is slower but the loop is the same.

**Validate aggressively**: the simulator's quality is the agent's quality.

## Implications for harness engineering

- **Simulator as a tool.** [07-model-context-protocol](07-model-context-protocol.md). Standard tool interface.
- **Iterative scenario exploration.** Multi-step agent that generates, runs, interprets, refines.
- **Citation of simulator runs.** Each claim links to a specific simulator output.
- **Stochastic uncertainty communication.** Agents must convey distributions, not just medians.
- **Validation discipline.** Simulator + surrogate + agent each have eval.
- **Compute budgeting.** Simulations are expensive; budget per task ([123-robustness-fault-tolerance](123-robustness-fault-tolerance.md)).
- **Domain experts in the loop.** Especially for scenario design and validation. See [23-human-in-the-loop](23-human-in-the-loop.md).
- **Real-experiment integration**: longer cycle times, queue-based.

The one-sentence takeaway: **simulation-augmented agents combine LLM hypothesis generation and narrative reasoning with simulator quantitative evaluation — turning what-if analysis from speculation into grounded forecasts, with citations to specific simulator runs as the trust mechanism.**

## See also

- [85-alphaevolve](85-alphaevolve.md) — evolutionary coding agent for scientific / algorithmic discovery.
- [102-clawgym-scalable-claw-agents](102-clawgym-scalable-claw-agents.md) — agent + sandbox training.
- [69-agent-world-self-evolving-training-arena](69-agent-world-self-evolving-training-arena.md) — agents in synthetic environments.
- [37-neuro-symbolic-ai](37-neuro-symbolic-ai.md) — symbolic + neural reasoning.
- [135-trustworthy-generation](135-trustworthy-generation.md) — citation pattern.
- [123-robustness-fault-tolerance](123-robustness-fault-tolerance.md) — cost budgets.
- [140-traditional-ml-genai-hybrid](140-traditional-ml-genai-hybrid.md) — broader hybrid pattern.
- [149-sector-use-case-catalog](149-sector-use-case-catalog.md) — sector applications.
