# 190 — Agentic World Modeling: Foundations, Capabilities, Laws, and Beyond

**Paper.** Meng Chu, Xuan Billy Zhang, Kevin Qinghong Lin, Lingdong Kong, Jize Zhang, Teng Tu, Weijian Ma, Ziqi Huang, Senqiao Yang, Wei Huang, Yeying Jin, Zhefan Rao, Jinhui Ye, Xinyu Lin, Xichen Zhang, Qisheng Hu, Shuai Yang, Leyang Shen, Wei Chow, Yifei Dong, Fengyi Wu, Quanyu Long, Bin Xia, Shaozuo Yu, Mingkang Zhu, Wenhu Zhang, Jiehui Huang, Haokun Gui, Haoxuan Che, Long Chen, Qifeng Chen, Wenxuan Zhang, Wenya Wang, Xiaojuan Qi, Yang Deng, Yanwei Li, Mike Zheng Shou, Zhi-Qi Cheng, See-Kiong Ng, Ziwei Liu, Philip Torr, Jiaya Jia (~42 authors) — *Agentic World Modeling: Foundations, Capabilities, Laws, and Beyond* — arXiv:2604.22748v1 — HKUST / NUS / Oxford / NTU / CUHK / HKU / SUTD / SMU / UW — April 24, 2026. Project page: https://agentic-world-modeling.xyz/. Awesome list: https://github.com/matrix-agent/awesome-agentic-world-modeling. HuggingFace daily papers: 219 upvotes (week of 2026-W18).

> **Disambiguation.** This paper (2604.22748) is *not* the same as [105-agentic-world-modeling-visual-generation](105-agentic-world-modeling-visual-generation.md) (arXiv:2604.28185, Wu et al., visual-generation focus). They share co-author Ziwei Liu and the phrase "agentic world modeling," but the visual-generation paper is a survey of generative-image systems with a five-level "Atomic → Conditional → In-Context → Agentic → World-Modeling Generation" capability ladder; *this* paper is a foundational taxonomy of world modeling for AI agents in general — across robotics, web agents, social simulation, and scientific discovery — organized as 3 capability levels (L1/L2/L3) × 4 governing-law regimes (Physical / Digital / Social / Scientific). Read both: 2604.28185 is the visual-pipeline view (where world-modeling-generation is the *terminus*); 2604.22748 is the AI-agent view (where world modeling is the *substrate*). The two converge at the L4/L5 vs L2/L3 boundary — the same operational distinction (correlation policy vs internalized causal dynamics) appears in both, framed for different audiences.

**One-line definition.** A 400-paper taxonomy that re-grounds "world models" for the agent era: world modeling is a **three-level capability progression** (L1 Predictor — local one-step transition operators; L2 Simulator — multi-step rollouts respecting governing laws; L3 Evolver — autonomous evidence-driven revision of the model itself) **× four governing-law regimes** (Physical, Digital, Social, Scientific), with explicit boundary conditions, decision-centric evaluation (rejecting Fréchet Video Distance for Action Success Rate and Counterfactual Outcome Deviation), and a Lakatosian + Duhem-Quine philosophical foundation that argues genuine L3 evolution requires a **symbolic substrate**, not just deeper latent dynamics.

## Why this paper matters

The world-models literature has fragmented along three independent axes: the RL-and-robotics line (PILCO → World Models 2018 → DreamerV3 → MuZero → DIAMOND), the generative-video line (Sora, Genie 2, Oasis, GameNGen, UniSim), and the scientific-surrogate line (GraphCast, AlphaFold, neural operators). Each line has its own evaluation conventions — PSNR/FVD for video, return for RL, RMSE on held-out trajectories for surrogates — and the result is that "world model" has become a marketing term for any system that outputs a future state. A 4K video of a stable scene rolls out for ten seconds and gets called a world model; so does a 1.5B-parameter latent-dynamics RNN; so does a 200B-parameter Sora variant. None of these are the same kind of object, and none can be substituted for one another in an agent's planning loop without a structural mismatch.

