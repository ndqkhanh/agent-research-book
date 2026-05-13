# 235 — Inference Compression Scaling: speculative decoding, MoE, distillation, quantization

**Papers.** Yaniv Leviathan, Matan Kalman, Yossi Matias — *Fast Inference from Transformers via Speculative Decoding* — arXiv:2211.17192 — Google Research — 2023. William Fedus, Barret Zoph, Noam Shazeer — *Switch Transformer: Scaling to Trillion Parameter Models with Simple and Efficient Sparsity* — arXiv:2101.03961 — Google Brain — 2021. Albert Q. Jiang, Alexandre Sablayrolles, Antoine Roux et al. — *Mixtral of Experts* — arXiv:2401.04088 — Mistral AI — 2024. Geoffrey Hinton, Oriol Vinyals, Jeff Dean — *Distilling the Knowledge in a Neural Network* — arXiv:1503.02531 — 2015. Elias Frantar, Saleh Ashkboos, Torsten Hoefler, Dan Alistarh — *GPTQ: Accurate Post-Training Quantization for Generative Pre-trained Transformers* — arXiv:2210.17323 — IST Austria / ETH — 2022. Ji Lin, Jiaming Tang, Haotian Tang, Shang Yang, Wei-Ming Chen, Wei-Chen Wang, Guangxuan Xiao, Xingyu Dang, Chuang Gan, Song Han — *AWQ* — arXiv:2306.00978 — MIT — 2023. Companion: EAGLE-3 ([94](94-eagle3-spec-decoding.md)), DeepSeek-V3 (MoE 671B/37B active), R1-Distill series ([232](232-rl-on-reasoning-traces-scaling.md)).

**One-line definition.** A family of inference-side compression techniques — **speculative decoding** (verifier-accept-reject with a draft model; 2–4× wall-clock speedup), **mixture-of-experts** (sparse activation; 4–8× FLOPs reduction at fixed effective capacity), **distillation** (large teacher → small student; 5–10× parameter reduction at minor capability loss), **quantization** (FP16 → INT8/INT4/FP4; 2–4× memory + ~2× speed) — that **collectively shift the agent-era inference cost curves by 10–50×** at fixed task accuracy, making thinking-models ([232](232-rl-on-reasoning-traces-scaling.md)) and long-context ([234](234-context-length-scaling.md)) deployments economically viable.

## Why this paper matters (inference economics gates which agent harnesses are deployable)

The agent era's defining shift — thinking models that emit 5–30k tokens per query, multi-agent ensembles ([224](224-multi-agent-parallel-scaling.md)), long trajectories ([237](237-agent-trajectory-scaling.md)) with rich memory ([233](233-memory-scaling-for-agents.md)) — multiplies inference cost per task by 10–100× compared to single-shot LLM use. Without inference compression, these architectures would be prohibitively expensive for production. Speculative decoding, MoE, distillation, and quantization are the engineering substrate that makes the agent era affordable.

Speculative decoding (Leviathan-2023) is the canonical example. The insight: token-by-token autoregressive decoding underuses GPU parallelism because each token depends on the previous. A small **draft model** (cheap, fast) generates K candidate tokens; the **target model** (the actual large model) verifies all K in parallel via a single forward pass that scores them; tokens whose scores match are accepted, others rejected and re-drafted. Wall-clock speedup is **2–4×** with no quality loss — the target model's distribution is preserved exactly. EAGLE-3 ([94](94-eagle3-spec-decoding.md)) extends this to multi-step prediction with feature-level draft heads and reaches **3.5–6× speedup** on Llama-3 / Qwen-2.5 / DeepSeek-V3 — the speedup *grows* with model size because the draft-to-target FLOPs ratio improves.

Mixture of Experts (Switch Transformer 2021, Mixtral 2024, DeepSeek-V3 2024) reduces the active-parameter count per token while preserving total parameter count for capability. DeepSeek-V3 is 671 B total parameters with **37 B active per token** — inference FLOPs match a 37 B dense model while capability matches a 200 B+ dense model. The resulting unit economics — ~$0.30 per million input tokens vs GPT-4o's ~$5 per million — drove the open-source community's late-2024 / early-2025 capability convergence with proprietary frontier.

