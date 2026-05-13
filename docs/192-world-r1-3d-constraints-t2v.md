# 192 — World-R1: Reinforcing 3D Constraints for Text-to-Video Generation

**Paper.** Weijie Wang, Xiaoxuan He, Youping Gu, Yifan Yang, Zeyu Zhang, Yefei He, Yanbo Ding, Xirui Hu, Donny Y. Chen, Zhiyuan He, Yuqing Yang, Bohan Zhuang — *World-R1: Reinforcing 3D Constraints for Text-to-Video Generation* — arXiv:2604.24764v1 — Zhejiang University / Microsoft Research — April 27, 2026 — accepted at **ICML 2026**. Code: https://github.com/microsoft/World-R1 (MIT, 325 stars). Project page: https://microsoft.github.io/World-R1/. Tech report: https://microsoft.github.io/World-R1/tech.html. HuggingFace daily papers: 115 upvotes (week of 2026-W18).

**One-line definition.** World-R1 is the first online-RL post-training recipe for text-to-video diffusion that injects **3D-reconstructibility as the reward signal** (not as an architectural inductive bias) — using Depth Anything 3 to reconstruct each generated video into a 3D Gaussian Splatting field, scoring meta-view fidelity, reconstruction LPIPS, and trajectory error, then optimizing **Flow-GRPO-Fast** on Wan2.1-T2V-1.3B/14B with no architectural change and no inference-cost overhead — yielding **+10.23 dB PSNR** on 3D-consistency benchmarks while preserving aesthetic quality through a **periodic decoupled training** safeguard against motion-suppression reward hacking.

## Why this paper matters

The "world model" framing of text-to-video has been mostly aspirational. Sora was marketed as a world simulator; Genie 2, Oasis, GameNGen, and UniSim sit in the same conceptual neighborhood; CogVideoX, Wan2.1, HunyuanVideo, and MovieGen optimize for visual fidelity and semantic alignment but make no operational claim about world-modeling validity. The taxonomy paper [190-agentic-world-modeling-taxonomy](190-agentic-world-modeling-taxonomy.md) argues that pixel coherence is not decision-usability and proposes Counterfactual Outcome Deviation as the disqualifier. World-R1 is the first paper to attack the gap from the *training-signal* side: instead of architectural 3D inductive biases (Fantasyworld's 3D decoder head, ViewCrafter's depth-conditioned generation, voxel/NeRF latents), it leaves the base T2V model and inference cost untouched and post-trains a **purely geometric reward** that asks: *can this video be reconstructed into a coherent 3D scene with a consistent camera trajectory?*

The recipe is sharp. (1) Camera-aware latent initialization injects intended motion as discrete noise warping on the initial latent, with no parameter change. (2) Stochastic rollouts under Flow-GRPO's reverse-time SDE produce sample groups for advantage estimation. (3) The reward stack: Depth Anything 3 reconstructs each rollout into a 3DGS field; Qwen3-VL scores a held-out meta-view (penalizes floaters, billboards, texture stretching); LPIPS-on-rerender measures fidelity to the input frames; trajectory error compares the recovered camera against the prompt's intended motion. (4) The composite reward is R = R_3D + λ_gen · R_HPSv3 — geometry plus aesthetics. (5) **Periodic decoupled training** is the load-bearing safeguard: every 100 RL steps, the system disables the 3D reward and trains briefly on aesthetic-only signal, preventing the model from collapsing to static rigid scenes (the trivial reward-hacking solution).

The quantitative results are large enough to settle the structural question. World-R1-Small (post-trained Wan2.1-T2V-1.3B) lifts PSNR from 17.40 → 27.63 (+10.23 dB), SSIM from 0.550 → 0.858, LPIPS from 0.467 → 0.201 against single-view 49-frame reconstruction. The Large variant (Wan2.1-T2V-14B) lifts PSNR from 19.76 → 27.67 (+7.91 dB). At 121-frame long-horizon evaluation, the large model holds up — PSNR 26.32 vs 18.32 baseline. Multi-view consistency (MVCS) lifts to 0.989 (Small) and 0.993 (Large). VBench aesthetic and imaging scores improve simultaneously. User study: 92% win rate on geometric consistency, 86% on overall preference.

