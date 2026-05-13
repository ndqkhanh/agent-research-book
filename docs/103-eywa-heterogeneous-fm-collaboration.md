# 103 — Eywa: Heterogeneous Scientific Foundation Model Collaboration

**Paper.** Zihao Li, Jiaru Zou, Feihao Fang, Xuying Ning, Mengting Ai, Tianxin Wei, Sirui Chen, Xiyuan Yang, Jingrui He — *Heterogeneous Scientific Foundation Model Collaboration* — arXiv:2604.27351v1 — University of Illinois Urbana-Champaign — April 30, 2026. Code: https://github.com/Violet24K/Eywa.

**One-line definition.** Eywa is the first agentic framework where an LLM coordinates **non-LLM scientific foundation models** (Chronos for time series, TabPFN for tabular, etc.) as first-class peers — not via lossy serialization to text but through a structured **Tsaheylu** bond (compiler ϕ_k + adapter ψ_k, instantiated over MCP) — deployed as three drop-in replacements for current agent stacks: EywaAgent (one LLM + FMs), EywaMAS (heterogeneous topology with mixed agent types), and EywaOrchestra (a conductor LLM picks agents, FMs, and topology per task).

## Why this paper matters

The current agentic toolbox makes one tacit assumption that is wrong for science: **natural language is a universal interface**. When an LLM agent encounters a 1,000-point time series or a 50,000-row tabular dataset, today's harnesses serialize it into tokens (often dozens of times the original byte count), then ask the LLM to "reason" over the serialized form. This burns tokens, loses precision, and asks a model trained on text corpora to do work that domain-specific foundation models — Chronos, TabPFN, GraphCast, AlphaFold, ChGNet — were *built* to do natively. Domain models exist; they're capable; they're locked out of agentic systems because they don't speak language.

