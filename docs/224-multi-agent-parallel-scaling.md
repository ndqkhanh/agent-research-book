# 224 — Multi-Agent and Parallel Scaling: where "more agents" is and isn't all you need

**Papers.** Junlin Wang, Jue Wang, Ben Athiwaratkun, Ce Zhang, James Zou — *Mixture-of-Agents Enhances Large Language Model Capabilities* — arXiv:2406.04692 — Together AI / Stanford / Duke — 2024. Yilun Du, Shuang Li, Antonio Torralba, Joshua B. Tenenbaum, Igor Mordatch — *Improving Factuality and Reasoning in Language Models through Multi-Agent Debate* — arXiv:2305.14325 — MIT / Google Brain — 2023. Junyou Li, Qin Zhang, Yangbin Yu, Qiang Fu, Deheng Ye — *More Agents Is All You Need* — arXiv:2402.05120 — Tencent — 2024. Companion: AgentVerse (Chen et al., arXiv:2308.10848), MetaGPT ([91](91-metagpt-deep.md)), ChatDev ([92](92-chatdev.md)), Diversity-Collapse-MAS ([98](98-diversity-collapse-mas.md)), Recursive Multi-Agent Systems ([189](189-recursive-multi-agent-systems.md)), Multi-Hop Multi-Agent Reckoning ([202](202-multi-agent-multi-hop-reckoning-2026.md)).

**One-line definition.** Empirical work showing that **stacking K agents in parallel and aggregating their outputs (debate, mixture-of-agents, sample-vote)** improves accuracy on hard reasoning tasks **up to a model- and task-specific saturation**, with **MoA's 6 open-weights agents (Together-AI Reference 2024) reaching AlpacaEval 2.0 LC win-rate 65.1 % vs GPT-4o's 57.5 %** at a fraction of the cost — but the curves bend sharply when *agents collapse to the same priors* ([98](98-diversity-collapse-mas.md)) or *coordination overhead exceeds capability gain*.

## Why this paper matters (multi-agent is the fourth scaling axis, with strong but bounded returns)

After pretraining ([216](216-pretraining-scaling-laws-foundation.md)), per-call test-time compute ([217](217-test-time-compute-scaling.md)), and trajectory budget ([237](237-agent-trajectory-scaling.md)), the fourth axis is **parallel agents**: instantiate K base-model instances, run them on the same problem, and aggregate. The aggregation can be debate (Du-2023), mixture-of-agents stacking (MoA, Wang-2024), majority vote with verification ([223](223-verifier-and-best-of-n-scaling.md)), structured organization (MetaGPT, ChatDev, AgentVerse), or recursive composition ([189](189-recursive-multi-agent-systems.md)). The axis matters because at the agent-era frontier in 2025–2026, **a 6-agent open-weights MoA setup costing roughly $1.50 per task beats a $20 GPT-4o single-call** on AlpacaEval, MT-Bench, and FLASK — making it the dominant unit-economics frontier for many production deployments.

Wang-2024 (MoA) is the cleanest paper. They define a **layered architecture**: layer 1 contains K reference agents (e.g. Qwen-110B-Chat, WizardLM-22B, LLaMA-3-70B-Instruct, Mixtral-8x22B, dbrx-instruct, Qwen-72B-Chat); each receives the prompt and produces an answer. Layer 2 contains an *aggregator* agent that receives the prompt + all K layer-1 responses and produces a refined answer. Optionally stack additional aggregator layers. The headline: **MoA-Lite (3 reference + 1 aggregator using Qwen-110B) reaches 59.3 LC win-rate on AlpacaEval 2.0** versus GPT-4o 57.5; the full **MoA (6 reference + 1 aggregator)** reaches **65.1**, beating GPT-4 Omni convincingly. They report the same trend on MT-Bench (9.25 vs GPT-4o 9.19) and FLASK. The key insight: **"collaborativeness" of LLMs** — each model produces better outputs when shown other models' answers, *even if those answers are individually worse than its own*.