Take this paper seriously and three downstream things change. (1) **3D-reconstructibility becomes a primary T2V quality dimension** — orthogonal to FVD and text-alignment, and likely the key dimension the [190](190-agentic-world-modeling-taxonomy.md) MREP test (Counterfactual Outcome Deviation) most cleanly approximates. (2) **Architecture-free post-training upgrades close the gap with architecturally-3D-aware models** — Fantasyworld and ViewCrafter add inference cost; World-R1 adds none and matches or beats them on reconstruction. (3) **The reward-hacking problem in video RL has a published mitigation pattern** — periodic decoupled training. Future video-RL papers can adopt this directly rather than rediscovering the static-scene collapse failure mode.

## Problem it solves

- **Generation-time 3D awareness costs inference.** Architectural 3D inductive biases (extra decoder heads, depth conditioning, voxel/NeRF latents) require modifications to the inference graph, increasing VRAM and latency. World-R1 leaves inference unchanged.
- **Scaling-only doesn't fix 3D consistency.** Wan2.1-14B vs Wan2.1-1.3B closes the gap from 17.40 → 19.76 PSNR — a 2.36 dB gain from 10× parameters. World-R1 lifts the 1.3B model by 10.23 dB, dwarfing the scaling effect.
- **Camera control alone is insufficient.** CameraCtrl, ReCamMaster, Trajectory-Attention, GCD inject camera trajectories at the input but do not enforce 3D-reconstructibility on the output. World-R1's reward closes the loop: the trajectory must be *recoverable* from the generated frames.
- **Reward-hacking under geometric supervision.** Pure 3D rewards push the model toward static rigid scenes (trivially reconstructible). Without intervention, the policy learns to generate near-frozen videos that suppress all non-rigid motion. Periodic decoupled training is the explicit mitigation.
- **No architecture-free baseline for "video as world model."** Pre-World-R1, the only way to make a T2V model 3D-consistent was architectural surgery. World-R1 demonstrates that an alignment-only intervention suffices, which has implications for every existing T2V model on the market.

## Core idea in one paragraph

Treat 3D-reconstructibility as a reward signal rather than an architectural prior. Generate video rollouts with stochastic Flow-GRPO sampling on a frozen Wan2.1-T2V backbone augmented with parameter-free camera-aware noise warping at the initial latent; reconstruct each rollout into a 3D Gaussian Splatting field via Depth Anything 3; score the rollout on (a) meta-view fidelity (Qwen3-VL judge of a rendered held-out viewpoint), (b) input-frame reconstruction quality (1 − LPIPS), and (c) trajectory error against the prompt's intended camera motion; combine with HPSv3 aesthetic reward at λ_gen = 1; optimize the policy with Flow-GRPO-Fast (group-relative advantage, clipped objective, KL to reference). Every 100 steps, disable the 3D reward and fine-tune briefly on the aesthetic reward over a small dynamic subset of prompts to prevent the model from collapsing to static rigid scenes — periodic decoupled training is the central safeguard against the obvious reward-hacking pathology. Result: a Wan2.1 model whose generated videos satisfy 3D-reconstructibility 10 dB better than baseline, with no architectural change, no inference overhead, and no 3D-supervised dataset.

## Mechanism (step by step)

### Base model

The "policy" is the open-source **Wan2.1-T2V** flow-matching diffusion transformer:
- World-R1-Small ← Wan2.1-T2V-1.3B
- World-R1-Large ← Wan2.1-T2V-14B
- Resolution: 832 × 480 during RL post-training.
- No architectural modification; LoRA-only post-training is supported (`scripts/infer_wan_lora.py`).

### Step 1 — Camera-aware latent initialization (parameter-free)