This paper is the first attempt to put a *structural* taxonomy under the discourse — not a survey ranking but an operational ladder with explicit boundary conditions. The L1/L2/L3 split is the load-bearing claim: a Predictor (L1) that does one-step transitions is qualitatively different from a Simulator (L2) that composes those transitions into action-conditioned rollouts which respect domain laws under intervention, which is qualitatively different from an Evolver (L3) that autonomously revises its own model when predictions fail against new evidence. The orthogonal Physical / Digital / Social / Scientific regime axis matters because the *verification* shape changes: physical laws are simulator-specifiable, digital laws are deterministically verifiable, social laws are reflexive (beliefs change state) and normative, and scientific laws are latent causal mechanisms that must be discovered empirically. A world model good at one regime is not automatically good at another, and the failure modes are regime-specific.

Two consequences land hardest. First, the **decision-centric evaluation argument** (Section 6, App. E.6): the paper rejects FVD ("masks catastrophic planning failures") and proposes Action Success Rate plus **Counterfactual Outcome Deviation** — the task-relevant trajectory distance between two rollouts that diverge at a single intervention step k. If COD ≈ 0, the model is action-insensitive and useless for planning, regardless of pixel fidelity. This single metric is a sharp test that disqualifies most current "world-model" video generators including, on the paper's reading, Sora as it is. Second, the **L3 requires symbolic substrate** thesis: pure-latent systems can do L1 and (under composition discipline) L2, but genuine *revision of the model's own dynamics* against new evidence requires structures the system can name, point at, and selectively replace — propositions, parameters, modules — which the paper argues are best supported by symbolic or hybrid neuro-symbolic representations.

Take this paper seriously and three downstream things change. (1) **"World model" stops being a single category** — when an architect picks Sora vs DreamerV3 vs GraphCast vs GameNGen, they're picking systems at *different levels of the taxonomy* solving *different regime tasks*; the choice is no longer about scale or modality. (2) **Evaluation infrastructure must be rebuilt** — Action Success Rate and Counterfactual Outcome Deviation are not just additional metrics; they're the metrics that actually correlate with planning utility, and benchmark suites that ignore them are measuring the wrong thing. (3) **Agentic harnesses gain a "world-model-aware" routing decision** — for a given task, what level and regime does my world model need to be? A coding agent's world is L2 Digital; a robotics agent's world is L2 Physical with L3 aspirations; a discovery agent is L2 Scientific with mandatory L3.

## Problem it solves

- **"World model" is overloaded.** The literature uses the term for everything from a 5-layer MLP latent-dynamics module (DreamerV1) to a 200B-parameter video generator (Sora). The boundary conditions for what counts as a world model are not specified; the paper makes them explicit (rollout, intervention sensitivity, constraint consistency).
- **Cross-regime transfer is unmeasured.** A model good at Physical (DreamerV3 on robotics) is silently assumed to be a candidate for Digital (web agents) or Scientific (drug discovery). The paper formalizes that the four regimes have different verification shapes and requires regime-specific evaluation.
- **Visual fidelity ≠ decision utility.** The video-generation community optimized FVD/PSNR/CLIP-score for years; those metrics saturated and stopped tracking planning utility. The paper's MREP (Minimal Reproducible Evaluation Package) replaces them with action-conditioned metrics.
- **The Sora-as-world-simulator claim is unfalsifiable as currently posed.** OpenAI's "Sora is a world simulator" framing does not specify intervention sensitivity or constraint consistency. The paper provides the falsification protocol: COD on counterfactual prompts.
- **Self-improving world models lack a governance shape.** Continuous self-improvement risks benchmark overfitting and adversarial-evidence overwriting (where a few noisy traces silently rewrite verified domain laws). The paper's L3 boundary conditions (evidence-grounded diagnosis, persistent asset update, governed validation) define the governance.

