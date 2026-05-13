# 189 — Recursive Multi-Agent Systems: Scaling Agent Collaboration Through Recursive Latent-Space Computation

**Paper.** Xiyuan Yang, Jiaru Zou, Rui Pan, Ruizhong Qiu, Pan Lu, Shizhe Diao, Jindong Jiang, Hanghang Tong, Tong Zhang, Markus J. Buehler, Jingrui He, James Zou — *Recursive Multi-Agent Systems* — arXiv:2604.25917v1 — UIUC / Stanford / NVIDIA / MIT — April 28, 2026. Project page: https://recursivemas.github.io/. Code: https://github.com/RecursiveMAS/RecursiveMAS (~310 stars). Models: https://huggingface.co/RecursiveMAS. HuggingFace daily papers: 242 upvotes (week of 2026-W18).

**One-line definition.** RecursiveMAS recasts a heterogeneous multi-agent system as a **single recursive language model** in which each frozen agent plays the role of one "layer," consecutive rounds pass *continuous hidden states* between agents (no token decoding except in the final round), and only a tiny **RecursiveLink** (≈13M params, 0.31% of system) is trained — yielding **+8.3% accuracy at 75.6% fewer tokens and 2.4× speedup** over text-mediated MAS at matched recursion depth, while resolving the gradient-vanishing problem that has prevented prior latent-MAS work from being trainable end-to-end.

## Why this paper matters

Multi-agent systems and recursive language models have been on parallel tracks for the last three years and never met. Recursive LMs (Geiping's recurrent depth, Bae et al.'s Mixture-of-Recursions, Zhu et al.'s LoopLM/Ouro, Jolicoeur-Martineau's tiny recursive nets) demonstrated that *unrolling a single network deeper at inference time* can substitute for parameters — a token-efficient form of test-time compute. Multi-agent systems (CAMEL, ChatDev, MoA, MALT, TextGrad) demonstrated that *coordinating multiple specialized models through text* can outperform any individual model. RecursiveMAS is the first paper to lift recursive scaling from the *single-network* level to the *system* level, and the consequences are larger than the simple combination would suggest.

Three structural advantages emerge that neither parent line could achieve alone. First, **inter-agent communication moves into latent space**: the last-hidden-state of one agent is mapped, via a 13M-parameter linear+MLP residual block, into the input-embedding space of the next agent — skipping vocabulary projection (the m·|V|·d_h cost), token decoding, and re-tokenization on every hop. Second, **gradient flow survives recursion**: the paper proves (Theorem 4.1) that text-mediated recursive systems suffer ‖∂ℛ/∂h‖₂ ≤ O(ε) gradient vanishing because the softmax-argmax bottleneck collapses the Jacobian when the model is confident; the residual structure of the RecursiveLink keeps ‖∂ℛ/∂h‖₂ ≥ Ω(1) with high probability, making the system *trainable* with a single end-to-end cross-entropy on the final round. Third, **frozen heterogeneous specialists become composable as RLM layers**: a math-tuned, code-tuned, and science-tuned LLM can be run as `(A_math → A_code → A_science)` for n rounds, gradients backpropagating through every link, only ~13M parameters touching the optimizer.

The empirical result that lands hardest is the *Pareto-improving recursion curve*: at r=3 the system uses 75.6% fewer tokens than text-mediated MAS, runs 2.4× faster, and posts **+8.3% absolute accuracy averaged across nine benchmarks** (88.0 MATH500, 86.7 AIME25, 86.7 AIME26, 66.2 GPQA-Diamond, 42.9 LiveCodeBench, 79.3 MedQA). For comparison, MoA — the strongest prior text-mediated MAS — averages 4.1pp behind, and LoopLM (single-agent recursive) averages 6.3pp behind. Take this paper seriously and three downstream things change. (1) **Inter-agent text becomes a vestigial cost** — once the latent-bridge pattern is mainstream, any production MAS that decodes text between agents is paying a tax it cannot justify. (2) **The "harness owns orchestration, model owns reasoning" split now has a third tier**: the *recursive-link* layer that owns continuous-state translation between heterogeneous models, sitting between harness and model. (3) **Test-time scaling has a system-level axis** — instead of (or alongside) extending CoT length or sampling more rollouts, you can deepen the recursion and amortize the cost across token savings.