Following the *Go-with-the-Flow* paradigm, prompts are parsed for motion tokens (`push_in`, `pull_out`, `orbit_left/right`, `pan_left/right`, `move_left/right`, composites, `static`). Each motion token maps to a sequence of camera extrinsics; the extrinsics are projected to 2D optical flow under fronto-parallel scene assumptions; the optical flow is injected as **discrete noise warping on the initial latents**. A density-tracker normalizer preserves unit variance to prevent training instability.

This is parameter-free — no new layers, no new conditioning embeddings — and replaces ad-hoc camera control encoders (CameraCtrl-class). Ablation shows removing the noise warping yields significantly slower convergence and inferior trajectory alignment.

### Step 2 — Stochastic rollout via Flow-GRPO SDE

The deterministic ODE sampler is converted to a reverse-time SDE so the policy is stochastic and admits group-relative advantage estimation (Eq. 1–2):

  dx_t = [v_t(x_t) + (σ_t² / 2t)(x_t + (1 − t) v_t(x_t))] dt + σ_t dw

  x_{t+Δt} = x_t + [v_θ(x_t, t) + (σ_t² / 2t)(x_t + (1 − t) v_θ(x_t, t))] · Δt + σ_t · √Δt · ε

Group size G = 8 across 48 parallel groups. Hardware: 48 × H200 (Small) / 96 × H200 (Large).

The "Fast" variant of Flow-GRPO injects SDE noise only at randomly selected intermediate timesteps (denoise-step reduction), keeping rollout cost tractable while preserving advantage-estimation diversity.

### Step 3 — 3D reward extraction

For each generated rollout x₀:

1. **Reconstruct.** Depth Anything 3 produces a 3D Gaussian Splatting field Φ_GS and an estimated camera trajectory Ê for the video. (No NeRF, no per-scene optimization — direct feed-forward 3DGS.)
2. **Compute three sub-rewards** (Eq. 9: R_3D = S_meta + S_recon + S_traj):
   - **S_meta — meta-view fidelity.** Render Φ_GS from a held-out viewpoint; **Qwen3-VL** scores 0–9 (× 0.1 → [0, 1]) on the rendered meta-view, penalizing floaters, billboard artifacts, and texture stretching. The held-out viewpoint forces the 3D field to be valid beyond just the input frames.
   - **S_recon = 1 − LPIPS(x, x̂).** Perceptual fidelity between input frames and 3DGS-rerendered frames.
   - **S_traj.** Trajectory error: exp(−L₂) on translation, exp(−geodesic) on rotation, comparing Ê to the prompt's intended camera trajectory (the same one that produced the noise warping).
3. **Range**: R_3D ∈ [0, 3].

### Step 4 — Composite reward

Aesthetic reward **R_gen = (1/K) · Σ_t HPSv3(x_t)** averaged over K frames, ∈ [−1, 1]. HPSv3 is the modern human-preference score model.

**Composite (Eq. 8 / 10):**
  R(x, c) = R_3D(x, Ê, c) + λ_gen · R_gen(x, c),  λ_gen = 1

Both rewards are normalized to comparable scales; λ_gen = 1 is the only setting reported.

### Step 5 — Flow-GRPO-Fast optimization

Group-relative advantages (Eq. 3) and clipped surrogate objective with KL to a frozen reference policy (Eq. 4):

  Â^i_t = (R(x₀^i, c) − mean({R})) / std({R})

  J(θ) = E [ (1/T) · Σ_t (L_clip(r^i_t, Â^i_t) − β · KL(π_θ ‖ π_ref)) ]

The KL term anchors the post-trained policy to the Wan2.1 prior, preventing aesthetic regression as the geometric reward shapes behavior.

### Step 6 — Periodic decoupled training (anti-reward-hacking)

The load-bearing safeguard. Pure R_3D rewards push the policy toward static scenes (which are trivially 3D-consistent). Without intervention, ablation shows the model suppresses non-rigid motion; VBench AVG drops from 85.21 → 82.64.