## Core idea in one paragraph

A world model is not a single system but a position in a **3 × 4 grid**: capability level (L1 Predictor, L2 Simulator, L3 Evolver) × governing-law regime (Physical, Digital, Social, Scientific). The capability levels are a containment hierarchy — L2 invokes L1 at each step, L3 invokes L2 before each update — with explicit boundary conditions: L1→L2 requires long-horizon coherence, intervention sensitivity, and constraint consistency; L2→L3 requires evidence-grounded diagnosis, persistent asset update, and governed validation. The regime axis defines verification shape: Physical regimes admit ground-truth via physics engines; Digital regimes admit deterministic specification; Social regimes are reflexive (beliefs change state) and normative (governed by what should happen); Scientific regimes have latent causal mechanisms that must be discovered empirically. The paper grounds the taxonomy philosophically — Hume's "constant conjunction" for L1, Lewis's "closest possible worlds" for L2, Lakatos's "hard core / protective belt" plus Duhem-Quine holism for L3 — and operationally, with a Minimal Reproducible Evaluation Package built on Action Success Rate and Counterfactual Outcome Deviation that explicitly rejects Fréchet Video Distance as a planning-utility metric. The thesis is normative as much as descriptive: the field should treat world models as **decision-usable simulators that continuously evolve through formalized, evidence-driven hypothesis testing**, not as parameter-scale frontiers chasing prettier pixels.

## The 3 × 4 taxonomy

### Capability axis

| Level | Name | Defining property | Boundary conditions to enter |
|---|---|---|---|
| **L1** | **Predictor** | One-step local transition operators | (entry condition) |
| **L2** | **Simulator** | Multi-step action-conditioned rollouts respecting domain laws | Long-horizon coherence; intervention sensitivity; constraint consistency |
| **L3** | **Evolver** | Autonomous evidence-driven revision of the model itself | Evidence-grounded diagnosis; persistent asset update; governed validation |

L1 factorizes into four local operators:
- **State inference** q_φ(z_t | o_{≤t}, a_{≤t-1}) — observe → infer latent state.
- **Forward dynamics** p_θ(z_t | z_{t-1}, a_t) — latent transition under action.
- **Observation decoding** p_ψ(o_t | z_t) — render the latent.
- **Inverse dynamics** π_η(a_t | z_{t-1}, z_t) — what action explains a state change.

L2 is the trajectory-level query p̂(τ | z₀, a_{1:H}, c) for horizon H and context c. The three boundary conditions are *sharp tests*: a model passes only when all three hold simultaneously. Long-horizon coherence rules out compounding error; intervention sensitivity rules out action-insensitive rollouts (the Counterfactual Outcome Deviation test); constraint consistency rules out hallucinated futures that violate governing laws.

L3 maintains an explicit update loop (M_t, d_t) → diagnose + distill + validate → M_{t+1} where d_t is fresh evidence. The three boundary conditions are governance: diagnosis must be grounded in measurable evidence (not reasoning hallucinated from prior context); updates must be persistent (not ephemeral patches that reset on next deployment); validation must be governed (regression and robustness gates that prevent silent overwriting of verified laws).

### Regime axis

| Regime | Verification shape | Anchor systems | Failure shape |
|---|---|---|---|
| **Physical** | Physics engines as ground truth | DreamerV3, RoboCasa, MuZero, DIAMOND, TD-MPC2 | Sim-to-real gap; off-manifold dynamics |
| **Digital** | Deterministic program semantics, API contracts, UI state machines | OSWorld agents, SWE-bench, GUI agents | API drift; UI state-machine violations |
| **Social** | Reflexive (beliefs change state); normative (what should happen) | Sotopia, sandbox simulations, digital twin societies | Reflexivity collapse; norm violations |
| **Scientific** | Latent causal mechanisms; experimental measurement | GraphCast, neural operator learning, AlphaFold-as-surrogate | Causal misidentification; out-of-distribution validity |

