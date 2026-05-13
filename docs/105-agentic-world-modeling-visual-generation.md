# 105 — Visual Generation in the New Era: From Atomic Mapping to Agentic World Modeling

**Paper.** Keming Wu, Zuhao Yang, Kaichen Zhang, Shizun Wang, Haowei Zhu, Sicong Leng, Zhongyu Yang, Qijie Wang, Sudong Wang, Ziting Wang, Zili Wang, Hui Zhang, Haonan Wang, Hang Zhou, Yifan Pu, Xingxuan Li, Fangneng Zhan, Bo Li, Lidong Bing, Yuxin Song, Ziwei Liu, Wenhu Chen, Jingdong Wang, Xinchao Wang, Xiaojuan Qi, Shijian Lu, Bin Wang (27 authors) — *Visual Generation in the New Era: An Evolution from Atomic Mapping to Agentic World Modeling* — arXiv:2604.28185v1 — Tsinghua / NTU / HKU / NUS / Baidu / StepFun / MiroMind et al. — April 30, 2026 — 129 pages, 411 references (188 from 2025 alone). Project page: https://github.com/EvolvingLMMs-Lab/Evolving-Visual-Generation.

**One-line definition.** A 129-page capability-oriented taxonomy and survey that reframes visual generation as a five-level progression — **L1 Atomic → L2 Conditional → L3 In-Context → L4 Agentic → L5 World-Modeling** — analyzes the six technical drivers behind it (diffusion-to-flow-matching, unified understanding-and-generation, post-training with DPO/GRPO, reward modeling, synthetic data, improved visual representations), and stress-tests the frontier with case studies (jigsaw, fluid dynamics, multi-turn drift, physics) that expose how conventional FID/CLIP scores hide structural, temporal, and causal failures the new agentic-generation regime makes unavoidable.

## Why this paper matters

The visual-generation field has reached an evaluation cliff. Frontier systems like Nano Banana and GPT-Image-2 produce images so photorealistic that FID and CLIP-score have saturated and no longer discriminate progress. Yet on tasks that require *spatial reasoning* (jigsaw reconstruction), *physical consistency* (fluid dynamics under intervention), *identity preservation* (multi-turn editing), or *causal understanding* (counterfactual scene change), the same systems fail systematically. The mismatch between perceptual benchmarks and operational capability is no longer manageable: training that optimizes for FID literally pulls the field away from where the next capability gains live.

This paper is the first to give the community a vocabulary to talk about that gap. The five-level taxonomy is not a survey trick; it is a *capability ladder* with explicit operational distinctions:

- **L1 (Atomic):** one forward pass, no explicit constraint — text → plausible image.
- **L2 (Conditional):** one forward pass with one explicit condition — depth map / reference / layout → constrained image.
- **L3 (In-Context):** one forward pass over rich context — multi-reference, accumulated history → coherent output.
- **L4 (Agentic):** *multiple* forward passes orchestrated by external control — plan → render → verify → refine.
- **L5 (World-Modeling):** generation anchored by *internalized causal dynamics* — render scenarios that respect physical laws, intervention effects, and counterfactual structure.

The L3/L4 boundary (single vs multiple forward passes) and the L4/L5 boundary (correlation policy vs internalized causality) are the two design points that determine whether your system is a "smarter image generator" or a *world model* you can plan against.