Mitigation: every 100 RL steps, disable R_3D and train on a ~500-prompt dynamic subset using only R_gen (HPSv3 aesthetic). The policy is briefly pulled back toward dynamic, aesthetically-rewarded behavior before geometric optimization resumes. The result is a Pareto improvement on both axes: geometric consistency rises to the World-R1 numbers and aesthetic scores improve as well.

This is the published mitigation that future video-RL work can adopt directly.

## Empirical results

### Headline 3D-consistency (Table 2; reconstruction-based, single-view 49-frame)

| Method | PSNR ↑ | SSIM ↑ | LPIPS ↓ |
|---|---:|---:|---:|
| CogVideoX-1.5-5B | 24.44 | 0.783 | 0.242 |
| Wan2.1-T2V-1.3B | 17.40 | 0.550 | 0.467 |
| Wan2.1-T2V-14B | 19.76 | 0.629 | 0.405 |
| **World-R1-Small** | **27.63** | **0.858** | **0.201** |
| **World-R1-Large** | **27.67** | **0.865** | **0.162** |

- World-R1-Small lifts the 1.3B base by **+10.23 dB PSNR**, **+0.308 SSIM**, **−0.266 LPIPS**.
- World-R1-Large lifts the 14B base by **+7.91 dB PSNR**, **+0.236 SSIM**, **−0.243 LPIPS**.
- World-R1-Small *beats* Wan2.1-T2V-14B (10× larger) by +7.87 dB PSNR — RL post-training dwarfs scaling.

### Long-horizon (121-frame) generalization

World-R1-Large vs Wan2.1-14B baseline:
- PSNR: 26.32 vs 18.32 (+8.00 dB)
- SSIM: 0.828 vs 0.558 (+0.270)
- LPIPS: 0.257 vs 0.534 (−0.277)

Long-horizon results are weaker than 49-frame (PSNR drops from 27.67 → 26.32) but the relative gain over baseline holds.

### Reconstruction-independent: MVCS (multi-view consistency, from GeoVideo)

| Model | MVCS |
|---|---:|
| Wan2.1-1.3B | 0.974 |
| **World-R1-Small** | **0.989** |
| Wan2.1-14B | 0.963 |
| **World-R1-Large** | **0.993** |

MVCS is computed on multi-view sets *without* requiring 3D reconstruction, so this confirms the gain isn't an artifact of the reward's reconstruction stack.

### Camera control accuracy (World-R1-Large)

- RotErr: 1.21
- TransErr: 1.30
- CamMC: 2.95

(Lower is better. Comparable or superior to dedicated camera-control methods like CameraCtrl, ReCamMaster, Trajectory-Attention.)

### VBench (aesthetic / imaging quality, World-R1-Small)

- Aesthetic: 65.74 (vs Wan2.1-1.3B 62.43)
- Imaging: 67.53 (vs Wan2.1-1.3B 66.51)
- Motion Smoothness: 98.55 (vs 97.44)
- Subject Consistency: 97.58

VBench scores *improve* alongside 3D consistency — the periodic decoupled training preserves and slightly enhances aesthetic quality.

### User study (25 participants, double-blind)

- 92% win rate on **geometric consistency**
- 76% win rate on **camera-control accuracy**
- 86% win rate on **overall preference**
- Human/auto-metric agreement: 91.17% across 30 video pairs × 20 raters

### Baselines compared (paper §4)

- CogVideoX-1.5-5B
- Wan2.1-T2V-1.3B / 14B
- Wan2.2-T2V-5B / 14B
- Camera-control methods: CameraCtrl, ReCamMaster, Trajectory-Attention, GCD, DAS, TrajectoryCrafter, CamCloneMaster
- 3D-aware T2V methods: ViewCrafter, AC3D, VidCraft3, RealCam-I2V, FlashWorld, Voyager, Fantasyworld

**Notable absence: no direct comparison to closed-source Sora, Veo, MovieGen, HunyuanVideo.** This is a real scope limitation — the paper frames itself narrowly as a Wan2.1 post-training recipe and doesn't benchmark against the systems most associated with the "video as world model" discourse.