The regime axis matters because the *cost of wrong* differs. A Physical L2 simulator that violates Newtonian momentum is wrong but fixable. A Social L2 simulator that violates reflexivity (beliefs that don't update under new information) is structurally wrong — its outputs misrepresent the system being simulated. A Scientific L2 surrogate that fits training distribution but violates a latent causal mechanism may pass all conventional tests until deployment.

## The four pillars (paper structure)

- **Foundations** (§2). The POMDP tuple ℰ = (𝒳, 𝒜, Ω, T, O, R, γ); latent-dynamics factorization; counterfactual semantics on intervention; Markov factorization. Philosophical anchors: Hume → L1, Lewis → L2, Lakatos + Duhem-Quine → L3. Historical four eras: (1) Mathematical Principles (–1956), (2) Symbolic Intelligence (1956–1986), (3) Connectionist Resurgence (1986–2020), (4) Generative Revolution (2020–present). Representation theory: latent (VAE/RSSM/JEPA) vs symbolic; the survey's claim that "the endpoint of L3, namely genuine revision of governing laws, requires symbolic substrate."
- **Capabilities** (§3–5). The L1/L2/L3 ladder with method surveys per level. L1: representation learning, model-based RL, token & diffusion-based predictors. L2: per-regime instantiations, cross-domain composition, failure modes. L3: evolution policy, per-domain examples (AlphaEvolve in materials, robotic grasp learning, installation-failure recovery loops), evidence quality, governance / maturity.
- **Laws** (interleaved through L2/L3). The four governing-law regimes plus a set of empirical regularities the survey claims (capacity scales with horizon; epistemic drift risk; constraint enforcement cost; evidence requirement at L3; regime-coupling cost; latency tiering). These are not parametric scaling laws (no "loss = a · N^{-α}" curves); they are operational regularities about *how systems break*.
- **Beyond** (§8). Open problems per level; cross-regime hybrids (autonomous driving = physical + social); meta-world modeling (where governing laws themselves become learnable); governance risks (continuous self-improvement leading to benchmark overfitting and adversarial-evidence overwriting).

## Capabilities catalogue

A world model worth the name supports four progressively stronger capabilities:

1. **Rollout** — multi-step trajectory generation under an action sequence.
2. **Intervention sensitivity** — counterfactual edits to the action sequence induce stable, predictable changes in the trajectory.
3. **Constraint consistency** — generated futures respect governing-law invariants of the regime.
4. **Closed-loop use** — the trajectory is decision-usable (a downstream planner / controller can act on it).

Plus L3-only:

5. **Evidence-grounded diagnosis** — when a prediction fails against new evidence, the system localizes the failure to a specific operator or parameter.
6. **Persistent asset update** — the diagnosed correction persists across sessions / deployments.
7. **Governed validation** — the corrected model passes regression and robustness tests before becoming canonical.
8. **Active probe / experiment design** — the system designs its own data-collection policy to disambiguate competing hypotheses.

These eight are the operational test set. A "world model" claim should specify which capabilities it has and which it lacks.

## "Laws" — empirical regularities

The paper's *Laws* chapter does **not** present quantitative parametric scaling curves. It identifies six operational regularities about how world models behave at scale:

1. **Capacity scales with horizon.** Longer rollout horizon H requires deeper architectures or more parameters, sub-linear in H.
2. **Epistemic drift risk.** L2 systems can produce internally coherent but constraint-violating trajectories — the model "feels right" but is silently wrong on a governing law.
3. **Constraint enforcement cost.** Maintaining governing-law invariants (energy conservation, API correctness, social norms) trades off against action sensitivity; over-constraining produces stale, unresponsive simulations.
4. **Evidence requirement at L3.** Revisions must be grounded in measurable held-out probes; without that gate, L3 collapses into stylistic drift.
5. **Regime coupling cost.** Hybrid regimes (autonomous driving = physical + social; embodied scientific lab = physical + scientific) require *aligned* constraint satisfaction; the constraints from each regime must be consistent or the model has no feasible region.
6. **Latency tiering.** Real-time robotics demands < 100 ms inference (favoring lightweight latent dynamics like RSSM); offline scientific planning tolerates minutes per step (favoring Bayesian surrogates with explicit uncertainty).

These are testable claims, not curves; the paper offers them as design principles for builders.

## Decision-centric evaluation (MREP)

The Minimal Reproducible Evaluation Package is the paper's most operationally useful contribution. The diagnosis: visual-fidelity metrics (FVD, PSNR, CLIP-score) saturate well before planning utility does, and "masks catastrophic planning failures" because a fully action-insensitive video — same future regardless of action — can score perfect on visual metrics. The replacement:

- **Action Success Rate (ASR).** For a task t and an agent π that plans against the world model, the fraction of episodes in which π achieves the task goal. ASR depends on π's planning competence as well as the world model's quality, which is a feature not a bug — it measures *decision-usability*.
- **Counterfactual Outcome Deviation (COD).** Given two action sequences a₁ and a₂ that diverge at step k (a₁_{<k} = a₂_{<k}, a₁_k ≠ a₂_k, both well-formed), COD is the task-relevant distance between the resulting trajectories τ₁ and τ₂. If COD ≈ 0 across many counterfactual pairs, the model is *action-insensitive* — it generates plausible-looking futures regardless of what the agent does, and is therefore useless for planning regardless of FVD.
- **Reject FVD as primary.** FVD is retained only as a sanity check on output well-formedness, not as a quality signal.

The COD test is sharp: it will *disqualify* most current pixel-fidelity-optimized video generators when applied directly. The paper uses this as the basis for its critique that "high-fidelity video generation has been conflated with actionable simulation" — a line aimed at the Sora-as-world-simulator framing.

## Implementation efficiency notes

§7.3 documents the operational toolbox L2 builders draw from at scale:

- **Few-step distillation** to bring L2 rollout cost down (consistency models, progressive distillation).
- **Quantization and pruning** for deployment.
- **KV-cache compression** including 2-bit KV-cache quantization and token eviction — the L2 rollout's memory bottleneck is cache-resident, not compute-resident.

These are tactics, not the thesis; the thesis is that without the COD test passing, no efficiency win matters because the model isn't planning-usable.

## Distinctions from related concepts

- **Ha & Schmidhuber 2018 ("World Models")** — the canonical L1 (VAE encoder + RNN dynamics + linear controller). The paper extends to L2/L3 not addressed in the original.
- **Dreamer line (V1/V2/V3, RSSM, symlog)** — flagship L1/L2 system. Achieves multi-step rollout but the survey notes failure outside training manifold; positioned as exemplar L2 with documented intervention-sensitivity gaps.
- **JEPA / V-JEPA / I-JEPA** — L1 representation method without pixel reconstruction; emphasizes latent coherence; compatible with L2 elevation.
- **Sora / Genie 2 / Oasis / GameNGen** — L2-adjacent generative video. Achieve visual rollouts but unclear on intervention sensitivity and constraint consistency under the COD test. The survey's explicit critique: pixel coherence is not decision-usability.
- **UniSim** — unified-simulator L2 candidate; survey questions cross-domain constraint consistency.
- **Pearl's causality / LeCun's autonomous machines** — adjacent frameworks for what "real" world modeling demands. The paper's contribution beyond these is to formalize *autonomous structural hypothesis-testing* as an explicit boundary condition (L3), not just a desideratum.
- **Visual Generation paper ([105](105-agentic-world-modeling-visual-generation.md))** — Wu et al.'s 5-level visual ladder (Atomic → Conditional → In-Context → Agentic → World-Modeling Generation) maps onto this paper's axes as: Atomic = L1 + Physical regime image proxy; Conditional/In-Context = enhanced L1 with prompt grounding; Agentic = L1+L2 boundary with external verifier; World-Modeling Generation = L2/L3 in the Physical regime. Reading both gives a stereo view: 105 is the visual-pipeline staircase, 190 is the cross-regime taxonomy.

## Anchor systems by regime

- **L1 across regimes (~70+ exemplars):** PILCO, E2C, PETS, Ha & Schmidhuber 2018, DreamerV1/V2/V3, MuZero, EfficientZero, TD-MPC2, DeepMDP, MBPO, IRIS, TransDreamer, STORM, DIAMOND, Delta-IRIS.
- **Physical L2:** RoboCasa, sim-to-real pipelines, Brooks et al. 2024 video prediction.
- **Digital L2:** OSWorld, SWE-bench, GUI agents, ClawBench / ClawEval ([34](34-clawbench-live-web-tasks.md), [38](38-claw-eval.md)).
- **Social L2:** Sotopia, sandbox simulations, digital-twin societies (Generative Agents lineage).
- **Scientific L2:** GraphCast, neural operator learning, molecular surrogates, AlphaFold-as-surrogate.
- **L3 examples:** AlphaEvolve (materials), robotic grasp self-improvement, installation-failure recovery loops.
- **Benchmarks per regime:** Physical (RoboCasa, Atari, continuous control), Digital (OSWorld, SWE-bench, ClawBench), Social (Sotopia), Scientific (ScienceWorld, DiscoveryBench).

## Open problems ("Beyond")

- **L1**: scaling to high-resolution, high-action-dimension environments; preserving Markov factorization under partial observability.
- **L2**: long-horizon coherence without compounding error; intervention sensitivity off-manifold; efficient constraint enforcement at scale.
- **L3**: automated *blame assignment* (Duhem-Quine holism — when a prediction fails, the failure is distributed across perception, dynamics, and policy; localizing it is non-trivial); active hypothesis generation; preventing overfitting to deployment quirks; rollback governance when an L3 update degrades verified capabilities.
- **Cross-regime**: hybrid regimes (autonomous driving, embodied scientific lab); regime transfer (an L2 trained in physical-regime video to a related social-regime task).
- **Meta-world modeling** (§8.3): systems where governing laws themselves are learnable — the four-regime axis becomes a search space rather than a fixed taxonomy.
- **Governance**: continuous self-improvement leading to benchmark overfitting and knowledge contamination; "adversarial or noisy evidence could silently overwrite critical, previously verified domain laws."

## Failure modes and limitations

- **Taxonomy without quantitative laws.** The paper's "Laws" are operational regularities, not parametric curves; readers expecting Chinchilla-style fits will be disappointed. This is a structural choice (the field doesn't have the data for parametric world-model scaling laws yet) but worth flagging.
- **Symbolic-substrate claim is normative.** The thesis that L3 requires symbolic substrate is argued philosophically (Lakatos hard-core / protective-belt structure) and operationally (revisability requires named structure), not empirically shown. A pure-latent system that *demonstrates* L3 boundary conditions would falsify the claim; the survey's position is that no such system exists.
- **Regime axis is paper-defined.** The four regimes are clean but not exhaustive — Embodied Multi-Modal, Quantum, and Hybrid Human-AI Collaborative are conceivable additions. The paper is normative about its taxonomy without engaging deeply with extensions.
- **MREP needs adoption.** Action Success Rate and Counterfactual Outcome Deviation are well-specified but require benchmark suites to instantiate them. Until major benchmarks ship MREP-compliant probes, the metrics remain proposals rather than measurements.
- **Sora critique is implicit.** The paper does not directly evaluate Sora / Genie 2 under MREP; the critique is structural ("pixel coherence is not decision-usability") rather than empirical. A direct head-to-head COD measurement would strengthen the case.
- **L3 examples are sparse.** AlphaEvolve, robotic grasp self-improvement, and installation-failure recovery are the showcase L3 examples; none are fully autonomous, evidence-driven, governed-validation systems. The L3 category is partly aspirational.