Eywa is the first framework that takes this seriously. The construction is deliberately minimal: a bidirectional FM↔LLM interface (the *Tsaheylu* bond, after the Na'vi neural connection in Avatar) consisting of a query compiler ϕ_k that translates LLM intent into a structured FM invocation, and a response adapter ψ_k that converts the FM's domain-specific output into language-consumable form. Instantiate ϕ and ψ as a Model Context Protocol (MCP) server and the LLM gets to *delegate native computation* to the right specialist instead of pretending to do it in tokens. Theorem 3 proves the inclusion is strict: any task family where domain FMs outperform serialized-text LLM reasoning admits an EywaAgent that outperforms any LLM-only agent. Empirically, EywaAgent simultaneously **lifts utility +6.6%, cuts tokens −30%, and cuts latency −10%** versus a single-LLM baseline. The Pareto improvement is unusual: most agentic upgrades trade quality for cost or vice versa.

Take this paper seriously and three things change. (1) Multi-agent systems should care about *cross-modality* heterogeneity, not just *cross-LLM* heterogeneity — the empirical message of the EywaMAS vs MoA/X-MAS comparison is that mixing GPT and Claude buys you nothing on scientific tasks; mixing GPT and TabPFN buys you everything. (2) Topology should be a **task-conditional decision**, not a fixed framework choice — EywaOrchestra makes this explicit and gets 99% of EywaMAS utility at ~70% of its cost. (3) MCP graduates from a "tool-call protocol" to a **native interface for foundation-model federation**: any specialist model with input/output schemas can sit on the other side of an MCP boundary and become an agent's hands.

## Problem it solves

- **Language-as-universal-interface bottleneck.** Serializing structured data into tokens loses fidelity (precision, ordering, magnitudes), explodes token cost (Figure 1, left: single-LLM baseline 4,469 tokens vs EywaAgent 3,137), and asks the LLM to do domain-specific computation it was never trained to do.
- **Foundation-model isolation.** Specialized FMs lack language interfaces and integrate with agent frameworks only through hand-crafted procedures and rigid tool wrappers. Refine, Debate, MoA, and X-MAS all assume natural-language-only communication.
- **Fixed-topology limitation.** Static topologies (sequential, hierarchical, debate) suit some tasks and waste compute on others. No existing system selects topology and agent composition based on task characteristics.

## Core idea in one paragraph

Build a two-sided contract between LLM agents and domain foundation models: a *compiler* that turns the agent's state into a structured FM invocation (ϕ_k : S → U_k), and an *adapter* that converts the FM's output into a representation the LLM can absorb (ψ_k : O_k → Z_k). Implement both as MCP server endpoints so any FM with declared schemas joins the federation without retraining. Add a per-step *control policy* C : S → {invoke, skip} that lets the LLM decide whether to delegate or continue language reasoning. Stack three deployments on this primitive: EywaAgent fuses one LLM with multiple FMs; EywaMAS replaces a subset of agents in any conventional MAS topology with EywaAgents (mixed types coexist); EywaOrchestra adds an LLM-conductor that *selects* agents, FMs, and topology per task. Theorems 3 (single-agent strict improvement) and the MAS analog (Appendix A.4) prove that the language-only baseline is a *strict subset* of the Eywa configuration space — adding FMs never hurts the achievable optimum. EywaBench (3 domains × 3 modalities × 9 datasets) verifies the theory and shows the gains are larger than within-LLM heterogeneity (multiple LLMs alone) by a wide margin.

## Mechanism (step by step)

### Formalism

LLM agent: A_LLM : S → Δ(M).
Domain FM: F_k : X_k × U_k → O_k (input space X_k, structured control U_k, output space O_k).
Multi-agent system: M = (A, G), agents A = {A₁, …, Aₙ} and topology G; at step t, s_i^t = Update_i(s_i^{t-1}, m_{−i}^t), m_i^t ~ A_i(s_i^t).

Task τ = (q, x, y*, ℓ) with input space *factorized* as X = X_lng × X₁ × … × X_m. Objective: min_G E_τ [ℓ(ŷ_G(τ), y*)].

**Assumption 1 (Domain Advantage).** For tasks with informative domain component x_k:
  E[ℓ_k(F_k(x_k), y*)] < E[ℓ_k(A_LLM(serialize(x_k)), y*)]

Empirically validated (Table 1): time-series and tabular sub-tasks see consistent FM > LLM-on-serialized advantage.

### Tsaheylu — the FM↔LLM bond

Given domain k, define an interface pair (ϕ_k, ψ_k):
- **Query compiler** ϕ_k : S → U_k. Given the LLM's state, produce a structured FM invocation (dataset id, prediction horizon, conditioning variables, etc.).
- **Response adapter** ψ_k : O_k → Z_k. Convert the FM's domain-specific output into a language-consumable representation Z_k (e.g. "predicted next-month value 45.3 ± 2.1").

Pipeline:
  τ → [LLM interpret] → s →[ϕ_k]→ u_k →[F_k(x_k, u_k)]→ o_k →[ψ_k]→ z_k →[LLM synthesis]→ ŷ

**MCP instantiation.** Each FM is exposed as a remote MCP service with a declared schema over U_k. ϕ_k becomes the LLM's structured tool call; the MCP server retrieves x_k, runs F_k(x_k, u_k), returns o_k; ψ_k transforms o_k to natural language at the boundary. No FM needs to "speak" language; no LLM needs to "understand" raw FM outputs.

### EywaAgent

A_eywa = (A_LLM, F, ϕ, ψ, C) where C : S → {invoke, skip} is a per-step *control policy*.

```text
At step t with state s^t:
  if C(s^t) = skip:    z^t = A_LLM(s^t)                    # language-only branch
  if C(s^t) = invoke:  u = ϕ(s^t); o = F(x, u); z^t = ψ(o)  # delegate to FM
  s^{t+1} = s^t ∪ {z^t}
```

Crucially, EywaAgent **subsumes** language-only agents (set C ≡ skip) while strictly enlarging the function class.

**Theorem 3 (strict improvement).** Under Assumption 1, with F_LLM and F_Eywa the function classes induced by language-only and Eywa agents respectively,
  inf_{f ∈ F_Eywa} E [ℓ(f(x), y*)]  <  inf_{f ∈ F_LLM} E [ℓ(f(x), y*)].

The proof (Appendix A.3) is direct: skip-only recovers F_LLM; the invoke branch unlocks new optima where ϕ + F + ψ outperforms LLM-on-serialized.

### EywaMAS — plug-and-play heterogeneous MAS

M_Eywa = (A, G) where A is a *mixed* set of LLM agents and EywaAgents and G is any conventional topology (debate, hierarchical, sequential, looped). The system reduces to a vanilla MAS when no EywaAgents are present, and lifts the achievable optimum strictly when any EywaAgent replaces an LLM agent (Appendix A.4).

The plug-and-play property matters operationally: existing MAS frameworks (MetaGPT, ChatDev, AutoGen) can be upgraded by swapping individual agents without redesigning the topology.

### EywaOrchestra — dynamic orchestration

Configuration space C induced by candidate language models M_LLM, candidate FMs M_FM, and topology pool Π.

Conductor P (an LLM) maps task input (q, x) to a configuration c ∈ C; the system instantiates c and executes.

Algorithm 1:
1. c ← P(q, x)
2. instantiate heterogeneous system from c
3. execute on (q, x) and return ŷ

Define R*_fixed = min_c E_τ ℓ(F_c, y*) and R_oracle = E_τ [min_c E ℓ(F_c, y*)]. By construction R_oracle ≤ R*_fixed, with strict inequality when different task regions favor different configurations. EywaOrchestra trades a small utility loss vs the best fixed config for a large cost reduction by *adapting* the topology and FM selection per task — a Pareto improvement on the cost dimension.

### EywaBench

Multi-task, multi-domain, multi-modality:
- Domains: physical (material, energy, space), life (biology, clinic, drug), social (economy, business, infrastructure).
- Modalities: natural language, time series, tabular.
- Sources: DeepPrinciple, MMLU-Pro, fev-bench, TabArena.
- Unified utility metric u ∈ [0, 1]: soft-match for language, normalized prediction error for time-series/tabular.

## Empirical results

### Headline (Table 1, condensed to overall numbers)

| Method | Utility | Tokens | Latency (s) |
|---|---:|---:|---:|
| Single-LLM-Agent | 0.6154 | 4,469 | 25.22 |
| **EywaAgent** | **0.6558** (+6.6%) | **3,137** (−29.8%) | **22.78** (−9.7%) |
| Refine MAS | 0.6294 | 8,673 | 60.59 |
| Debate MAS | 0.6460 | — | — |
| MoA (heterogeneous LLMs) | 0.6273 | — | — |
| X-MAS (heterogeneous LLMs) | 0.6188 | — | — |
| **EywaMAS** | **0.6761** (best) | 11,214 | 72.11 |
| **EywaOrchestra** | **0.6746** | 8,335 (−25.8% vs MAS) | 48.16 (−20.5% vs MAS) |

Five observations:
- **Single-agent EywaAgent is a Pareto improvement.** +6.6% utility and −30% tokens vs LLM-only — the rare upgrade that simultaneously lifts quality and cuts cost.
- **EywaMAS beats every conventional MAS.** +7.4% over Refine, +4.6% over Debate, +7.8% over MoA, +9.3% over X-MAS.
- **LLM-only heterogeneity is *not* enough.** MoA (0.6273) and X-MAS (0.6188) underperform homogeneous Debate (0.6460). Mixing different LLMs adds little; mixing LLMs with FMs adds a lot.
- **Domain-skewed gains.** EywaAgent already saturates economy (0.8048) and business (0.7371) sub-domains in single-agent mode; multi-agent adds overhead.
- **EywaOrchestra is the cost-quality sweet spot.** 99.8% of EywaMAS utility (0.6746 / 0.6761) at ~70% of its cost — adaptive topology amortizes when single-agent is enough.

### Robustness

- **LLM temperature** ∈ [0.0, 1.0] — Eywa methods stable; peak around 0.6–0.7.
- **FM temperature (TabPFN)** — robust across calibration range.
- **Prompt design** — ReAct and detailed slightly best; Eywa robust across default / detailed / CoT / ReAct.
- **LLM backbone (Table 2).** gpt-4.1-nano (0.5680) → gpt-5-nano (0.6558) → gpt-5-mini (0.6640). Stronger backbones help, with the same Eywa structure intact.

The robustness picture matters: the +6.6% utility gain is **structural**, not a hyperparameter accident.

## Variants and ablations

- **Single vs Multi vs Orchestrated** — utility 0.6558 → 0.6761 → 0.6746 (with the orchestra variant cheaper).
- **Topology pool Π** — debate is the EywaMAS default; the conductor in EywaOrchestra picks across sequential / hierarchical / looped / custom.
- **FM selection** — the conductor automatically attaches the right FM (Chronos for time series, TabPFN for tabular) per task domain.
- **Prompt strategy** — default / detailed / CoT / ReAct all produce gains; ReAct + detailed is marginally best.
- **Backbone** — three GPT scales tested, monotone improvement.

## Failure modes and limitations

- **Conductor scalability.** A single LLM-conductor scales poorly to large configuration spaces. Future work: learned routers or search-based selection (cf. [87-routellm](87-routellm.md)).
- **Hand-designed Tsaheylu interfaces.** ϕ_k and ψ_k are hand-implemented per FM. Adding a new FM requires writing an MCP wrapper and adapter — not fully automated.
- **Domain coverage.** EywaBench is broad across science but doesn't include vision, audio, or symbolic representations. Generalization to those modalities is unclear.
- **Cost-quality trade-off in EywaOrchestra.** The 0.6746 vs 0.6761 gap is small but real; latency-insensitive applications may prefer the fixed EywaMAS.
- **Proof-of-concept FMs.** Chronos and TabPFN are general-purpose; integration with truly specialized models (AlphaFold, GraphCast, ChGNet) is not yet demonstrated.

## When to use, when not

**Use** for scientific tasks with heterogeneous modalities (time-series forecasting, tabular prediction, knowledge synthesis); when domain-specific FMs exist and can be wrapped with MCP; when both quality and token-efficiency matter; for adaptive task solving where complexity varies (orchestra picks single vs multi); for tasks requiring both language reasoning (planning, synthesis) and specialized computation.

**Don't use** for pure-language tasks where domain FMs offer no advantage; for unstructured modalities (image / audio / video) without an FM available; for real-time low-latency requirements where multi-agent overhead is prohibitive (mitigated but not eliminated by Orchestra); for proprietary FMs without programmable access; for new scientific domains where Tsaheylu interfaces would have to be hand-tuned for an untested wrapper.

## Implications for harness engineering

- **MCP graduates to a foundation-model federation interface.** [07-model-context-protocol](07-model-context-protocol.md) framed MCP as a tool protocol; Eywa shows the same protocol carries *native foundation-model invocation*. Any harness with MCP support is a candidate Eywa host. The compiler/adapter pair is just two new MCP-server-side hooks.
- **Cross-modality > cross-LLM heterogeneity.** [98-diversity-collapse-mas](98-diversity-collapse-mas.md) showed that "multi-agent ≠ multi-perspective" — mixing LLMs structurally collapses to similar reasoning. EywaMAS demonstrates the alternative: heterogeneity that matters is *modality* heterogeneity, not *model-vendor* heterogeneity. Production multi-agent harnesses should plan FM federation as a top-line requirement, not an afterthought.
- **Adaptive topology is a harness primitive.** [42-langchain-deep-agents](42-langchain-deep-agents.md) and [73-multica](73-multica-managed-agents-platform.md) ship with fixed topology defaults. EywaOrchestra's conductor is a 50-line addition that pays for itself: pick single-agent for simple tasks, multi-agent for complex ones, the right topology in each case. Pairs naturally with the routing literature ([86-frugalgpt](86-frugalgpt.md), [87-routellm](87-routellm.md)) — but routing model choice and routing topology are *different* axes; Eywa adds the second.
- **Subagent delegation as native computation.** [02-subagent-delegation](02-subagent-delegation.md) describes orchestrator-worker for LLM workers; Eywa generalizes the worker to *any FM with a schema*. The orchestrator-worker pattern is the same; the worker pool just got bigger.
- **Per-step delegation policy C(s).** The control policy "should I invoke the FM or continue reasoning?" is a non-trivial harness design point. Default to "invoke when the task input has informative non-language structure" — but the threshold itself is a harness choice that should be measured. Maps to the broader confidence-driven router pattern (cf. RouteLLM follow-ups).
- **MetaGPT-style role agents become Eywa-style domain agents.** [20-metagpt-role-based-workflows](20-metagpt-role-based-workflows.md) assigns roles by SDLC function (architect, engineer, QA). Eywa shows the same template can assign roles by *modality* (time-series specialist = Chronos-backed agent, tabular specialist = TabPFN-backed agent, language coordinator = LLM-only agent).
- **Pareto-improving upgrade pattern.** Most agentic upgrades trade utility for cost (multi-agent → better, slower) or cost for utility (caching → cheaper, equivalent). Eywa is a structural upgrade that wins on both axes, like [86-frugalgpt](86-frugalgpt.md)'s cascade routing did for general LLM serving. Both should be in the default toolbox before any harness ships to production.
- **EywaBench template for multi-modality benchmarks.** Three domains × three modalities × unified utility metric is a reusable template. Most agent benchmarks are single-modality (text reasoning, code, web browsing); EywaBench is the first to grade across language + time-series + tabular with one number. Harness eval suites should include at least one multi-modality benchmark of this shape.

The one-line takeaway for harness designers: **MCP isn't just a tool protocol — it's the federation API for non-LLM foundation models, and "what's the right FM for this sub-task?" is now as important as "what's the right LLM?"**