Du-2023 (multi-agent debate) preceded MoA and used a different structure: K agents produce answers, then K rounds of "see the others' answers; revise yours." After a few rounds, agreement converges to a more accurate answer than any single agent. Empirically, debate added 6–10 absolute points on math benchmarks and reasoning tasks for GPT-3.5-class models. The mechanism: peer-induced revision functions like a *distributed verifier* — wrong answers get pruned through critique rounds.

Li-2024 ("More Agents Is All You Need") strips the structure to its bones: just sample K times and majority-vote. The accuracy of the ensemble follows a smooth power-law-like curve in K, sharply rising from K=1 to K=8 and saturating around K=20–40 depending on task. They report consistent gains across LLama-2-Chat, GPT-3.5, GPT-4 — the simplest possible multi-agent recipe scales cleanly.

Take this evidence seriously and three things change. **First**, you stop thinking of multi-agent as a "system architecture choice" and start thinking of it as a **scaling axis with its own returns curve**, comparable to TTC and trajectory length. **Second**, you understand that the **right K depends on diversity**, not just count — six different open-weights models in MoA outperform six copies of the same model because diversity, not headcount, drives the gain ([98-diversity-collapse-mas](98-diversity-collapse-mas.md)). **Third**, you architect aggregation as a **first-class harness component**, not a free addition: aggregator selection, role specialization, debate protocol, and verifier integration are co-equal with model selection.

## Problem it solves (extract more capability per unit inference cost via parallelism)

1. **Single-agent test-time compute saturates.** Past N=64 BoN samples on MATH, additional samples help slowly (Lightman 2023, [223](223-verifier-and-best-of-n-scaling.md)). Diversifying *which* model produces samples opens a new axis with steeper returns.
2. **Open-weights ensembles can beat closed flagship at lower cost.** MoA at ~$1.50/task beats GPT-4o at ~$20/task on AlpacaEval and MT-Bench. The economics are favourable when the user-facing latency tolerates parallel calls.
3. **Verifier-augmented BoN (215) is single-source.** It samples N from one generator. Multi-agent generalizes: N samples from K different generators, aggregated with a verifier or aggregator. Sample diversity rises; oracle pass@N rises; closed gap to oracle is verifier-bound.
4. **Hard reasoning benefits from peer critique.** Du-2023 shows that GPT-3.5-class models in 3-round debate produce ~+10 % on GSM8K, MATH, and reasoning benchmarks vs single-agent — at modest extra compute.
5. **Role specialization unlocks complex workflows.** MetaGPT ([91](91-metagpt-deep.md)), ChatDev ([92](92-chatdev.md)), AgentVerse — software-engineering agent teams with PM/Architect/Engineer/QA roles outperform single-agent attempts at multi-file software tasks.
6. **Scaling law with K must be characterized.** Without a curve, you cannot pick K. Li-2024 fits explicit K-scaling curves; MoA characterizes layer count and reference-agent diversity ablations.

## Core idea in one paragraph

A **multi-agent system** instantiates K agents (instances of one or more base models) running on the same problem, with an **aggregation mechanism** that converts the K outputs into one. Aggregation regimes form a spectrum: (a) **majority vote** (Li-2024) — simple, robust, saturates around K=20–40; (b) **debate** (Du-2023) — K agents iterate with peer-critique; capability climbs over rounds; (c) **mixture-of-agents** (Wang-2024) — layered architecture with an aggregator that synthesizes K reference outputs; outperforms voting and debate on AlpacaEval / MT-Bench / FLASK; (d) **role-specialized organizations** (MetaGPT, ChatDev, AgentVerse) — agents have distinct prompts/roles producing structured artefacts with division of labour; best for multi-step workflows like software development; (e) **recursive composition** ([189](189-recursive-multi-agent-systems.md)) — agents instantiate sub-agents recursively, treating multi-agent as a meta-language. The key empirical finding across all regimes: capability scales smoothly in K **provided agents are diverse**; with K copies of the same model and seed, returns plateau around K=4–8; with K different models or different specializations, returns extend to K=20+. Aggregator quality is the load-bearing component — a strong aggregator (or PRM verifier, [223](223-verifier-and-best-of-n-scaling.md)) recovers most of the pass@K oracle gap, a weak one wastes the compute. Diversity collapse ([98](98-diversity-collapse-mas.md)) is the dominant failure mode: agents converge to the same priors via shared training data, shared prompts, or in-context contagion, after which K-scaling gains evaporate.