Distillation (Hinton 2015 origin; productized in DistilBERT 2019, GPT-4 → GPT-4o-mini 2024, R1 → R1-Distill 2025) trains a small student on a large teacher's outputs. The R1-Distill series ([232](232-rl-on-reasoning-traces-scaling.md)) is paradigmatic: R1's 671 B MoE reasoning capability transferred to Qwen-32B, Qwen-14B, Llama-70B students that fit on consumer or single-server hardware. R1-Distill-Qwen-32B reaches AIME 72.6 % — beating o1-preview at less than 5 % of o1's per-token cost.

Quantization (GPTQ 2022, AWQ 2023, FP8 across 2024, FP4 in 2025) reduces precision of model weights from FP16 to INT8, INT4, or even FP4. With careful calibration, INT4 quantization preserves 99 %+ of FP16 capability on most benchmarks while halving memory and speeding inference ~2×. FP4 (NVIDIA Blackwell) and aggressive quantization-aware training push further.

Take this evidence seriously and three things change. **First**, you understand that **model selection is now a multi-axis decision** — base model, MoE active-fraction, distillation tier, quantization level — each with its own cost-quality trade-off. **Second**, you architect the inference path with **speculative decoding by default** for thinking-model deployments — without it, long-trace inference is bottlenecked by autoregressive serial latency. **Third**, you adopt **distillation pipelines** as a first-class production discipline: train the largest thinking model your budget allows (or use a published one), distill to deployment-size, quantize for serving — the three-stage path to production-grade agent inference.

## Problem it solves (deploy thinking, long-context, multi-agent harnesses at affordable per-task cost)

1. **Autoregressive decoding is sequentially bottlenecked.** Each token forward pass takes ~10–50 ms on H100 for 70B; a 10K-token thinking trace takes 100–500s. Speculative decoding parallelizes draft-then-verify.
2. **Dense-model FLOPs dominate inference cost.** A 70B dense model at 32K context costs more per query than most production budgets allow. MoE with sparse activation reduces by 4–8×.
3. **Frontier models do not fit on consumer hardware.** GPT-4 / Claude / Gemini frontier weights are gated and large. Distilled open students (Qwen-32B, Llama-70B, Phi-4) approach frontier capability at deployable scale.
4. **FP16 memory is a binding constraint.** A 70B FP16 model is 140 GB; INT4 is 35 GB, fits on a single A100 80 GB with KV cache headroom.
5. **Long-context inference scales linearly with prefill.** Without prefix caching and KV-cache-aware MoE, long-context is prohibitively expensive.
6. **Thinking models multiply inference cost by 10–30×.** Without compression, R1 / o1 / extended-thinking deployments are not economically viable for high-volume use.

## Core idea in one paragraph

Inference cost is the binding constraint on agent-era architectures. Four orthogonal compression techniques — **speculative decoding** (parallelize autoregressive generation via draft-and-verify), **mixture of experts** (sparse activation: route each token through a small subset of experts; total parameters large but per-token FLOPs small), **distillation** (transfer capability from a large teacher to a small student via SFT on teacher outputs or feature alignment), **quantization** (reduce numerical precision with calibration to preserve quality) — compose multiplicatively. A 70B FP16 dense model running at 50 tokens/sec on H100 costs roughly $0.005 per query of 1K tokens. Apply speculative decoding (3× speed) → $0.0017. Switch to a 70B MoE with 14B active (5× FLOPs reduction) → $0.0003. Distill to 32B (2× faster, mild quality loss) → $0.0002. Quantize to INT4 (2× speed) → $0.0001. The composed factor is **50× cheaper** at modest quality loss; for thinking models with 30k-token outputs, the same factor applies and turns a $0.15 query into $0.003. This is what enables o1-class reasoning, MoA-style multi-agent ([224](224-multi-agent-parallel-scaling.md)), long-context ([234](234-context-length-scaling.md)) trajectories ([237](237-agent-trajectory-scaling.md)) to ship to production. Each technique has its own scaling law in (compression factor, quality retention), each composes with the others.

## Mechanism (step by step)

### (a) Speculative decoding — draft-and-verify

Leviathan-2023 §2: at each step, run a small **draft model** Q to generate K candidate tokens autoregressively (cheap because Q is small). Then forward-pass the **target model** P over the K candidates **in parallel** (a single batched call). Compute acceptance probabilities `p(t_i)/q(t_i)`; accept tokens up to the first rejection; sample a replacement at the rejection point from the residual distribution. The output distribution is identical to the target's distribution (Theorem 1) — no quality loss.

Speedup factor depends on:

