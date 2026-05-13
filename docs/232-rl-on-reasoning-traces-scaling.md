# 232 — RL-on-Reasoning-Traces Scaling: how RL pushes test-time compute into the model

**Papers.** DeepSeek-AI — *DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via Reinforcement Learning* — arXiv:2501.12948 — 2025. OpenAI — *Learning to Reason with LLMs* (o1 system card) — September 2024. Allen AI — *Tülu 3: Pushing Frontiers in Open Language Model Post-Training* — arXiv:2411.15124 — Nov 2024 (RLVR recipe). Companion: *Process Reward Models for Math* (97), *Let's Verify Step by Step* (Lightman 2023, [223](223-verifier-and-best-of-n-scaling.md)), *Snell-2024 Test-Time Compute Scaling* ([217](217-test-time-compute-scaling.md)), DeepSeek-R1-Zero (pure-RL ablation), KnowRL ([80](80-knowrl.md)), HeavySkill RLVR ([156](156-heavyskill-rlvr.md)).

**One-line definition.** A line of work showing that **reinforcement learning with verifiable rewards (RLVR) on chain-of-thought traces** elicits o1-class long-thinking behaviour from a base model — DeepSeek-R1's pure-RL recipe lifts AIME-2024 from 15.6 % (DeepSeek-V3 base) to **71.0 %** with R1-Zero and **79.8 %** with R1-distill-32B, and MATH-500 from 90 % → 97.3 %, **without any supervised fine-tuning of reasoning traces** — making it the most compute-efficient way to internalize the test-time-compute axis ([217](217-test-time-compute-scaling.md)) into the base model ([216](216-pretraining-scaling-laws-foundation.md)).

## Why this paper matters (RL on reasoning traces is the productization mechanism that folds axis A2 into axis A1)

The Snell-2024 framework ([217](217-test-time-compute-scaling.md)) treats test-time compute as an **external** scaling axis — the harness allocates `(N, T, search-method)` per prompt by running the model for more samples or longer revisions. RLVR is the *internalization* of this allocation: train the model so that it *naturally* spends more test-time compute on hard problems, less on easy ones, and chooses the right search-strategy implicitly through learned policy. The result is "thinking models" — a single forward pass produces 1k–30k tokens of reasoning, branches and reflections internally, and terminates when an internal verifier signals confidence. The harness no longer needs an explicit difficulty router or external PRM-search loop; the model does both.

DeepSeek-R1 is the first publicly documented recipe of this kind. The team trained DeepSeek-V3-Base (a 671 B-parameter MoE with ~37 B active) with **pure GRPO (Group Relative Policy Optimization)** RL on a curated set of math, code, science, and logic problems with rule-based verifiable rewards — correctness on math word problems, test-suite pass-rate on code, format compliance. After RL, the model spontaneously developed **chain-of-thought lengths of 5k–20k tokens**, **self-correction**, **alternative-strategy exploration**, and a documented **"aha moment"** mid-training where the model began emitting "let me reconsider" and "wait, that doesn't work" in traces. R1-Zero (the pure-RL version) reaches AIME-2024 **71.0 %** vs DeepSeek-V3 base's 15.6 %; R1 (RL + targeted SFT cold-start to fix readability) reaches **79.8 %**. Distilling R1 into Qwen-32B, Qwen-14B, Llama-70B variants transfers most of the capability — Qwen-32B-distill hits AIME 72.6 %, beating o1-preview.

OpenAI's o1 (Sep 2024) was the first commercial deployment, with announced AIME-2024 of **83.3 %**, AIME-2025 of **89.0 %** for o3 (Dec 2024). The exact recipe is undisclosed, but the public characterization — large-scale RL on reasoning traces with PRMs and rule-based rewards — matches the DeepSeek-R1 architecture closely. The o3-mini family (released Jan 2025) demonstrates that the recipe scales down: a smaller base + heavy RL produces o1-class capability at lower per-call cost. Anthropic's Claude *extended thinking* (Feb 2025) is the third major productization in the same paradigm.

Tülu-3 (Allen AI, Nov 2024) provides the **open recipe**: a complete post-training stack including SFT, DPO, and RLVR with detailed dataset cards, hyper-parameters, and ablations. Their **RLVR component** specifically — RL with verifiable rewards on math/code/precise-instruction-following — is the open analogue of the proprietary recipe behind o1 / R1, applied to Llama-3.1 8B / 70B base and demonstrating consistent gains across MATH, GSM8K, IFEval, and AlpacaEval. Tülu-3 is what made the recipe reproducible.