## Problem it solves

- **Text-mediated MAS pays the full vocabulary projection on every hop.** When agent A_i finishes a turn it materializes m tokens at cost m·|V|·d_h plus context-extension cost; agent A_{i+1} re-tokenizes and re-embeds that text. With four agents and three rounds, you pay this twelve times. RecursiveMAS pays it *once* — only the final round decodes.
- **Latent-MAS prior work was stuck at the inference-only stage.** Du et al.'s "communicate entirely in latent space," Cache-to-Cache, KVcomm, and Zou et al.'s latent collaboration all proposed swapping text for hidden states between agents, but none could train the system end-to-end because gradient flow through softmax-bottlenecked text channels collapsed under entropy-confidence.
- **Recursive language models were stuck at single-model scope.** Geiping's recurrent depth, LoopLM/Ouro, Mixture-of-Recursions, and tiny recursive nets all unroll a *single* network to amortize parameters. None addressed the system-level question — what if the unrolled "layer" is itself a heterogeneous LLM specialist?
- **Distributional mismatch between hidden-state output and embedding-input.** A frozen LLM's last-hidden-state lives in a different distribution than its input-embedding space; feeding raw last-hidden into the next layer's input destabilizes generation. The RecursiveLink's job is to bridge that gap with minimal trainable parameters.
- **Cross-agent dimension mismatch.** Heterogeneous models (Llama-3.1-8B vs Qwen3-7B vs DeepSeek-Coder-7B) have different hidden dimensions; a residual-only bridge cannot handle this. The Outer Link adds a learned linear projection W₃ to absorb the dimension shift.

## Core idea in one paragraph

Treat a MAS of N frozen heterogeneous agents as a single recursive computation `H^{(r)} = f_θ(H^{(r-1)})` for r = 1…n, where one "function evaluation" is a sequential pass through all N agents. Between agents, replace text generation with **direct hidden-state transfer through a RecursiveLink** — a residual two-layer MLP (Inner Link, same dim) or a residual MLP plus learned linear projection (Outer Link, cross-dim) that maps the previous agent's last-hidden to the next agent's input embedding. Train in two stages: an **inner warm-start** that fits each agent's Inner Link on a cosine-similarity loss between the link's output and the ground-truth answer's input embedding (so each agent fluently generates "latent thoughts"), then an **outer end-to-end** stage that unrolls n rounds of the full system, decodes only the final round, and back-propagates a single cross-entropy loss through every link. Keep all LLM weights frozen; only the RecursiveLinks update. The result is a system whose function class strictly contains both single-LLM and text-mediated-MAS, with provably non-vanishing gradient flow under the entropy-confidence regime where text-mediated recursion fails.

## Mechanism (step by step)

### Notation and primitives

A system 𝒮 holds N agents 𝒜 = {A₁, …, A_N}, each parameterized by frozen f_{θ_i}, producing last-hidden-state H_i ∈ ℝ^{T × d_{h_i}} per turn. The system state ℋ = {H₁, …, H_N}. A task input x ∈ V* is tokenized once into token embeddings E ∈ ℝ^{T × d_h}; thereafter, all inter-agent communication is in continuous space.

**Eq. 1 — Latent generation within an agent.** For m latent thoughts:
  h_{t+1} = f_θ([E_{≤t}; h_t]),  t = 0, …, m−1
The agent autoregressively extends its last-hidden trajectory rather than decoding tokens. Saturation point empirically m ≈ 80 (Fig. 8 / Table 9).

**Eq. 2 — Recursive computation within a single agent.**
  H^{(0)} = E,  H^{(r)} = f_θ(H^{(r-1)}),  r = 1, …, n

**Definition 2.1 — Recursive MAS evolution.** With n recursion rounds:
  𝒮^{(0)} →^{H^{(1)}} 𝒮^{(1)} →^{H^{(2)}} … →^{H^{(n)}} 𝒮^{(n)}

Only 𝒮^{(n)}'s output projects through the LM head to produce text.

### RecursiveLink — the only trainable component

Two link types serve two roles. The **Inner Link** maps an agent's own last-hidden back to its own input-embedding space (intra-agent latent autoregression):

**Eq. 3 — Inner Link.**
  ℛ_in(h) = h + W₂ · σ(W₁ · h)