- **Draft acceptance rate α.** Higher α → more accepted tokens per verification call → more speedup.
- **Draft-to-target FLOPs ratio.** Smaller draft → cheaper per-step cost.
- **Verification step parallelism.** Single forward pass scores K tokens.

Empirical: Llama-3-70B target + Llama-3-8B draft achieves α ≈ 0.7, 3× wall-clock speedup. EAGLE-3 ([94](94-eagle3-spec-decoding.md)) uses feature-level draft heads (no separate draft model) and multi-step look-ahead; 3.5–6× on Llama-3 / Qwen-2.5.

### (b) Mixture of Experts — sparse activation

A standard transformer layer has a feed-forward (FFN) block applied to every token. MoE replaces the FFN with **N expert FFNs** plus a **router** that selects top-K (typically K=1 or K=2) experts per token.

- **Total params:** ~N × dense-FFN (large; 671 B for DeepSeek-V3).
- **Active params per token:** ~K × dense-FFN + router (small; 37 B for V3 with K=8 of 256 experts).
- **Compute per token:** ~K × dense-FFN — a fraction of the dense equivalent.

Trade-offs:

- Memory still scales with total params (must store all experts).
- Routing introduces load balancing concerns; auxiliary losses keep experts utilized.
- Capability per active param is similar to dense; capability per total param is better.

Empirical: Mixtral-8x22B (141 B total / 39 B active) matches GPT-3.5 / Llama-2-70B at lower inference cost. DeepSeek-V3 (671 B / 37 B active) matches GPT-4-class at ~10× less inference FLOPs.

### (c) Distillation — knowledge transfer to a smaller student

Hinton-2015: train student on teacher's *softmax outputs* (soft labels) rather than hard labels. Student loss is KL(student | teacher) at temperature T, plus optional cross-entropy on hard labels.

Variants:

- **Response distillation.** Generate teacher outputs on a curated prompt set; SFT student on (prompt, teacher-output) pairs. R1-Distill series ([232](232-rl-on-reasoning-traces-scaling.md)).
- **Feature distillation.** Match intermediate hidden states; preserves more capability but requires architectural compatibility.
- **DPO/RPO distillation.** Use teacher's preference rankings to train student via direct preference optimization.

Empirical retention: a 32B student of a 671B teacher typically retains 85–95 % of teacher capability on benchmarks; often *exceeds* the teacher on simple tasks where distillation acts as a regularizer.

### (d) Quantization — reduce precision

GPTQ-2022 and AWQ-2023 are post-training quantization methods that calibrate weights at INT4 precision while preserving FP16-equivalent activations.

- **GPTQ.** Layer-by-layer weight quantization minimizing reconstruction error on a calibration set; 2nd-order info via Hessian.
- **AWQ.** Activation-aware: weights paired with high-magnitude activations preserved at higher precision.
- **FP8 / FP4 native.** NVIDIA H100 native FP8; Blackwell native FP4. Hardware-supported precision saves both memory and FLOPs.

Empirical: INT4 with GPTQ/AWQ preserves 99 %+ of FP16 capability on Llama-2-70B, MMLU; FP8 is essentially lossless; FP4 has 1–3 % degradation on hard benchmarks.

### (e) Composition — multiplicative effects

These four techniques compose roughly multiplicatively because they act on different dimensions:

- Speculative decoding: parallelizes time.
- MoE: reduces FLOPs per token.
- Distillation: reduces parameter count.
- Quantization: reduces bytes per parameter.

Composed: ~50× total inference cost reduction at <5 % capability loss is achievable on a frontier model.

### (f) Prefix caching for long context

Long-context economics ([234](234-context-length-scaling.md)) require KV-cache reuse: when multiple queries share a prefix (system prompt, RAG context, conversation history), cache the prefix's KV state and reuse across calls. Reduces prefill cost by the prefix-share ratio. Critical for production agent harnesses with stable prompts.

### (g) Continuous batching and paged attention

vLLM-style continuous batching: dynamically batch requests at varying decoding stages; paged attention manages KV-cache memory like virtual memory. Increases throughput 5–10× over naïve batching.

## Empirical results (table)

**Table 1 — Speculative decoding speedups (EAGLE-3, [94](94-eagle3-spec-decoding.md))**

| Target model | Method | Speedup | Quality loss |
|---|---|---:|---:|
| Llama-2-70B | Speculative (8B draft) | 2.8× | 0 |
| Llama-3-70B | EAGLE-2 | 3.5× | 0 |
| Llama-3-70B | EAGLE-3 | **5.6×** | 0 |
| Qwen-2.5-72B | EAGLE-3 | 4.8× | 0 |
| DeepSeek-V3 (MoE) | EAGLE-3 | 3.2× | 0 |