## Variants and ablations (on World-R1-Small)

- **Remove R_3D.** Geometric consistency collapses to near-baseline; LPIPS regresses to ~0.45. R_3D is "fundamental for establishing geometric consistency."
- **Remove R_gen.** Aesthetic and perceptual quality drop sharply. R_gen is "indispensable for maintaining high perceptual fidelity." Both rewards are required.
- **Remove implicit camera conditioning (noise warping).** "Significantly slower convergence and inferior trajectory alignment." The parameter-free camera injection is doing real work.
- **Remove periodic decoupled training.** Model overfits to rigid scenes; suppresses non-rigid dynamics. VBench AVG 82.64 (degenerate, motion suppressed) vs 85.21 (full recipe). This is the most important ablation in the paper.
- **Scene-complexity breakdown.** Big gains on static and single-object scenes; smaller absolute gains on long-horizon (PSNR 23.59 — lowest tier) and non-rigid (LPIPS 0.267 vs 0.548 baseline) — still improved but harder.

## Failure modes and limitations

- **Reward hacking → motion suppression.** The recurring failure mode if you remove periodic decoupled training. With the safeguard, mitigated but not eliminated; long-horizon and non-rigid breakdown numbers are still the weakest.
- **Inherited base-model weaknesses.** Per the paper: "Dense multi-object composition, fine-grained non-rigid motion, detailed hand dynamics, and very long-horizon scene evolution may still inherit artifacts." World-R1 is alignment, not new capability — what Wan2.1 cannot generate, World-R1 cannot rescue.
- **RL compute cost.** Online RL with video rollouts + reward evaluation is "more expensive than standard post-training." 48 × H200 (Small) and 96 × H200 (Large) are the published configurations; cost not in dollars but the hardware floor is high.
- **Reward dependence on the reconstruction stack.** Failures in Depth Anything 3 or Qwen3-VL propagate as reward noise. Not quantified; the assumption that the reconstruction stack is always reliable is implicit.
- **Coverage of evaluation.** Reconstruction-centric benchmark naturally favors the method's own training signal. No comparison to Sora / Veo / MovieGen / HunyuanVideo / Wan2.2 on all axes. Comparison is to open-source T2V at Wan2.1's tier, not to the field's frontier.
- **No connection to Counterfactual Outcome Deviation.** The paper doesn't run the [190](190-agentic-world-modeling-taxonomy.md) MREP test directly. Whether a 3D-reconstructible video is also action-sensitive (the COD test) is an empirical question this paper doesn't answer. Reconstructibility is a *necessary* condition for being a planning-usable simulator, not sufficient.
- **Wan2.1-only.** Recipe is demonstrated on flow-matching diffusion transformers in the Wan2.1 family; whether it transfers to autoregressive video models, Sora-class architectures, or interactive game-generation models (Genie 2, GameNGen) is unstated.

## When to use, when not

**Use** when post-training a T2V model that needs to produce 3D-consistent output without architectural change (existing infrastructure, deployed inference graph, fixed VRAM budget); when geometric consistency is a primary product requirement (camera-controlled video, 3D scene synthesis, view-consistent narrative); when the team has access to GPU clusters (48–96 × H200 floor); when periodic decoupled training fits the RL budget; for upgrading open-source T2V models on the Wan2.1 / CogVideoX / similar tier.

**Don't use** as a substitute for action-conditioned world modeling — World-R1 produces 3D-consistent *visualizations*, not 3D-consistent *simulators-you-can-plan-against* (that requires the COD test [190](190-agentic-world-modeling-taxonomy.md)); for tasks dominated by non-rigid motion (dance, fluid simulation, dense people-in-motion) where reward hacking risk is highest and gains are smallest; when low-resource RL is required (the 48 × H200 floor is high); for inference-only deployments where the gains have already been baked in by other 3D-aware methods (use the ablation losses to decide); for closed-source T2V (Sora, Veo, MovieGen) where you can't access weights or training stack.