Two-layer MLP with GELU activation σ and residual connection. The residual is critical: it preserves semantic content; the MLP only learns the dense-to-shallow distributional shift between last-hidden and input-embedding.

The **Outer Link** maps one agent's last-hidden into the *next* agent's input-embedding space (inter-agent bridge), and must handle differing hidden dimensions:

**Eq. 4 — Outer Link.**
  ℛ_out(h) = W₃ · h + W₂ · σ(W₁ · h)

W₃ is a learned linear projection that absorbs the dimension change d_{h_i} → d_{h_{i+1}}. The MLP branch + linear branch sum gives both a learned-distribution shift and a baseline "identity-with-projection" path that gradients can flow through even when the MLP is poorly initialized.

### Two-stage training

**Stage 1 — Inner warm-start (per agent, frozen LLM).**
  ℒ_in = 1 − cos(ℛ_in(H), Emb_{θ_i}(y))
where y is the ground-truth answer text and Emb_{θ_i}(·) is agent i's input-embedding lookup. This regresses the inner link onto the answer's input-embedding so the agent can fluently *continue generating in latent space* on the next pass without seeing a tokenized prompt extension.

**Stage 2 — Outer end-to-end (whole system).**
  ℒ_out = CE(LM_head_{θ_N}(𝒮^{(n)}(𝒮^{(n-1)}(…𝒮^{(1)}(x)))), y)
Single cross-entropy on the final round's decoded tokens. Gradients flow through every Outer Link via the residual structure (the linear-plus-MLP path keeps the Jacobian well-conditioned; see Theorem 4.1 below).

Hyper-parameters from the paper: AdamW, lr 5e-4, cosine schedule, batch 4, max-seq 4096, m ≈ 80 latent thoughts, recursion rounds r ∈ {1, 2, 3}.

### Four collaboration patterns instantiated

The framework is topology-agnostic; the paper reports four canonical instantiations:

1. **Sequential** — Planner → Critic → Solver. Each agent specializes by role; latent state passes through three Outer Links per round.
2. **Mixture** — three domain specialists (Math, Code, Science) → Summarizer. The summarizer aggregates four parallel last-hidden streams. Mixture pattern lifts +6.2% over the best individual specialist.
3. **Distillation** — large Expert + small Learner. The Learner attends to the Expert's last-hidden through an Outer Link; the Learner alone runs at inference. AIME26 score 83.3 (Learner-alone) vs 90.0 (Expert) at 1.5× speed.
4. **Deliberation** — Reflector + Tool-Caller (Python or Tavily search). The Reflector emits latent intent; the Tool-Caller decodes a tool call (the only intermediate decoding) and feeds latent results back. +4.8% over the tool-calling agent baseline.

### Theory — why latent recursion is trainable where text recursion is not

**Proposition 3.1 — runtime advantage.** Text-MAS round cost is Θ(N(m|V|d_h + (t+m)d_h² + (t+m)²d_h)). RecursiveMAS round cost is Θ(N(m d_h² + (t+m)d_h² + (t+m)²d_h)). The vocabulary-projection term m|V|d_h is replaced by m d_h². Since |V| ≈ 10⁵ and d_h ≈ 4·10³, the token-decode cost is ~25× the per-token compute cost; eliminating it across N agents and n rounds is where the 75.6% token reduction at r=3 comes from.

**Theorem 4.1 — gradient stability.** Under "Realistic Assumptions" (App. A.2): when next-token entropy ≤ ε ≪ 1 (a confident model), the text-recursive Jacobian satisfies ‖∂ℛ_text / ∂h‖₂ ≤ O(ε) — gradients vanish through softmax-argmax. By contrast, RecursiveMAS's residual link satisfies, with probability ≥ 1 − δ:
  ‖∂ℛ / ∂h‖₂  ≥  Ω(1 − √((1/d_h) · log(1/δ)))

i.e. gradient norm stays near 1 across loops. Proof (App. A.3) uses the softmax-Jacobian collapse for the text branch and Kaiming-init spectral-norm bounds for the residual MLP. The corollary is operational: text-recursive SFT is unstable past 1–2 rounds; RecursiveMAS is stable at 3+ rounds.

## Empirical results

### Headline (Sequential pattern, r=3, vs strongest baselines)