**Table 2 — MoE active-param efficiency**

| Model | Total params | Active params | Per-query FLOPs vs dense equiv |
|---|---:|---:|---:|
| Mixtral-8x7B | 47 B | 13 B | 0.28× of dense-47 B |
| Mixtral-8x22B | 141 B | 39 B | 0.28× of dense-141 B |
| DeepSeek-V3 | 671 B | 37 B | 0.055× of dense-671 B |
| Qwen-2.5-MoE-Plus | ~50 B | ~14 B | 0.28× of dense-50 B |

**Table 3 — Distillation retention (R1-Distill series, [232](232-rl-on-reasoning-traces-scaling.md))**

| Student (size) | Teacher | AIME-2024 | MATH-500 | Retention vs teacher |
|---|---|---:|---:|---:|
| Qwen-32B (distill) | R1 (671B/37B) | 72.6 % | 94.3 % | 91 % / 97 % |
| Qwen-14B (distill) | R1 | 69.7 % | 93.9 % | 87 % / 96 % |
| Qwen-7B (distill) | R1 | 55.5 % | 92.8 % | 70 % / 95 % |
| Llama-70B (distill) | R1 | 70.0 % | 94.5 % | 88 % / 97 % |

**Table 4 — Quantization quality retention (Llama-2-70B, MMLU)**

| Precision | MMLU | Memory | Speed (rel.) |
|---|---:|---:|---:|
| FP16 | 69.0 % | 140 GB | 1.0× |
| FP8 | 68.9 % | 70 GB | 1.7× |
| INT8 (GPTQ) | 68.5 % | 70 GB | 1.6× |
| INT4 (GPTQ) | 67.8 % | 35 GB | 2.4× |
| INT4 (AWQ) | 68.2 % | 35 GB | 2.4× |
| FP4 (Blackwell) | 67.2 % | 35 GB | ~3× |

**Table 5 — Composed compression factor (illustrative)**

| Stage | Cumulative factor | Quality loss |
|---|---:|---:|
| Baseline (Llama-3-70B FP16, dense) | 1.0× | 0 |
| + Speculative decoding (3.5×) | 3.5× | 0 |
| + Switch to MoE (5×) | 17.5× | small |
| + Distillation to 32B (2×) | 35× | ~3 % |
| + INT4 quantization (2×) | **70×** | ~5 % |

## Variants and ablations

- **Multi-step speculative.** Draft K tokens at depth D (tree-structured drafts); higher acceptance, more verification cost.
- **Self-speculation.** Use the model's own early layers as draft; no separate draft model.
- **Lookahead decoding.** Parallel generation via Jacobi-style fixed-point iteration; competitive with speculative.
- **Sparse + dense hybrid models.** Some layers MoE, some dense; balance flexibility and capability.
- **Continuous batching with priority.** Production schedulers prioritize latency-bound queries.
- **Paged attention (vLLM, SGLang).** KV-cache as virtual memory; reduces fragmentation.
- **Prompt compression.** LLMLingua, RAG-CEPE — compress long prompts via learned compression model.
- **Cascade routing.** Try cheap model first, escalate on uncertainty (FrugalGPT [86](86-frugalgpt.md), RouteLLM [87](87-routellm.md)).
- **Disaggregated prefill / decode.** Separate hardware for compute-bound prefill vs memory-bound decode (TensorRT-LLM, vLLM).
- **FlashAttention / FlashInfer.** IO-aware attention kernels; orthogonal to compression but co-essential for inference economics.

## Failure modes and limitations

- **Speculative draft mismatch.** Low draft acceptance (< 0.5) erases speedup; quality is preserved but cost is not reduced.
- **MoE load imbalance.** Without auxiliary balancing loss, some experts hot, others idle; capability degrades.
- **MoE memory remains large.** Total params still loaded into GPU memory; doesn't help memory-bound deployments.
- **Distillation has a ceiling.** Capability retention typically 85–95 %; the last 5–15 % requires the full teacher.
- **Quantization at INT3 / INT2 fails.** Below INT4, quality drops significantly; only specialized tasks tolerate.
- **Calibration set sensitivity.** GPTQ/AWQ depend on a representative calibration set; mismatched calibration → quality degradation in deployment.
- **Speculative cannot help on extreme prefill.** Speculative speeds up *decoding* (token generation); prefill of long context is unaffected.
- **MoE inference frameworks are immature.** vLLM, TensorRT-LLM only recently support MoE efficiently; older frameworks fall back to dense-equivalent.
- **Distillation can amplify teacher biases / errors.** Reasoning shortcuts in teacher transfer to student; verification at distillation time helps.
- **Stacking compressions amplifies error.** Each stage adds 1–3 % quality loss; composed stack at full compression can be 10 %+ worse than baseline.