## Implications for harness engineering

- **3D-reconstructibility is a reward, not an architecture.** This is the structural lesson. Every existing T2V deployment can in principle adopt this recipe — leave the inference graph untouched, post-train on a reconstruction reward. Harness designers shipping video-generation as a tool ([136-multimodal-rag](136-multimodal-rag.md), [137-voice-agents](137-voice-agents.md), [105-agentic-world-modeling-visual-generation](105-agentic-world-modeling-visual-generation.md)) get a near-free upgrade path.
- **Periodic decoupled training is a generalizable mitigation pattern.** Any RL post-training that risks reward hacking by collapsing to a trivial-but-high-reward subspace (static scenes here; safe-but-useless responses in RLHF; refusal-spam in safety alignment) can borrow the recipe: every N RL steps, disable the constrained reward and run a brief auxiliary phase on the diversity-preserving signal. Harness teams running RLHF / RLAIF cycles ([97-qwen-prm](97-qwen-prm.md), [80-knowrl](80-knowrl.md)) should add this to the standard toolbox.
- **The reconstruction-as-reward pattern generalizes beyond video.** Image generation can use 3D-reconstructibility from a single image (mesh-from-image, NeRF distillation); 3D scene generation can use trajectory-recoverability; long-form text generation can use entity-graph-recoverability. The unifying principle: ground a perceptual quality dimension by *recovering a structured representation* and scoring its fidelity.
- **Connects to the [190](190-agentic-world-modeling-taxonomy.md) L1/L2/L3 ladder.** World-R1 produces a system at L1 (one-pass video predictor) that is *closer to L2-readiness* by virtue of 3D-reconstructibility. Whether it is L2 — passes the boundary conditions of long-horizon coherence, intervention sensitivity, constraint consistency — is empirically open. Long-horizon coherence: improved (8 dB gain at 121 frames). Intervention sensitivity: untested (no COD probes). Constraint consistency: improved (3D-reconstructibility is a soft physical constraint). Two of three tests pass; the third is the open question.
- **Verifier-loop pattern with structured verifiers.** [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md) describes planner/generator/evaluator harnesses. World-R1's evaluator stack (Depth Anything 3 + Qwen3-VL + LPIPS + trajectory) is a multi-component structured verifier; the harness pattern of "generate → verify-with-multiple-specialists → re-generate" maps directly. Multimodal harnesses ([136-multimodal-rag](136-multimodal-rag.md), [105-agentic-world-modeling-visual-generation](105-agentic-world-modeling-visual-generation.md)) should adopt structured-verifier stacks as the default.
- **Eywa's federation comes back.** [103-eywa-heterogeneous-fm-collaboration](103-eywa-heterogeneous-fm-collaboration.md) federates non-LLM FMs at *inference time*. World-R1 federates non-LLM FMs (Depth Anything 3, Qwen3-VL) at *training time* — they sit on the reward side of the loop, scoring rollouts. The same pattern (LLM coordinates with domain FMs through structured interfaces) shows up at both inference and training.
- **Co-evolution with the verifier.** [169-coevoskills-co-evolutionary-verification](169-coevoskills-co-evolutionary-verification.md) describes co-evolving generator and verifier. World-R1 freezes the verifier (Qwen3-VL is fixed). Future versions could co-evolve the verifier, learning to detect new failure modes as the generator improves — a natural extension when the RL recipe is mature.
- **Camera control as a controllable axis for visual agents.** Agents that produce video as part of a larger plan (e.g., a video-storyboard agent, a UI-walkthrough generator) need camera control. World-R1's parameter-free noise-warping injection is a recipe for adding camera-axis controllability to any flow-matching T2V model without adding parameters. This is the kind of harness primitive that lives between [04-skills](04-skills.md) (skills as capabilities) and [07-model-context-protocol](07-model-context-protocol.md) (tools as interfaces).
- **Sora-as-world-simulator critique gets an empirical lever.** [190](190-agentic-world-modeling-taxonomy.md) argues structurally that pixel coherence is not decision-usability. World-R1 provides the lever: a 3D-reconstructibility test that any T2V model can be evaluated on, plus a recipe to upgrade an open-source baseline to pass it. Future "video as world model" claims should be evaluated against both World-R1's reconstruction metrics and the [190](190-agentic-world-modeling-taxonomy.md) MREP COD test.
- **The PSNR gap dwarfs the scaling gap.** World-R1-Small (1.3B) beats Wan2.1-14B by +7.87 dB PSNR. Post-training >> scaling for this capability dimension. This is consistent with the broader RL post-training findings ([86-frugalgpt](86-frugalgpt.md), [97-qwen-prm](97-qwen-prm.md)) and reinforces the harness lesson: spend the next compute on alignment, not parameters.
- **The "world model" name is doing some marketing work.** Microsoft's GitHub description, the project page name "World-R1," and the connection to broader world-model discourse position the paper in the agentic-world-modeling neighborhood. The paper itself is more modest — it's a T2V post-training recipe with 3D-reconstructibility as the signal. Harness designers reading "World-R1" should not expect an interactive simulator (Genie 2, GameNGen) — they should expect a 3D-aware video generator with no architectural overhead.