| Method | MATH500 | AIME25 | AIME26 | GPQA-D | LiveCodeBench | MedQA | Avg |
|---|---:|---:|---:|---:|---:|---:|---:|
| Single LoRA | 83.1 | 70.0 | 73.3 | 62.0 | 37.4 | 76.1 | 66.9 |
| Single Full-SFT | 83.2 | 73.3 | 76.7 | 62.8 | 38.6 | 77.0 | 68.6 |
| MoA (Wang 2025b) | 79.8 | 60.0 | 63.3 | 47.6 | 27.0 | 57.5 | 55.8 |
| TextGrad | 84.9 | 73.3 | 76.7 | 62.5 | 39.8 | 77.2 | 69.1 |
| LoopLM (Ouro-2.6B) | 84.6 | 66.7 | 63.3 | 48.1 | 24.9 | 56.4 | 57.3 |
| Recursive-TextMAS | 85.8 | 73.3 | 73.3 | 61.6 | 38.7 | 77.0 | 68.3 |
| **RecursiveMAS r=3** | **88.0** | **86.7** | **86.7** | **66.2** | **42.9** | **79.3** | **74.9** |

Aggregate gain: **+8.3%** vs Recursive-TextMAS at matched recursion depth; **+8.0%** vs full-SFT; **+19.1%** vs the strongest text-mediated MAS (MoA).

### Pareto curve across recursion depth

| r | Tokens vs text-MAS | Speedup | Avg accuracy |
|---:|---:|---:|---:|
| 1 | −34.6% | 1.2× | (baseline depth) |
| 2 | −65.5% | 1.9× | (intermediate) |
| 3 | **−75.6%** | **2.4×** | **74.9** |

This is the Pareto-improving curve: deeper recursion *reduces* total cost while *increasing* accuracy, because text-decoding cost dominates and is amortized away.

### Training-cost comparison

| Method | GPU mem | Cost per run |
|---|---:|---:|
| Full-SFT | 41.40 GB | $9.67 |
| LoRA | 21.67 GB | $6.64 |
| **RecursiveMAS** | **15.29 GB** | **$4.27** |

Trainable params: 13.12M (RecursiveMAS) vs 15.92M (LoRA, 0.37%) vs 4.21B (full SFT, ~100%). The RecursiveLink is *smaller than LoRA* and *320× smaller than full SFT*.

### Pattern-level highlights

- **Mixture** lifts +6.2% over the best individual specialist; the summarizer's job is purely aggregation, but in latent space.
- **Deliberation** lifts +4.8% over the tool-calling agent baseline; the Reflector's latent intent is more structured than text would be.
- **Distillation** lets a small Learner reach 83.3 on AIME26 (vs 90.0 Expert) at 1.5× speed.

## Variants and ablations

- **Recursion depth** r ∈ {1, 2, 3}: monotone Pareto improvement on both axes; ablation beyond r=3 not reported.
- **Latent-thought count** m: saturation around 80; Fig. 8 / Table 9 shows diminishing returns past 100.
- **Inner Link only vs Outer Link only**: Inner alone sustains intra-agent latent autoregression but cannot bridge dimensions; Outer alone destabilizes intra-agent generation. Both are required.
- **Frozen vs joint LLM training**: paper trains only the links; whole-system fine-tuning is left to future work.
- **Backbones**: math/code/science specialists tested (Llama, Qwen, DeepSeek family). Heterogeneity is a feature; same-family configurations also work but show smaller gains.
- **Patterns × backbones**: Sequential is the workhorse; Mixture and Deliberation each amplify on their domain (multi-task / tool-use).

## Failure modes and limitations