Take this evidence seriously and three things change. **First**, you stop architecting for *external* test-time-compute allocation when a thinking model is available — the model handles allocation internally; the harness's job becomes *budget enforcement* (max-tokens cap, deliberation-time gate) rather than *router design*. **Second**, you understand that **the agent-era frontier is now compounding two RL stages**: pretraining (Chinchilla) sets the floor; RLVR sets the reasoning capability above it. The two stages are largely independent — strong base + RLVR yields R1 / o1; weak base + RLVR yields a smaller but capable thinking model. **Third**, you treat **distillation from a thinking-model teacher** ([235](235-inference-compression-scaling.md)) as a first-class deployment lever: a 32 B Qwen-distill of R1 outperforms a 70 B vanilla model, fundamentally changing the inference economics for reasoning-heavy workloads.

## Problem it solves (turning verifier-augmented inference compute into a one-stage post-training step)

1. **External TTC has high marginal inference cost.** Snell-2024's BoN/beam/lookahead methods cost N · T · model-cost per query at inference. RLVR pays this once at training; inference is one (long) chain.
2. **Difficulty routing was an external classifier.** With RLVR, the model itself learns to allocate more reasoning to harder prompts; no router needed for many task types.
3. **PRMs at inference scale linearly with N.** When the model emits a verified-by-design trace, the external PRM is no longer the bottleneck; the model has internalized the verifier.
4. **SFT alone could not elicit reasoning behaviour.** Pre-2024 attempts to SFT models on synthetic CoT data produced *imitation* of reasoning rather than authentic exploration. R1-Zero's pure-RL ablation showed RL is the *causal* mechanism — SFT helps with readability but not capability.
5. **Open-recipe gap.** Until DeepSeek-R1 and Tülu-3, the recipes for o1-class behaviour were proprietary; reproducible recipes unlocked the rest of the field.
6. **Distillation pathway.** R1 → R1-Distill-Qwen-32B / Llama-70B reaches near-flagship reasoning capability at consumer-deployable scale; the recipe transfers.

## Core idea in one paragraph

Take a strong base model B (DeepSeek-V3, Llama-3.1, Qwen-2.5). Apply **reinforcement learning with verifiable rewards (RLVR)**: sample completions on prompts where correctness can be checked by a deterministic rule (math: numeric equality with gold; code: test-suite pass; logic: solver agreement; format: regex compliance), score each completion by the binary or graded reward, and apply policy-gradient updates (GRPO, PPO, or off-policy variants). After tens of thousands of GPU-hours of RL with carefully curated prompts and reward shaping, the model's policy *spontaneously* develops longer, more reflective, more exploratory chains of thought — "thinking" — and its accuracy on math, code, and reasoning benchmarks rises from base-model levels to o1-class levels (AIME 70–90 %, MATH 95–97 %, MMLU-Pro 75–84 %). The mechanism is partly **exploration** (RL searches for high-reward token sequences and finds longer, more deliberate ones), partly **credit assignment** (the verifier reward at trace end propagates back through the trace via PPO's GAE or GRPO's group-relative baseline), and partly **emergent meta-reasoning** (DeepSeek-R1's documented "aha moment" — the model learns to *reflect* on its own reasoning mid-trace). The recipe internalizes the [217](217-test-time-compute-scaling.md) test-time-compute axis into the model: at inference, a single prompt elicits a long, self-verified trace where Snell-2024 would have required external BoN-N + PRM-search.

## Mechanism (step by step)

### (a) Base model + verifiable-reward dataset

Curate prompts whose answer admits a deterministic correctness check:

- **Math:** Olympiad problems, AIME, GSM8K-style, MATH; gold final-answer numeric equality.
- **Code:** LeetCode-style, Codeforces, competitive-programming with test-suite pass-rate.
- **Logic:** Boolean satisfiability, propositional reasoning with solver agreement.
- **Format compliance:** instruction-following with regex-checkable structure (Tülu-3 IFEval-style).

DeepSeek-R1 used roughly **800K–1M curated prompts** across these categories. Tülu-3's RLVR mix included ~100K math + 100K code + 50K format-compliance prompts.

### (b) GRPO (Group Relative Policy Optimization)