## When to use, when not

**Adopt the full compression stack** for production agent deployments where per-query cost matters (high-volume, multi-turn, thinking-model, multi-agent); for edge / on-device deployment where memory and latency are tight; for cost-sensitive open-weights deployments matching closed-flagship economics. The strongest case is **thinking-model deployment** ([232](232-rl-on-reasoning-traces-scaling.md)) where 30K-token traces multiply inference cost.

**Skip aggressive compression** when quality is the binding constraint and a 3–5 % degradation is unacceptable (high-stakes safety, medical, legal); when deployment scale is small enough that compute cost is dominated by other factors (few queries / day); when the inference framework lacks MoE / quantization support; when the task is mostly long-prefill (long context, RAG with stable prompts) and the bottleneck is prefill not decode (speculative doesn't help; prefix caching does).

## Implications for harness engineering

- **Default to speculative decoding for thinking-model serving.** [232-rl-on-reasoning-traces-scaling](232-rl-on-reasoning-traces-scaling.md), [94-eagle3-spec-decoding](94-eagle3-spec-decoding.md) — without it, long traces are too slow.
- **Distillation pipeline as production discipline.** Train (or use) a large teacher; distill to deployment-size; quantize for serving. R1 → R1-Distill-Qwen-32B is the template.
- **MoE for high-volume inference at frontier capability.** DeepSeek-V3, Mixtral, Qwen-MoE — capability per inference dollar dominates dense at scale.
- **Quantization-aware finetuning.** [80-knowrl](80-knowrl.md), [156-heavyskill-rlvr](156-heavyskill-rlvr.md) — RLVR on quantized base preserves FP-trained capability post-deploy.
- **Cost router by compression tier.** [86-frugalgpt](86-frugalgpt.md), [87-routellm](87-routellm.md), [88-confidence-driven-router](88-confidence-driven-router.md), [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md) — easy queries → smallest distilled student; hard → full thinking model.
- **Prefix caching for long context.** [234-context-length-scaling](234-context-length-scaling.md) — essential when prompts share stable prefixes.
- **Continuous batching by default.** vLLM / SGLang / TensorRT-LLM — production agents have variable per-query workload.
- **HIR observability of compression metrics.** [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md) — log spec-decode acceptance rate, MoE expert utilization, quantization tier.
- **Cost-discipline gates per compression tier.** [p13-cost-discipline](../projects/polaris/docs/research/p13-cost-discipline.md), [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md) — tier-aware caps.
- **Skill engine sized for distilled student.** [04-skills](04-skills.md), [09-skill-engine](../projects/polaris/docs/blocks/09-skill-engine.md) — skills should fit on the deployed (distilled, quantized) model.
- **Subagent delegation runs on cheaper models.** [02-subagent-delegation](02-subagent-delegation.md), [18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md) — subagents need not match orchestrator's tier; mix-and-match by task.
- **Multi-agent ensembles benefit most from cheap inference.** [224-multi-agent-parallel-scaling](224-multi-agent-parallel-scaling.md) — K-agent ensembles only economical if per-agent cost is low.
- **Distill + RLVR composes.** [232-rl-on-reasoning-traces-scaling](232-rl-on-reasoning-traces-scaling.md), [156-heavyskill-rlvr](156-heavyskill-rlvr.md) — distill thinking-model traces into smaller student, then RLVR-finetune for domain.
- **Verifier infrastructure compresses similarly.** [97-qwen-prm](97-qwen-prm.md), [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md) — quantized PRMs are nearly indistinguishable from FP16 PRMs in BoN performance; ship them quantized.

**One-line takeaway for harness designers.** **Inference compression is the engineering discipline that makes the agent era affordable — speculative decoding, MoE, distillation, quantization compose multiplicatively to 10–50× cost reduction at modest quality loss, and shipping a thinking-model agent in production *requires* the full stack, not any single technique.**