- **No dedicated Limitations section** in v1 — the paper jumps from results to a one-paragraph Conclusion. Implicit caveats below.
- **LLM weights are frozen.** The recursion-with-fine-tuning regime is untested; jointly tuning the underlying LLMs may break the RecursiveLink's residual assumption.
- **Theory assumes confident outputs.** Theorem 4.1's "next-token entropy ≤ ε" precondition holds for math/code/science tasks where the answer distribution is sharp; behavior on high-entropy tasks (open-ended generation, creative writing) is uncharacterized.
- **Recursion sweep stops at r=3.** Diminishing returns or instability beyond r=3 are not shown. Whether the curve continues, plateaus, or inverts is open.
- **Latent-thought budget m is hand-tuned.** No adaptive m mechanism; the saturation point may shift across task domains.
- **Plug-and-play of new agents at inference is not demonstrated.** The Outer Link's W₃ projection is jointly trained with the system; swapping in a new agent at test time would require retraining the link.
- **Inner-loop warm-start cost is not quantified separately.** Total training cost numbers ($4.27) appear to include the warm-start; isolating its share would help reproducibility.
- **Benchmarks are reasoning-heavy.** Nine benchmarks across math, code, sci/med, and search-QA — no open-ended, multimodal, or long-horizon agentic tasks (tool-using web agents, GUI tasks, CodeBench-Lite-style live tasks).
- **No comparison to other recursive single-agent baselines beyond LoopLM.** Geiping's recurrent-depth 3.5B and Mixture-of-Recursions are not in the table.

## When to use, when not

**Use** for heterogeneous reasoning systems where multiple frozen specialists must collaborate (math + code + science, planner + critic + solver, expert + learner, reflector + tool-user); when token cost is a binding constraint (production serving, on-device inference); when the underlying agents are confident on their domain (entropy-low outputs); when the topology is fixed at deploy time so the joint training of W₃ is amortized.

**Don't use** for systems that need plug-and-play swapping of agents at inference time; for high-entropy generative tasks (open-ended writing, creative planning) where Theorem 4.1's preconditions don't hold; for single-agent applications where you'd be better off with single-network recursion (LoopLM/Ouro, Geiping recurrent-depth); for very long horizons or tool-using agents where the latent-only channel doesn't carry enough discrete state (the Deliberation pattern's tool-call decode is the only escape hatch demonstrated); for production systems that don't tolerate the additional 13M-param translator layer in the dependency graph.

## Implications for harness engineering