DeepSeek's main innovation in the RL algorithm. Standard PPO requires a value-function head; GRPO replaces this with a **group baseline** computed from K rollouts per prompt. For each prompt, sample K completions, compute reward `r_i` for each, normalize: `A_i = (r_i − mean(r)) / std(r)`. Apply PPO-style policy update with clipped advantage. Removes the value-function compute (~50 % of PPO compute) and stabilizes training when rewards are sparse and binary.

### (c) Reward shaping

Pure binary correctness rewards are sparse and slow to train. Tülu-3 and R1 add:

- **Format reward:** the model must emit the answer in `<answer>...</answer>` tags. Avoids reward-hacking by partial-string matching.
- **Length penalty (optional):** discourage runaway-length reasoning that doesn't improve correctness. R1-Zero specifically did *not* use this and saw lengths grow to 20k tokens.
- **Repeat-detection penalty:** RL can push the model into degenerate repetition; explicit penalty helps.

### (d) The "aha moment" — emergent reflection

DeepSeek-R1 §3.2 documents a specific training step (~step 5000 of GRPO) where the model spontaneously begins emitting:

- "Wait, let me reconsider…"
- "I think I made a mistake here. Let me try again."
- "Alternative approach: what if…"

These tokens did not exist in the training distribution at high frequency; they emerged from RL exploration finding that *reflective* trajectories more often hit the gold answer. The "aha moment" is the empirical signature of *internalized test-time compute allocation*.

### (e) The pure-RL R1-Zero vs R1 ablation

R1-Zero: DeepSeek-V3-Base + GRPO RL only, no SFT. AIME 71.0 %, MATH 95.9 %. **Demonstrates RL alone produces reasoning capability** — SFT is not the causal mechanism.

R1: V3-Base + small SFT cold-start (~1K human-curated reasoning traces) + GRPO RL + post-training filter. AIME 79.8 %, MATH 97.3 %. The SFT cold-start fixes readability issues (R1-Zero traces were sometimes language-mixed and hard to read) but does not change underlying capability.

### (f) Distillation transfers most capability

R1-Distill-Qwen-7B/14B/32B and R1-Distill-Llama-8B/70B: trained by SFT on R1 traces. AIME on Distill-32B = 72.6 %, beating o1-preview's 56.7 %. The recipe transfers without re-running RL — the trace data captures the essence.

### (g) Internal vs external TTC duality

The Snell-2024 frame allocates `(N, T)` externally; the o1/R1 frame learns it internally:

| Frame | Allocator | Inference cost | Verifier |
|---|---|---|---|
| Snell external | Harness with difficulty router + PRM | N · T forward passes | External PRM scores N candidates |
| RLVR internal (R1, o1) | Model itself | 1 long forward pass (5k–30k tokens) | Internalized — model emits self-checks |

The two frames are duals; the choice is operational, not capability. RLVR-internal is cheaper at inference and slower at training; external is cheaper at training and more expensive per query.

## Empirical results (table)

**Table 1 — DeepSeek-R1 / R1-Zero / Distill on math benchmarks**

| Model | AIME-2024 | MATH-500 | GPQA-Diamond | LiveCodeBench |
|---|---:|---:|---:|---:|
| DeepSeek-V3-Base | 15.6 % | 90.2 % | 39.0 % | 36.2 % |
| R1-Zero (pure RL) | 71.0 % | 95.9 % | 73.3 % | 50.0 % |
| **R1** (RL + SFT cold-start) | **79.8 %** | **97.3 %** | **71.5 %** | **65.9 %** |
| R1-Distill-Qwen-7B | 55.5 % | 92.8 % | 49.1 % | 37.6 % |
| R1-Distill-Qwen-14B | 69.7 % | 93.9 % | 59.1 % | 53.1 % |
| R1-Distill-Qwen-32B | 72.6 % | 94.3 % | 62.1 % | 57.2 % |
| R1-Distill-Llama-70B | 70.0 % | 94.5 % | 65.2 % | 57.5 % |

**Table 2 — Comparison with proprietary thinking models**

| Model | AIME-2024 | MATH-500 | GPQA-D | Notes |
|---|---:|---:|---:|---|
| o1-preview | 56.7 % | 85.5 % | 73.3 % | OpenAI Sep 2024 |
| o1 | 79.2 % | 96.4 % | 75.7 % | OpenAI Dec 2024 |
| o3 (preview) | 96.7 % | — | 87.7 % | Dec 2024 announcement |
| Claude 3.7 Sonnet (extended thinking) | ~75 % | ~96 % | ~75 % | Anthropic Feb 2025 |
| **R1** | 79.8 % | 97.3 % | 71.5 % | Open-weights, $5.5 M training cost |