The one-line takeaway for harness designers: **3D-reconstructibility-as-reward upgrades existing T2V models by 10 dB without architectural change, periodic decoupled training is the generalizable safeguard against motion-suppression reward hacking, and the recipe brings open-source video generation a step closer to the [190](190-agentic-world-modeling-taxonomy.md) L2 boundary — but not yet across it.**

## References and cross-links

- Parent algorithm: Flow-GRPO (cited [24]); GRPO from DeepSeek [41].
- Concurrent peer: VGGRPO (arXiv:2603.26599, March 2026) — latent-space 4D reward; works on dynamic scenes more naturally; reward in latent space avoids VAE decode.
- Adjacent: Wan-R1 (arXiv:2603.27866) — verifiable RL for video reasoning, orthogonal; Video-R1 (arXiv:2503.21776) — RL for video reasoning in MLLMs, orthogonal.
- Architecturally-3D-aware T2V (avoided here): Fantasyworld (3D decoder head), ViewCrafter (depth-conditioned), AC3D, VidCraft3, RealCam-I2V, FlashWorld, Voyager.
- Camera-control lineage: CameraCtrl, MotionCtrl, MotionBooth, AnimatedDiff, ReCamMaster, GCD, Trajectory-Attention, DAS, Go-with-the-Flow [34] (the noise-warping inspiration).
- Internal cross-links: [105-agentic-world-modeling-visual-generation](105-agentic-world-modeling-visual-generation.md) (visual-pipeline taxonomy; World-R1 is an L4-grade agent on its 5-level scale), [190-agentic-world-modeling-taxonomy](190-agentic-world-modeling-taxonomy.md) (L1/L2/L3 ladder; World-R1 is L1 closer to L2), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md) (structured verifier pattern), [103-eywa-heterogeneous-fm-collaboration](103-eywa-heterogeneous-fm-collaboration.md) (FM federation at training-time), [169-coevoskills-co-evolutionary-verification](169-coevoskills-co-evolutionary-verification.md) (co-evolving generator/verifier), [136-multimodal-rag](136-multimodal-rag.md), [193-recursive-world-organizations-synthesis](193-recursive-world-organizations-synthesis.md).
- Project artifacts: paper https://arxiv.org/abs/2604.24764, project page https://microsoft.github.io/World-R1/, code https://github.com/microsoft/World-R1, tech report https://microsoft.github.io/World-R1/tech.html, MarkTechPost analysis https://www.marktechpost.com/2026/04/30/microsoft-researchs-world-r1-uses-flow-grpo-and-3d-aware-rewards-to-inject-geometric-consistency-into-wan-2-1-without-architectural-changes/.