- **Inter-agent text becomes a tax on production MAS.** [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md) and [98-diversity-collapse-mas](98-diversity-collapse-mas.md) identified that text-mediated MAS doesn't add as much value as the upvote-driven discourse claimed. RecursiveMAS shows there's a *separate* cost dimension: even when MAS *does* add value, the way agents currently communicate (text) is paying a 25× vocab-projection tax. Production harnesses with stable topologies should plan for a latent-bridge layer.
- **A new tier in the harness stack: the recursive-link layer.** [02-subagent-delegation](02-subagent-delegation.md) describes orchestrator-worker; [01-agent-loop-architecture](01-agent-loop-architecture.md) describes the think/act/observe loop. RecursiveMAS introduces a sub-loop layer between agents that lives in the harness, owns continuous-state translation, and is jointly trained with the system. A future Claude Code-style harness would have *three* trainable substrates: skills/memory (per-agent), subagent topology (per-system), and inter-agent links (per-pair).
- **MAS-as-RLM unifies test-time scaling axes.** Until now, test-time scaling lived on three separate axes: longer CoT (per token), more samples (per generation), more recursion (per network). RecursiveMAS adds the fourth: deeper system-level recursion. For a fixed compute budget, the harness now picks across all four — and the curves cross differently for different tasks. Expect the next routing literature ([86-frugalgpt](86-frugalgpt.md), [87-routellm](87-routellm.md), [88-confidence-driven-router](88-confidence-driven-router.md)) to grow a "recursion-depth" axis.
- **Eywa's MCP federation gets a latent companion.** [103-eywa-heterogeneous-fm-collaboration](103-eywa-heterogeneous-fm-collaboration.md) made MCP the federation API for non-LLM foundation models (text in, structured out). RecursiveMAS makes residual MLPs the federation API for *LLM-to-LLM* federation (latent in, latent out). Both are 13–50M-parameter bridges that turn the harness into a federation host. A fully realized "agent-as-federation" harness would have both: MCP boundaries for cross-modality FMs, RecursiveLinks for cross-LLM specialists.
- **Heterogeneity that matters is *training-objective* heterogeneity, not vendor heterogeneity.** [98-diversity-collapse-mas](98-diversity-collapse-mas.md) showed mixing GPT-4 + Claude buys you almost nothing because both reason similarly. Eywa demonstrated that *modality* heterogeneity (LLM + Chronos + TabPFN) buys you a lot. RecursiveMAS demonstrates a third axis: *task-tuning* heterogeneity — Llama-tuned-on-math + Qwen-tuned-on-code + DeepSeek-tuned-on-science, bridged by latent links, beat any single tuned model and any text-MAS over the same set. The harness's "vendor router" concept needs a "tuning-objective router" addition.
- **Verifier loops can move into latent space.** [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md) describes planner/generator/evaluator harnesses that decode text on every step. The Sequential pattern (Planner → Critic → Solver) demonstrates the same loop running entirely in latent space — verifier and evaluator never decode. For coding harnesses where the verifier is a separate LLM (rather than the test runner), the latent-bridge cost is gone; only the final output decodes.
- **Skills-as-recursive-layers.** [04-skills](04-skills.md) frames skills as model-invocable capability packages. RecursiveMAS suggests a different framing: a skill *is* a frozen specialist agent, and the harness recursively chains skills with latent links. Skill discovery ([177-skills-discovery-curator](177-skills-discovery-curator-strongest-2026-techniques.md)) becomes a search not over text-prompted invocations but over latent-bridged compositions.
- **Distillation gets a system-level recipe.** [110-transformer-llm-architecture](110-transformer-llm-architecture-for-agent-builders.md) and [120-rag-to-finetuning-spectrum](120-rag-to-finetuning-spectrum.md) cover model-level distillation. The Distillation pattern in RecursiveMAS distills *a multi-step reasoning trajectory* into a small Learner that runs alone at inference — yielding 83.3 AIME26 from a learner that would baseline far below. This is system-level distillation: the Expert is gone at inference; only the Learner remains, but its weights have been shaped by latent-bridge gradients flowing back from the Expert during training.
- **Deliberation is the bridge to tool use.** The Deliberation pattern's Tool-Caller is the only place where decoding is forced (because tools require structured strings). The harness pattern is recursive in latent space *until* a tool boundary, then decodes once, runs the tool, re-encodes the tool result back into latent. Future MCP-equipped harnesses can adopt this directly: latent-bridge between LLMs, MCP-tool-call boundary as the only decode gate, latent-bridge again on the tool result.
- **The RLM analogy buys a fresh design vocabulary.** Recasting MAS as RLM lets harness designers borrow concepts from network architecture: skip connections (cross-round residuals), gating (per-round invocation policy), pruning (early-exit if confident at round r=1), normalization (per-link layer norm). [156-heavyskill-parallel-reasoning-deliberation](156-heavyskill-parallel-reasoning-deliberation.md) and [167-autoskill-experience-driven-lifelong-learning](167-autoskill-experience-driven-lifelong-learning.md) both have parallel structures that could be re-expressed in this vocabulary.

The one-line takeaway for harness designers: **a frozen heterogeneous MAS is a single recursive language model in disguise — and the 25× cost penalty for treating it otherwise is no longer defensible once the latent-bridge pattern has a published recipe.**

## References and cross-links

- Parent recursive-LM line: Geiping et al. 2025 (arXiv:2502.05171), Zhu et al. 2025 LoopLM/Ouro (arXiv:2510.25741), Bae et al. 2025 Mixture-of-Recursions (arXiv:2507.10524), Jolicoeur-Martineau 2025 tiny recursive nets (arXiv:2510.04871). Internal: [32-recurrent-depth-implicit-reasoning](32-recurrent-depth-implicit-reasoning.md).
- Latent-MAS contrast: Du et al. 2025 (arXiv:2511.09149), Cache-to-Cache (arXiv:2510.03215), KVcomm, Zou et al. latent collaboration.
- Text-MAS baselines: CAMEL (Li 2023), ChatDev (Qian 2024), MoA (Wang 2025b), TextGrad (Yuksekgonul 2025), MALT (Motwani 2024). Internal: [92-chatdev](92-chatdev.md), [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md).
- Diversity-collapse / heterogeneity: [98-diversity-collapse-mas](98-diversity-collapse-mas.md), [103-eywa-heterogeneous-fm-collaboration](103-eywa-heterogeneous-fm-collaboration.md).
- Test-time scaling: s1 (Muennighoff 2025), [156-heavyskill-parallel-reasoning-deliberation](156-heavyskill-parallel-reasoning-deliberation.md).
- Synthesis: [193-recursive-world-organizations-synthesis](193-recursive-world-organizations-synthesis.md).