## Mechanism (step by step)

### (a) Majority vote (Li-2024)

Sample K independent rollouts from one model with non-zero temperature. For each problem, return the modal final answer. For math/code/structured tasks, "modal answer" means string- or symbolic-equality match. Accuracy curve `acc(K)` rises monotonically and saturates around K=20–40 depending on base model and task; Li-2024 reports +10–15 % absolute on MATH, GSM8K, MMLU at K=40 for LLaMA-2-Chat-7B.

**Strength:** simplest possible, no extra training, no aggregator.
**Weakness:** fails when modal answer is wrong; ignores quality.

### (b) Multi-agent debate (Du-2023)

Round 0: K agents independently answer. Round t: each agent receives others' round (t-1) answers and produces a revised answer. After T rounds, take majority vote or final-round agreement.

Du-2023 §4: with K=3, T=3, GPT-3.5-Turbo: GSM8K rises from 75 % single-agent to 85 % debate; MATH rises from 22 % to 30 %; reasoning benchmarks similar. Scaling K beyond 3 helps modestly; T beyond 4 saturates.

**Strength:** peer critique acts as distributed verifier; works without trained PRM.
**Weakness:** chain-of-thought contagion — if one agent's wrong reasoning is persuasive, debate converges to the wrong answer.

### (c) Mixture-of-Agents (Wang-2024) — the strongest result on AlpacaEval

Layer 1: K reference agents. Each independently answers prompt p.
Layer 2: aggregator agent A receives prompt p and the K reference answers `r_1, …, r_K`, with a structured aggregator-prompt: "*Below are responses from other models. Use them to produce a high-quality response.*" A produces final answer.
Optionally: layer 3+ takes layer-2 output and another set of K references; deeper stacks.

Wang-2024 §4 reports:

- **MoA-Lite** (3 ref + 1 aggregator, all Qwen-110B): **AlpacaEval 2.0 LC = 59.3** vs GPT-4o 57.5.
- **MoA** (6 ref + 1 aggregator): **AlpacaEval 2.0 LC = 65.1** vs GPT-4o 57.5.
- **MT-Bench**: MoA = 9.25; GPT-4o = 9.19.
- **FLASK**: MoA wins on robustness, correctness, factuality, depth; ties or loses on conciseness.

Reference agents: Qwen-1.5-110B-Chat, Qwen-1.5-72B-Chat, WizardLM-8x22B, LLaMA-3-70B-Instruct, Mixtral-8x22B-Instruct-v0.1, dbrx-instruct. Aggregator: Qwen-1.5-110B-Chat (open-weights).

### (d) Role-specialized organizations

[91-metagpt-deep](91-metagpt-deep.md), [92-chatdev](92-chatdev.md), AgentVerse — agents are assigned roles (Product Manager, Architect, Engineer, QA, Reviewer). Each role has its own system prompt and tool set. Workflow is structured: PM produces requirements → Architect produces design → Engineer produces code → QA produces tests → Reviewer integrates. The system performs better than a single-agent monolithic prompt on multi-file, multi-stage software tasks. The agentic equivalent of a Conway's-law-aware org chart.