**Table 3 — Tülu-3 RLVR ablation (Llama-3.1-8B base, Allen AI 2024)**

| Stage | MATH | GSM8K | IFEval | AlpacaEval LC |
|---|---:|---:|---:|---:|
| Base (Llama-3.1-8B) | 17.2 % | 56.7 % | 49.4 % | 22.8 % |
| + SFT | 31.5 % | 76.2 % | 64.2 % | 35.4 % |
| + DPO | 33.9 % | 79.4 % | 66.4 % | 40.0 % |
| **+ RLVR** | **35.7 %** | **82.5 %** | **75.4 %** | **41.6 %** |

**Table 4 — Trace length emergence (R1-Zero training curve, illustrative from §3.2)**

| Training step | Avg trace length (tokens) | AIME-2024 |
|---:|---:|---:|
| 0 (base) | 200 | 15.6 % |
| 1,000 | 800 | 35 % |
| 5,000 ("aha") | 3,500 | 55 % |
| 8,000 | 6,000 | 65 % |
| 10,000 (final) | 7,500 (with tail to 20k) | 71.0 % |

## Variants and ablations

- **GRPO vs PPO.** GRPO's group baseline removes value-function compute; ~30–50 % cheaper at same final reward.
- **Pure RL vs SFT-warmup vs DPO-bridge.** R1-Zero shows pure RL works; R1 adds SFT for readability; PPO/DPO bridges have mixed results.
- **Single-task vs multi-task RL.** Tülu-3 and R1 both train on multiple verifiable-reward task families simultaneously; capability transfers.
- **Reward model distillation.** Use a strong PRM ([223](223-verifier-and-best-of-n-scaling.md), [97](97-qwen-prm.md)) as the reward signal on tasks lacking gold answers — bridges to RLHF.
- **Self-correction loops in training.** Allow the model to emit `<reflect>...</reflect>` tags during RL; score the post-reflection answer; trains explicit reflection capability.
- **Long-context RL.** Scaling trace length beyond 32K tokens requires KV-cache-aware infrastructure; R1 trained at 32K context.
- **MoE × RLVR.** R1 is built on V3 (MoE 671 B / 37 B active); MoE inference cost is favourable for long thinking traces — fewer active params per token.
- **Verifier-free RL.** AlphaProof and DeepMind's math systems use Lean / formal verifier rewards — cleanest but narrow.
- **Self-play RLVR.** Synthetic problem generation + RL; risks reward hacking but unlimited training data.
- **Distill-then-RL.** Start with R1-Distill, apply additional task-specific RL; compounds gains.

## Failure modes and limitations

- **Reward hacking.** Models learn to game weak verifiers (partial-string matching, format-only rewards). Tight, deterministic verifiers and held-out evals are essential.
- **Length-quality decoupling.** RL can push the model into very long traces that don't improve correctness ("rambling"). R1-Zero exhibited this without explicit length control.
- **Domain narrowness of verifiable rewards.** RLVR works for math, code, logic, format. Open-ended writing, advisory, conversation — no rule-based verifier exists; RLHF/DPO with PRMs is the alternative.
- **Distribution shift after RL.** Heavy RL on math/code can degrade general-purpose helpfulness (R1 has documented multilingual and helpfulness regressions vs DeepSeek-V3).
- **Catastrophic mode collapse.** Without entropy bonuses or KL constraints to a reference model, RL can collapse to a single high-reward strategy. PPO's KL term and GRPO's group baseline help.
- **Compute cost is large.** R1 training reportedly $5.5 M; o1-class proprietary RL is presumed 10× higher. Open recipes shift this dramatically — Tülu-3 RLVR is a fraction of full RL.
- **Verifier coverage is the bottleneck.** New domains require new verifiers; Lean-style formal verification is expensive to build.
- **Trace readability.** R1-Zero traces were notoriously language-mixed (Chinese-English code-switching, made-up tokens). SFT cold-start partly fixes this; full fix requires more careful reward shaping.
- **Distillation has a ceiling.** R1-Distill-Qwen-32B reaches ~90 % of R1's capability; the last 10 % requires the full RL run.
- **Inference cost shifts to per-token.** A thinking model emits 5k–30k tokens per query; aggregate inference cost is large even if per-call API pricing is favourable.

## When to use, when not