Take the paper seriously and three things change. (1) **Visual generation enters the agent harness** — at L4 and above, generation is a *step inside the agent loop*, not a terminal output, and the harness must orchestrate plan → render → verify → refine over multiple passes. (2) **Evaluation methodology must be rebuilt** — the saturated metrics get replaced by stress-testing methodology mapped to taxonomy levels (Section 7's eight-dimensional case studies are a template, not just a results section). (3) **The closed-vs-open gap is now mostly a *system-level* gap** — the paper attributes Nano Banana / GPT-Image-2's lead to verifier-in-the-loop and dual-path encoders rather than larger diffusion backbones, which means open-source can close the gap by adopting the *system architecture*, not by training bigger models.

## Why a survey deserves a deep-dive

Most surveys are reading lists. This one is structurally a *position paper*: the taxonomy is a thesis (visual intelligence is a five-level progression, not a quality continuum), the technical-drivers chapter is an explanation of why the field is at L3/L4 and not yet L5, the stress-tests chapter is an indictment of current evaluation, and the closed-vs-open analysis (Section 3.3) is a reverse-engineering claim. Reading the paper transactionally for "what's the latest model" misses the point. Reading it for the *architecture of the field* — what changed, why, what's next — is what makes it worth 129 pages.

## The five-level taxonomy

| Level | Paradigm | Defining property | Key challenge | Representative methods |
|---|---|---|---|---|
| **L1 Atomic** | One pass, no explicit constraint | Stochastic plausibility (distribution matching) | Uncontrolled variation; spatial precision absent | DDPM, DiT, LDM/Stable Diffusion, LlamaGen, VAR |
| **L2 Conditional** | One pass + single explicit condition | Compositional controllability | Attribute binding & spatial precision | ControlNet, IP-Adapter, InstantID, GLIGEN, SD3 |
| **L3 In-Context** | One pass over rich context | Contextual coherence (visual logic + memory) | Identity preservation under cumulative state change | SEED-Data-Edit, ImgEdit, OmniGen, StoryMaker, Visual Persona, ReasonGen-R1 |
| **L4 Agentic** | Multi-pass with external control loop | Closed-loop agency (perception-action-feedback) | Grounded verification & self-correction in open-ended loops | GEMS, Gen-Searcher, JarvisArt, CoT-VLA, UniPi |
| **L5 World-Modeling** | Generation anchored by causal dynamics | Causal simulation (physics + intervention) | Causal faithfulness; correlation vs causation | Genie 2, GameNGen, Oasis, UniSim, GAIA-1 |

**L3 vs L4** — both involve rich context, but L3 is one forward pass; L4 is multiple passes orchestrated by an external controller making *dynamic* decisions. L3 is bounded by what the model can do in a single rollout; L4 is bounded by what the harness can verify and refine.

**L4 vs L5** — L4 agents reason from learned correlations in a policy loop; L5 internalizes world dynamics so generation itself encodes causal consequences of actions. L4 is "agent + generator with verifier"; L5 is "world simulator the agent plans against."

## The six technical drivers

The survey identifies six cross-cutting drivers advancing the trajectory. Each is interesting on its own; together they explain the L1→L4 journey and the obstacles to L5.

1. **Diffusion → Flow Matching.** Diffusion models replaced GANs' instability with stable iterative denoising, but required hundreds of sampling steps. Flow matching learns straight-line transport between noise and data, enabling 4–8 step sampling and dramatically improving deployment feasibility. Operationally, flow matching's ODE is more amenable to distillation, which makes L3-style multi-turn consistency under tight token budgets viable.

2. **Unified understanding-and-generation architectures.** Early models had separate vision encoders (understanding) and diffusion/AR backbones (generation). Recent systems (Chameleon, Emu3, BAGEL, BLIP3o-NEXT, Transfusion) route both modalities through shared backbones with unified tokenization. Generation gains access to reasoning-level representations; vision-language alignment tightens; understanding gains transfer into generation. Trade-off: shared parameters create a budget competition between the two.

3. **Improved visual representations.** Shift from VAE-only latents to multi-path strategies (VAE for fine-grained pixel fidelity + SigLIP/semantic encoders for concept grounding). Frozen pretrained encoders (DINOv2, SigLIP) replace trainable VAEs, accelerating training while improving semantic grounding. For AR models, discrete tokenization (VQ-VAE variants, residual VQ) enables a finite-MDP structure that RL can exploit.

4. **SFT + preference-based post-training.** SFT establishes baseline instruction-following from curated data. DPO and GRPO move beyond SFT's mode-seeking ceiling by optimizing pairwise or group-relative preferences. The deeper insight: preference optimization is *not* about external reward; it is about learning to prefer outputs that satisfy human judgement directly, with the discriminator in the loss rather than in a separate model.

5. **Reward modeling as supervisory interface.** Instead of hand-coded reward functions, frontier systems train discriminative reward models (HPSv3, MPS, VisionReward) or generative reward models (VLM-as-judge, OneReward) that predict holistic or decomposed quality. Credit assignment becomes explicit: dense rewards distributed along denoising trajectories outperform terminal-only rewards. Editing has its own specialized reward (EditReward) that scores instruction-following and preservation orthogonally.

6. **Large-scale synthetic data.** Shift from passive web scraping to active data engineering: foundation-model distillation (synthesizing training data from proprietary teachers), VLM-driven relabeling (replacing noisy alt-text with dense descriptions), video mining (before/after pairs from temporal coherence), 3D and code rendering (pixel-perfect ground truth for spatial/symbolic tasks), and aggressive multi-stage filtering (overgeneration with strict selection beats moderate generation with loose filtering). The motto changed from "billions of pairs" to "millions of curated pairs match millions of loosely-filtered ones."

These six interact: flow matching enables few-step sampling which makes L3 multi-pass affordable; unified U+G enables L4 closed-loop verification; preference post-training and reward modeling make L4 verifier-refiner loops actually trainable; synthetic data unlocks the rare task structures (spatial layout, structured content) that pretraining never saw.

## Architecture splits in the wild

Three contested architectural choices, all still in flux as of April 2026:

- **Diffusion-native** (Z-Image, Qwen-Image, Seedream, LongCat-Image) — stable, mature distillation toolchain, high edit fidelity.
- **AR-native** (LongCat-Next, Janus-Pro, HunyuanImage 3.0 with flow head) — simpler finite-MDP RL, harder to distill to few steps.
- **Hybrid** (Wan-Image, BLIP3o-NEXT) — AR semantic planner + diffusion/flow renderer; balances reasoning and fidelity, dominant on the L4 frontier.

Industrial convergence (frontier tech reports, 2025–2026) is striking: MM-DiT or hybrid is dominant, U-Net is retired, the four-stage skeleton **PT → CT → SFT → RL** is universal (8 of 10 reports include explicit continued-training), distillation is mandatory (6 of 10 ship explicit distillation, 8-step generation is the default deployment target), and aggressive overgeneration with strict filtering (5–45% retention) beats moderate generation with loose filtering.

## Empirical highlights

The survey is not primarily empirical, but the meta-findings on benchmarks make the case for the taxonomy more forcefully than any individual model could:

### Closed vs open gap

| Benchmark | Closed leader | Open leader | Gap |
|---|---|---|---|
| PRISM-Bench | GPT-Image-1: 86.3 | Qwen-Image: 79.9 | 6.4 pp |
| ImagenWorld | GPT-Image-1: 0.91 | LongCat / Seedream ~0.78–0.82 | 0.09–0.13 |
| Text rendering (word accuracy) | GPT-4o: 60.69–82.88% | Open-source: 0.11–2.98% | 30–60 pp |
| ChineseWord | LongCat-Image: 90.7 | Seedream 4.0: 58.5 | 32 pp |
| Long-text (60+ char) | Closed: stable | Open + AR: degrades sharply | — |
| GenExam (strict eval) | Best closed: 72.7% | Open-source: <5% | 60+ pp |

### Universal failure modes

- **PhyBench** — all models fail across mechanics, optics, thermodynamics, materials.
- **WISE (world knowledge)** — best open-source diffusion (LongCat) at 0.65; chemistry weakest universally.
- **R2I-Bench (causal reasoning)** — open-source <5% on causal tracks.
- **OmniGen on 10-type reference combinations** — even best unified model at 66.6% accuracy on 10-type spatial composition.

### Why closed pulls ahead (Section 3.3, speculative but evidence-grounded)

1. **Text-encoding path** — upstream frontier-grade VLM (Gemini-class) does planning and rephrasing before rendering.
2. **Visual-encoding path** — dual-path encoder (VAE for fidelity + semantic encoder for intent) vs single encoder in open systems.
3. **Generation-time understanding** — understanding is a persistent runtime constraint, not just upstream preprocessing.
4. **System-level agentic loop** — planner → verifier → refiner with silent self-correction vs single forward pass.

## The eight-dimensional stress test (Section 7)

The most operationally useful chapter. Rather than averaging scores across benchmarks, the authors design case studies that map directly onto taxonomy levels and expose specific failure modes:

- **Spatial structuring & layout precision** — jigsaw reconstruction, metro maps, isometric tile maps. Models default to *semantic hallucination* rather than geometric reasoning.
- **Physical reasoning & causal fidelity** — fluid dynamics counterfactuals, action-conditioned navigation, robotic manipulation, spatiotemporal trajectory, video re-rendering, material consistency. Universal failures expose the L4-vs-L5 gap.
- **Visual-textual integration** — physics exam solving, structured-content rendering. Fragile.
- **Multi-turn editing — silent drift & Markovian chaining** — open-source steeper drift after 3+ rounds; closed-source flatter degradation; restore-to-original fails under cascading drift.
- **Human-centric heredity** — appearance prediction; identity preservation vs instruction-fidelity trade-off unresolved.
- **Low-level vision** — depth, restoration; baseline gains, but not transformative.
- **Real-world applications** — urban planning, UI design, coding, diagrams. Where L4 agentic generation pays off most.
- **High-level vision** — OCR, keypoint, segmentation, detection. Generation as a perception substrate.

The methodological lesson: benchmark averages hide catastrophic failures on specific capability *combinations*. A model with 85% on six independent dimensions can fail when all six must be satisfied at once — exactly the regime real-world tasks operate in.

## Variants and ablations

The survey doesn't run ablations; it organizes the field's variants along clear axes:

- **RL algorithm fragmentation** — at least seven GRPO variants emerge across credit-assignment, group-synchronization, and pairing-free axes (MixGRPO/SRPO, MPO, DenseGRPO, DiffusionNFT, Flow-GRPO, ReFMA). Different MoE backbones favor different variants; "use GRPO" is no longer a sufficient prescription.
- **Data-construction strategies** — frontier distillation (ShareGPT-4o-Image 91K, Pico-Banana 400K) vs open-source pipelines (FLUX + ControlNet + segmentation) vs temporal extraction (ByteMorph-6M) vs programmatic/3D rendering. Each makes different quality/diversity trade-offs.
- **Conditioning mechanisms** — feature-wise addition (ControlNet) → cross-attention (IP-Adapter) → in-context self-attention (OmniGen, Qwen-Image) → 3D-RoPE temporal separation (Z-Image-Edit, Wan-Image). Each is more flexible than the last but more expensive.

## Failure modes and limitations

- **Evaluation blindness.** FID/CLIP saturated; pairwise human evaluation doesn't scale; VLM judges show 0.57–0.75 Spearman with humans and systematic biases (preferring AI outputs over human edits at 0.14–0.25 Cohen's κ); bilingual evaluation gaps that English-only benchmarks miss; structured-content evaluation absent.
- **Spatial reasoning** — counting and precise spatial positioning remain weakest dimensions for SOTA.
- **Physical knowledge** — universal failure on PhyBench across mechanics/optics/thermodynamics/materials.
- **Causal reasoning** — open-source <5% on GenExam and R2I-Bench causal tracks.
- **Identity preservation** — instruction-fidelity vs identity-preservation trade-off unresolved; no model satisfactorily balances both.
- **Long-horizon context** — degradation on 30+ word or 60+ character text; multi-turn consistency degrades exponentially.