## When to use, when not

**Use** when designing a world-model system from scratch and needing a coherent specification of what level + regime you're targeting; when evaluating a claimed world model and needing a specification-grounded checklist (the eight capabilities, the COD test); when arbitrating between competing systems that occupy different positions in the 3×4 grid; when planning a research roadmap that needs to specify which boundary conditions to attack; as the structural reference for a multi-regime production agent (e.g., autonomous driving = Physical L2 + Social L2 hybrid).

**Don't use** as a runnable framework — this is a taxonomy paper, not a system implementation; for parametric scaling-law guidance (the paper explicitly does not provide them); as a substitute for benchmark numbers — the MREP is a proposal, not a leaderboard; for visual-generation-only contexts where [105](105-agentic-world-modeling-visual-generation.md)'s 5-level ladder is more operational; as an authority on Sora-class systems' actual COD behavior (which the paper claims structurally but does not measure).

## Implications for harness engineering

- **World-model selection becomes a routing decision.** A coding agent's task is L2 Digital; an embodied agent's task is L2 Physical (with L3 aspirations as it accumulates tool use). The harness should *know* which regime it's operating in and route to a regime-appropriate world model rather than treating "world model" as a uniform substrate. The routing literature ([86-frugalgpt](86-frugalgpt.md), [87-routellm](87-routellm.md), [88-confidence-driven-router](88-confidence-driven-router.md)) gains a "regime-router" axis.
- **The MREP test belongs in the eval suite.** [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md) covers trajectory evaluation for LLM agents. Action Success Rate and Counterfactual Outcome Deviation are the world-model-specific analogs. Any harness that uses a world model for planning should include COD probes as gating tests; without them the world model could be silently action-insensitive.
- **The L1/L2/L3 split clarifies the verifier-loop literature.** [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md) describes planner/generator/evaluator harnesses. The paper's L2 boundary (rollout + intervention + constraint) is precisely what a planner needs from its world model; without all three, the verifier becomes a self-fulfilling oracle.
- **Symbolic substrate as a harness primitive.** [37-neuro-symbolic-ai](37-neuro-symbolic-ai.md) and [128-knowledge-graphs-as-substrate](128-knowledge-graphs-as-substrate.md) are positioned in this paper's L3 column — the symbolic structure isn't decoration, it's the substrate that makes evidence-driven revision possible. Future harnesses serious about long-horizon self-improvement should plan a symbolic layer (knowledge graph, propositional store, modular skill library) that L3-style update loops can edit.
- **Skills are L3-shaped.** [04-skills](04-skills.md), [19-voyager-skill-libraries](19-voyager-skill-libraries.md), [177-skills-discovery-curator](177-skills-discovery-curator-strongest-2026-techniques.md): the skill library is precisely the "persistent asset" of the L3 boundary condition. Skill discovery + skill curation = the diagnose + distill + validate loop. The paper's framework retroactively gives skill-system design a principled vocabulary.
- **Governance shapes for self-improving harnesses.** Continuous self-improvement in agent harnesses has the same governance shape the paper specifies: evidence-grounded diagnosis (test-driven validation), persistent asset update (skill library updates that survive sessions), governed validation (regression suites). [165-ralph-autonomous-loop](165-ralph-autonomous-loop.md) and [167-autoskill-experience-driven-lifelong-learning](167-autoskill-experience-driven-lifelong-learning.md) implement parts of this; the paper is the spec they should be measured against.
- **Multi-regime hybrid harnesses.** [149-sector-use-case-catalog](149-sector-use-case-catalog.md), [127-loan-processing-multi-agent-case-study](127-loan-processing-multi-agent-case-study.md), [137-voice-agents](137-voice-agents.md), [138-text-to-sql-agents](138-text-to-sql-agents.md), [139-ocr-document-agents](139-ocr-document-agents.md), [141-e-commerce-marketing-agents](141-e-commerce-marketing-agents.md): real production agents straddle regimes. A loan-processing agent is Digital L2 (API contracts, policy state machines) + Social L2 (norms around fairness, disclosure) + Scientific L2 (statistical surrogates of credit risk). The paper's regime-coupling-cost regularity is real and visible: each regime's constraint set must be aligned with the others.
- **Eywa's federation, RecursiveMAS's bridges, and the world-model substrate.** [103-eywa-heterogeneous-fm-collaboration](103-eywa-heterogeneous-fm-collaboration.md) federates non-LLM FMs; [189-recursive-multi-agent-systems](189-recursive-multi-agent-systems.md) bridges heterogeneous LLMs. This paper's taxonomy says: Eywa's federation already crosses regimes (Chronos = Physical/Scientific L1, TabPFN = Digital L1, language reasoner = Social/Digital L1+L2). RecursiveMAS's latent bridges keep specialists in their regimes while composing trajectories. The three together — taxonomy + federation + bridge — sketch a cross-regime agentic architecture that currently exists nowhere as a production system.
- **The L4/L5 line of [105] maps to L2/L3 here.** The visual-generation paper's L4 (multi-pass with external verifier) is the Digital-regime instantiation of this paper's L2 (decision-usable simulator); its L5 (internalized causal dynamics) is the Physical/Scientific-regime instantiation of L3 (evidence-driven evolver). Cross-reading the two papers gives the strongest currently-available picture of where world-model-style generation sits inside the broader agentic-world-modeling discourse.
- **Beyond L3 / Meta-World Modeling as a long-term harness target.** A harness whose own world model is itself learnable — where governing laws are revisable structures rather than fixed regimes — is the paper's most ambitious frontier. [36-autogenesis-self-evolving-agents](36-autogenesis-self-evolving-agents.md), [69-agent-world-self-evolving-training-arena](69-agent-world-self-evolving-training-arena.md), [171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md): these are the closest current neighbors. None has yet *demonstrated* governed law revision; the paper is the spec under which their progress should be measured.