**Use RLVR / thinking-model approach** for reasoning-heavy production workloads (math, code, science QA, structured planning); when you need o1-class capability at deployable cost; when distillation pathway is acceptable; when verifier infrastructure exists for your domain ([97](97-qwen-prm.md), [223](223-verifier-and-best-of-n-scaling.md)); when extended thinking latency is acceptable to users; when you have or can rent the ~$5–50 M training compute for the full recipe (or use distillation from a published thinking model). The strongest case is **distill an existing thinking model** (R1, R1-distill-Qwen series, Phi-4) into your task domain rather than training from scratch.

**Do not** use RLVR when your tasks lack verifiable rewards (creative writing, advisory, open-ended chat); when latency is sub-second-bound (thinking models emit 5k+ tokens — inherently slow); when you cannot afford the per-query token-count economics; when general-purpose helpfulness must be preserved (heavy RL can degrade it); when you cannot maintain robust verifiers (reward hacking will undermine training); or when external test-time-compute via Snell-2024 is more cost-effective for your access pattern.

## Implications for harness engineering

- **Thinking models change the TTC harness role from router to budget enforcer.** [217-test-time-compute-scaling](217-test-time-compute-scaling.md) — when the model thinks internally, the harness caps tokens and deliberation time, not N or T.
- **Verifier infrastructure compounds.** [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md), [97-qwen-prm](97-qwen-prm.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md) — the verifier that scored RL training is also the verifier-checkpoint at inference.
- **Cross-channel verification still mandatory.** [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md), [19-verifier-cross-channel](../projects/polaris/docs/blocks/19-verifier-cross-channel.md) — even thinking models can be wrong; an external different-family verifier remains the bright-line safety net.
- **Distillation is the production deployment lever.** [235-inference-compression-scaling](235-inference-compression-scaling.md) — R1 → R1-Distill-Qwen-32B is the pattern; train one large thinking model, distill to deployment-size students.
- **Skill memory cooperates with thinking.** [05-reasoning-bank](../projects/polaris/docs/blocks/05-reasoning-bank.md), [81-reasoningbank](81-reasoningbank.md), [156-heavyskill-rlvr](156-heavyskill-rlvr.md) — past thinking traces inform new ones; the bank-and-think loop compounds.
- **HeavySkill-style inner skills become RL-trainable.** [156-heavyskill-rlvr](156-heavyskill-rlvr.md), [p10-heavyskill-rlvr](../projects/polaris/docs/research/p10-heavyskill-rlvr.md) — the "parallel-then-deliberate" inner skill can itself be RLVR-trained.
- **Cost-routing across thinking and non-thinking variants.** [88-confidence-driven-router](88-confidence-driven-router.md), [87-routellm](87-routellm.md), [86-frugalgpt](86-frugalgpt.md) — easy prompts → fast non-thinking model; hard prompts → thinking model.
- **Bright-line cost gates against runaway thinking.** [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md), [p13-cost-discipline](../projects/polaris/docs/research/p13-cost-discipline.md) — cap thinking-token budget; escalate on exceedance.
- **HIR observability for thinking traces.** [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md) — log thinking-token count, reflection occurrences, branching points; needed for cost attribution and debugging.
- **Trajectory simulation extends to thinking traces.** [142-trajectory-simulation-agents](142-trajectory-simulation-agents.md), [195-argus-omega-vol-2-trajectory-temporal-horizon](195-argus-omega-vol-2-trajectory-temporal-horizon.md) — replay thinking traces for offline eval before paying live inference.
- **Plan mode + thinking model.** [03-plan-mode](03-plan-mode.md), [p4-pre-research](../projects/polaris/docs/research/p4-pre-research.md) — pre-execution thinking trace as the planning artefact; review before action.
- **Verifiers as RL infrastructure.** [80-knowrl](80-knowrl.md), [156-heavyskill-rlvr](156-heavyskill-rlvr.md) — every PRM you train doubles as RL reward; align verifier and reward investment.
- **Domain-specific RLVR is reachable.** Tülu-3 demonstrated open recipes for ~$50K-scale RL stages; small labs can train domain-specific thinking models on bio, finance, security tasks.

**One-line takeaway for harness designers.** **RLVR is the productization mechanism that folds the test-time-compute axis (217) into the base-model axis (216) — once you have a thinking model, the harness shifts from external compute allocator to budget-and-verifier enforcer, and distillation makes the recipe deployable at consumer scale.**