## When to use, when not (mapped to taxonomy levels)

**L1 (Atomic).** Use for creative content, brainstorming, entertainment where plausibility > correctness.
**L2 (Conditional).** Use for layout-guided design, style transfer, identity-consistent generation.
**L3 (In-Context).** Use for short multi-turn editing (2–3 turns acceptable), multi-reference composition under single-pass budgets.
**L4 (Agentic).** Use for tasks where verification can drive refinement (UI design with constraints, structured documents, photo retouching with explicit goals).
**L5 (World-Modeling).** Experimental — Genie 2, GAIA-1 are research prototypes, not production.

**Don't use** for spatial-or-physical-verification tasks (jigsaw, SAT-style constraints) — current models hallucinate plausibly rather than verify; long-horizon multi-turn consistency (5+ rounds) without external memory; world understanding for safety-critical embodied control; non-Latin script bilingual rendering unless using a frontier model optimized for the language pair; structural-correctness guarantees (chemistry, circuits, code-generated visuals) — generation must be paired with symbolic verification.

## Implications for harness engineering

- **Generation enters the agent loop at L4.** Past harnesses treated visual generation as a terminal output; at L4 it becomes a step *inside* a plan → render → verify → refine cycle. The harness owns the orchestration (cf. [01-agent-loop-architecture](01-agent-loop-architecture.md)) and the verifier (cf. [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md)). The closed-vs-open gap analysis explicitly attributes the lead to verifier-in-the-loop discipline — meaning [18-chain-of-verification-self-refine](18-chain-of-verification-self-refine.md) is the right pattern, not bigger models.
- **L5 world models become differentiable environment models for embodied agents.** Visual generation that internalizes world dynamics serves as a planning substrate (the agent simulates futures before acting). Current systems fail at causal faithfulness, but the architectural path (Genie 2, GAIA-1) is clear. Pairs naturally with [85-alphaevolve](85-alphaevolve.md)'s evolutionary planning over verifiable generation.
- **Reward modeling as supervisory interface.** [97-qwen-prm](97-qwen-prm.md) showed that LLM-as-judge step labels actively harm process-reward models for math; the visual generation literature reaches a similar conclusion (VLM judges are systematically biased; consensus filtering matters). The right reward stack is *not* "ask a strong VLM to score outputs" — it is a discriminative reward model trained on human-anchored preference data, with judge ensembles for bootstrap. Harness teams should adopt this as a default, not the obvious alternative.
- **Tool-augmented generation as L4 instantiation.** JarvisArt's tool-augmented photo retouching with executable action traces is the L4 pattern par excellence. The same template applies to other generative-as-tool harness designs: planning via reasoning tokens, retrieving reference images via search ([25-agentic-rag](25-agentic-rag.md)), rendering via diffusion, verification via reward models — with the harness, not the model, owning the loop.
- **Reflexion- and ToT-style refinement transfer.** [14-reflexion](14-reflexion.md) and [15-tree-of-thoughts-lats](15-tree-of-thoughts-lats.md) describe the verbal-reflection and search-tree patterns for reasoning agents. They transfer near-directly to L4 visual generation: reflect on a rendering's failure mode, search over candidate refinements, prune by verifier score. The L4 loop is, in effect, ToT over generations.
- **Stress-test methodology over benchmarks.** The Section 7 case-study methodology is a transferable evaluation pattern. Harness eval suites should include domain-specific stress tests probing failure modes mapped to capability levels — not just bulk averages. The same shape of methodology fits agentic-coding, agentic-research, and computer-use harnesses (this is also the lesson of [101-autoresearchbench](101-autoresearchbench.md)).
- **Distillation is mandatory in the harness recipe.** Six of ten frontier reports ship explicit distillation; 8-step generation is the default deployment target. Production harnesses should assume base models require few-step student models, and architecture/training choices must be compatible with downstream distillation (flow matching preferred, discrete AR less friendly).
- **Synthetic data as harness tool.** Section 5.1's data-construction pipeline (source → instruction → generation/editing → quality control → assembly) is reusable far beyond visual generation. Harnesses can use the model in a *data-construction loop* — generate, filter, verify, distill back into training — orchestrated by the harness itself. This is the visual-generation analog of ClawGym's synthesis recipe (file 102) and the broader self-evolving training arenas in [69-agent-world](69-agent-world-self-evolving-training-arena.md).
- **Closed-vs-open is now mostly system-level.** The most important practical claim in the paper: the closed-source lead is largely a system-level phenomenon (verifier loops, dual-path encoders, upstream VLM planning) rather than a backbone-scale phenomenon. Open-source can close the gap by adopting the *system architecture*, not by training larger backbones. This reframes the open-source roadmap.

The one-line takeaway for harness designers: **at L4 and above, generation is no longer a model output — it's a step in your agent loop, and the verifier you put around it determines your capability frontier more than the model you put in.**