[206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md), [205-lobehub-collaborative-teammate-platform](205-lobehub-collaborative-teammate-platform.md) — productized, persona-aware, identity-portable team architectures.

### (e) Diversity is what scales, not headcount

[98-diversity-collapse-mas](98-diversity-collapse-mas.md) — six copies of the same model with different prompts give marginal gain over one; six different models give large gain. Diversity sources:

- **Different model families** (Qwen + Llama + Mixtral + dbrx) — strongest.
- **Different model sizes** (8B + 22B + 70B + 110B) — moderate.
- **Different system prompts / personas** (skeptic + advocate + neutral) — modest.
- **Different temperature / decoding parameters** — weakest, but free.
- **Different tool sets / capabilities** — strong, but task-bound.

A diversity audit before scaling K is a high-ROI harness investment.

### (f) The aggregator dominates

A weak aggregator (or majority vote when answer space is continuous/structured) caps the system at single-agent performance. Strong aggregators (Wang-2024's structured aggregator-prompt; trained PRM-as-aggregator; cross-model adversarial verifier from [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md)) close most of the pass@K oracle gap.

### (g) Recursive multi-agent

[189-recursive-multi-agent-systems](189-recursive-multi-agent-systems.md) — agents instantiate sub-agents recursively, treating "more agents" as a recursive language with bounded depth. Capability scales with both K (breadth) and recursion depth (depth); the analogy is parallel-vs-sequential test-time compute applied to the multi-agent axis.

## Empirical results (table)

**Table 1 — MoA / MoA-Lite vs GPT-4o on AlpacaEval 2.0 (Wang-2024 §4)**

| System | AlpacaEval 2.0 LC win-rate | Cost / task | Notes |
|---|---:|---:|---|
| GPT-4o (May 2024) | 57.5 | ~$20 | Single call |
| MoA-Lite (3+1, Qwen-110B) | 59.3 | ~$0.50 | Open-weights |
| MoA (6+1, mixed open-weights) | **65.1** | ~$1.50 | Open-weights mix |
| Reference single-agent (Qwen-110B) | 51.0 | ~$0.30 | Single call |

**Table 2 — Du-2023 debate gains (GPT-3.5-Turbo, K=3, T=3)**

| Benchmark | Single agent | 3-agent debate | Δ |
|---|---:|---:|---:|
| GSM8K | ~75 % | ~85 % | **+10** |
| MATH | ~22 % | ~30 % | **+8** |
| Biographies (factuality) | ~63 % | ~73 % | **+10** |
| Strategy QA | ~71 % | ~80 % | **+9** |

**Table 3 — Li-2024 majority-vote scaling (LLaMA-2-Chat 7B)**

| K (samples) | MATH | GSM8K | MMLU |
|---:|---:|---:|---:|
| 1 | 11 % | 32 % | 47 % |
| 8 | 18 % | 51 % | 54 % |
| 20 | 22 % | 58 % | 58 % |
| 40 | **23 %** | **60 %** | **59 %** (saturated) |

**Table 4 — Diversity vs headcount (MoA ablation, illustrative)**

| Setup | AlpacaEval 2.0 LC |
|---|---:|
| 6× Qwen-110B (same model) | ~55 % |
| 6× different open-weights models | **~65 %** |
| 3 different + 3 same | ~60 % |

**Table 5 — Role-specialized organizations vs flat multi-agent on coding (MetaGPT, ChatDev)**

| System | HumanEval | MBPP | Software-Dev quality (LLM-judged) |
|---|---:|---:|---:|
| Single GPT-3.5 | ~67 % | ~77 % | ~5/10 |
| Flat 3-agent debate | ~74 % | ~80 % | ~6/10 |
| MetaGPT (5 roles, structured) | ~85 % | ~87 % | ~8/10 |
| ChatDev (waterfall org) | ~85 % | ~87 % | ~8/10 |

## Variants and ablations

- **Layer count in MoA.** 1 layer = no aggregator (just K independent samples); 2 layers = standard MoA; 3+ layers = recursive aggregation. Wang-2024 found 2 layers near-optimal; 3-layer gives small additional gain at 3× cost.
- **Aggregator model choice.** Wang-2024 ablates: smallest aggregator (Qwen-22B) gives 91 % of full-aggregator (Qwen-110B) performance — *the aggregator does not need to be the strongest model*; it needs to be strong at synthesis.
- **Reference-agent diversity.** Top-N most diverse vs random-N: top-N wins by ~3 LC points at fixed K=6.
- **Self-consistency (Wang et al. 2022).** Predecessor of Li-2024; majority-vote with chain-of-thought sampling; same shape, smaller K range.
- **Agent debate with role asymmetry.** [98-diversity-collapse-mas](98-diversity-collapse-mas.md) — assigning persistent roles (skeptic vs advocate) raises diversity and prevents collapse.
- **Verifier-as-aggregator.** Replace MoA aggregator with a trained PRM ([223](223-verifier-and-best-of-n-scaling.md)) that selects best of K — works on math/code; less effective on open-ended generation.
- **Mixture of small specialized agents.** Domain-specialized small models (math-finetune + code-finetune + retrieval-augmented) outperform K copies of one generalist on cross-domain tasks.
- **Cross-language aggregation.** Multi-lingual ensembles agents reasoning in different languages; modest gains, useful for translation-heavy tasks.

## Failure modes and limitations

- **Diversity collapse.** [98-diversity-collapse-mas](98-diversity-collapse-mas.md) — agents converge to shared priors via shared pretraining corpora, shared prompt format, or in-context contagion. Returns plateau and even reverse beyond a threshold K.
- **Persuasive-but-wrong agents poison debate.** Du-2023 acknowledges: a confidently-wrong rollout can shift majority. Asymmetric roles (skeptic) and adversarial cross-model pairs ([02](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md)) mitigate.
- **Aggregator quality bounds the system.** A weak aggregator caps performance at single-agent; the aggregator is the *new* bottleneck.
- **Coordination overhead.** With K=10+, prompt assembly, parallel inference, and aggregation latency dominate. Costs can outpace single-flagship calls.
- **Cost scales linearly with K.** ROI vs flagship single-call requires K · cost(reference) + cost(aggregator) < cost(flagship). At Qwen-110B + GPT-4o pricing this is favourable; at parity it is not.
- **Chain-of-thought contamination.** When agents see others' reasoning chains, they often anchor on visible (sometimes wrong) reasoning rather than reasoning afresh.
- **Reproducibility under non-determinism.** K parallel non-deterministic agents with non-deterministic aggregator → high variance per run. Production needs seeded runs and confidence bounds.
- **Role-specialized failure cascades.** [91-metagpt-deep](91-metagpt-deep.md) — if the PM agent produces bad requirements, downstream roles inherit them. Need verification-at-handoff.
- **Multi-hop reasoning ≠ multi-agent.** [202-multi-agent-multi-hop-reckoning-2026](202-multi-agent-multi-hop-reckoning-2026.md) — multi-agent does not magically solve multi-hop; sometimes it just spreads errors across more surfaces.
- **Latency for synchronous use.** MoA at K=6 with sequential aggregation has 2× the latency of a single call, even when reference agents parallelize. Not suitable for sub-second user-facing tasks.

## When to use, when not

**Use multi-agent scaling** when (a) the task admits multiple plausible approaches and aggregation can pick the best (math, code, factual QA, structured planning); (b) open-weights agents are available at lower per-call cost than the closed flagship; (c) latency tolerates 2–4× single-call wall-clock; (d) you can audit and enforce diversity (different models, different roles, adversarial verifier); (e) the task has high enough verifier or aggregator quality to close the pass@K gap. The strongest case is software development with role-specialized organizations ([91](91-metagpt-deep.md), [92](92-chatdev.md)) and AlpacaEval-style open-ended generation with MoA.

**Do not** use multi-agent when latency is a hard constraint, when you cannot enforce diversity (K copies of the same model collapse), when no good aggregator/verifier exists, when the task is intrinsically single-pass (latency-bounded chat), or when the cost of K · agent · coordination exceeds a single flagship call. Avoid recursive multi-agent without bounded depth — runaway compute is the failure mode.

## Implications for harness engineering

- **Treat K as a scaling axis with its own ROI curve.** Plot accuracy(K) per task; pick K from the curve, not by intuition. [13-daemon-and-scheduler](../projects/polaris/docs/blocks/13-daemon-and-scheduler.md), [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md).
- **Diversity audit before scaling.** [98-diversity-collapse-mas](98-diversity-collapse-mas.md) — measure inter-agent answer disagreement at K=2, 4, 8; if it collapses below a threshold, K-scaling is wasted.
- **Aggregator is a first-class component.** Train, version, monitor — like a verifier ([223](223-verifier-and-best-of-n-scaling.md)). Cross-model adversarial pair recommended ([02](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md)).
- **MoA shape is reusable.** Layer-1 ensemble + layer-2 aggregator is a generic harness pattern: replace ensemble with subagents ([02-subagent-delegation](02-subagent-delegation.md), [18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md)) for code or research; with retrieved evidence ([79-skill-rag](79-skill-rag.md)) for QA.
- **Role-specialized agents for multi-step workflows.** [91-metagpt-deep](91-metagpt-deep.md), [92-chatdev](92-chatdev.md), [205-lobehub-collaborative-teammate-platform](205-lobehub-collaborative-teammate-platform.md), [206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md) — Conway's-law-aware org charts for software, research, design.
- **Debate with verifier checkpoints.** [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md) — debate without a verifier is vulnerable to persuasive-but-wrong; run a verifier between rounds.
- **Cross-channel verification mandatory.** [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md), [19-verifier-cross-channel](../projects/polaris/docs/blocks/19-verifier-cross-channel.md) — aggregator family ≠ generator family ≠ verifier family.
- **Multi-hop ≠ multi-agent; combine carefully.** [199-search-r1-multi-hop](199-search-r1-multi-hop.md), [202-multi-agent-multi-hop-reckoning-2026](202-multi-agent-multi-hop-reckoning-2026.md) — multi-hop reasoning structure is orthogonal; combining requires explicit hop coordination.
- **Recursive depth bounded by cost gates.** [189-recursive-multi-agent-systems](189-recursive-multi-agent-systems.md), [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md) — recursive instantiation needs hard depth cap and budget gate.
- **Skill memory amortizes across agents.** [05-reasoning-bank](../projects/polaris/docs/blocks/05-reasoning-bank.md), [81-reasoningbank](81-reasoningbank.md), [156-heavyskill-rlvr](156-heavyskill-rlvr.md) — shared memory across agents lets aggregator integrate prior experience, not just current samples.
- **Collaborative-AI canon as productization template.** [206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md) — typed personal memory + multi-agent + MCP marketplace + branching + co-evolution + local-first privacy as the production-grade frame.
- **HIR observability per agent.** [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md) — log per-agent (model, prompt, response, score, contribution-to-aggregate). Enables diversity audits, post-hoc debugging, reward attribution.
- **Cost discipline on K.** [p13-cost-discipline](../projects/polaris/docs/research/p13-cost-discipline.md), [p15-crewai-flows](../projects/polaris/docs/research/p15-crewai-flows.md) — bright-line spend gates per request; cap recursive depth.

**One-line takeaway for harness designers.** **Multi-agent is the fourth scaling axis: capability rises smoothly in K so long as agents are *diverse* and the aggregator is *strong* — pick K from the curve, audit diversity before scaling, and treat the aggregator as a load-bearing trained model, not a scoring step.**