The one-line takeaway for harness designers: **"world model" is no longer one thing — it's a position in a 3×4 grid (capability × regime), and the COD test is the single sharpest disqualifier of systems that look planning-ready but aren't.**

## References and cross-links

- Foundation systems: Ha & Schmidhuber 2018; DreamerV1/V2/V3; MuZero; PILCO; JEPA / V-JEPA / I-JEPA.
- Generative-video adjacents: Sora; Genie 2; Oasis; GameNGen; UniSim; CogVideoX; HunyuanVideo; Wan2.1/2.2.
- Scientific surrogates: GraphCast; AlphaFold-as-surrogate; neural operator learning; molecular surrogates.
- Internal cross-links: [105-agentic-world-modeling-visual-generation](105-agentic-world-modeling-visual-generation.md) (visual-pipeline view), [37-neuro-symbolic-ai](37-neuro-symbolic-ai.md) (symbolic substrate), [128-knowledge-graphs-as-substrate](128-knowledge-graphs-as-substrate.md) (KG as L3 asset), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md) (planner/world-model loops), [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md) (trajectory eval), [86-frugalgpt](86-frugalgpt.md) / [87-routellm](87-routellm.md) (regime routing), [192-world-r1-3d-constraints-t2v](192-world-r1-3d-constraints-t2v.md) (L2 visual instance), [189-recursive-multi-agent-systems](189-recursive-multi-agent-systems.md), [103-eywa-heterogeneous-fm-collaboration](103-eywa-heterogeneous-fm-collaboration.md), [193-recursive-world-organizations-synthesis](193-recursive-world-organizations-synthesis.md).
- Project artifacts: paper https://arxiv.org/abs/2604.22748, project page https://agentic-world-modeling.xyz/, awesome list https://github.com/matrix-agent/awesome-agentic-world-modeling, arXiviq analysis https://arxiviq.substack.com/p/agentic-world-modeling-foundations.
