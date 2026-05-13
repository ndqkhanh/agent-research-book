# 234 — Context-Length Scaling: nominal vs effective context for agents

**Papers.** Cheng-Ping Hsieh, Simeng Sun, Samuel Kriman, Shantanu Acharya, Dima Rekesh, Fei Jia, Yang Zhang, Boris Ginsburg — *RULER: What's the Real Context Size of Your Long-Context Language Models?* — arXiv:2404.06654 — NVIDIA — 2024. Nelson F. Liu, Kevin Lin, John Hewitt, Ashwin Paranjape, Michele Bevilacqua, Fabio Petroni, Percy Liang — *Lost in the Middle: How Language Models Use Long Contexts* — arXiv:2307.03172 — Stanford / NYU / UC Berkeley / Allen AI — 2023. Bowen Peng, Jeffrey Quesnelle, Honglu Fan, Enrico Shippole — *YaRN: Efficient Context Window Extension of Large Language Models* — arXiv:2309.00071 — 2023. Yiran Ding, Li Lyna Zhang, Chengruidong Zhang, Yuanyuan Xu, Ning Shang, Jiahang Xu, Fan Yang, Mao Yang — *LongRoPE* — arXiv:2402.13753 — Microsoft — 2024. Greg Kamradt — *Needle in a Haystack* — gkamradt/LLMTest_NeedleInAHaystack — 2023. Companion: Gemini 1.5/2/2.5 (1M+ context), Claude 200K, MiniMax-Text-01 (4M).

**One-line definition.** A line of work showing that **nominal context length** (e.g. 1M tokens for Gemini 2.5) and **effective context length** (the prefix at which retrieval-task accuracy stays > 80 %) **diverge sharply** — RULER reveals frontier models with claimed 32K–128K windows have effective lengths of only **~4K–32K** on multi-hop / multi-document tasks, while Lost-in-the-Middle quantifies the **U-shaped attention** problem (recall on middle-of-context items drops to ~50 % of edge-recall) — establishing context length as a **measured-not-claimed** scaling axis whose effective extent caps trajectory-length scaling ([237](237-agent-trajectory-scaling.md)) and memory-system bypass ([233](233-memory-scaling-for-agents.md)).

## Why this paper matters (context length is the most over-claimed and under-measured agent-era scaling axis)

Frontier-model marketing has emphasized nominal context length — "1M tokens!" "200K!" "4M!" — but the effective length at which the model genuinely uses information from the full prefix is often **5–10× smaller** than the claim. RULER, NIAH, and Lost-in-the-Middle established the *measurement infrastructure* that lets practitioners distinguish nominal from effective. For agent harnesses ([237](237-agent-trajectory-scaling.md)), this matters concretely: trajectory length is bounded by effective context, not nominal; memory architectures ([233](233-memory-scaling-for-agents.md)) only bypass context decay if retrieval lands relevant content in the *effective* portion; long-context-only deployments without subagent decomposition ([02-subagent-delegation](02-subagent-delegation.md), [18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md)) hit ceilings well before nominal limits.

RULER (Hsieh-2024) is the cleanest measurement framework. They constructed a synthetic-but-task-rich benchmark with 13 task categories — multi-key NIAH, multi-hop tracing, frequent-words extraction, common-words extraction, single/multi-keyvalue retrieval, variable tracking, question-answering — at context lengths 4K, 8K, 16K, 32K, 65K, 128K. Each model gets an **effective length** = the largest length at which it sustains > 80 % average accuracy across tasks. Headline results (RULER §5): Llama-3-70B claims 8K nominal but RULER-effective is **~4K**; Llama-3.1-70B claims 128K but RULER-effective is **~32K**; GPT-4 claims 128K, effective ~64K; Gemini-1.5-Pro claims 1M, effective ~128K (best in class at the time). The gap is not a measurement artefact — it is a real capability shortfall on tasks that require integrating information distributed across the prefix.

Lost in the Middle (Liu-2023) established the **U-shaped attention** finding: when relevant information is placed at different positions within a long context, model accuracy is highest when the information is at the start or end and lowest when it is in the middle. For 30-document multi-doc-QA on GPT-3.5-Turbo, accuracy dropped from ~75 % (relevant doc at position 1) to ~52 % (relevant doc at position 15) and rose back to ~63 % (position 30). The same shape held for MPT-30B-Instruct, Claude-1.3, and others — it is an architectural property of attention with positional encodings, not a single-model artefact.

YaRN, LongRoPE, position-interpolation, attention-sink work, ring-attention, and the long-context training literature collectively defined the **engineering recipes** for extending nominal context: rotary-position-embedding extrapolation, scaled-dot-product-attention numerical stability, KV-cache management, training-data curation for long sequences. Without these, nominal extension produces gibberish past the training horizon; with them, nominal extends but effective context still requires explicit measurement.

Take this evidence seriously and three things change. **First**, you measure RULER-effective context per model in your stack and **plan trajectory budgets ([237](237-agent-trajectory-scaling.md)) against effective, not nominal**. **Second**, you arrange information for U-shape avoidance — put critical context at start and end, not buried mid-prefix; use subagent decomposition or memory-tiered retrieval ([233](233-memory-scaling-for-agents.md)) when you cannot. **Third**, you understand that **long context does not replace memory** — long context is a flat read; memory is a structured store with learned retrieval; both scale, but on different curves with different unit economics.

## Problem it solves (nominal-vs-effective context, U-shaped attention, and the limits of long-context-only architectures)

1. **Nominal lengths over-claim capability.** Pre-RULER, vendor claims like "1M context" were treated at face value. RULER's measurement-first approach replaced marketing with engineering data.
2. **NIAH alone was insufficient.** A single needle in a haystack tests *retrieval* but not *integration*. RULER's 13-task suite measures multi-hop, multi-key, variable-tracking — the agentic use cases.
3. **Lost-in-the-Middle is architectural.** It is not a quirk of one model; it appears across all positional-encoding schemes tested. Implication: information arrangement within the prompt is itself a harness lever.
4. **Position-encoding extrapolation is non-trivial.** Naïvely extending RoPE past training-time positions fails. YaRN, LongRoPE, and PI provide engineering recipes; effective context requires both extrapolation *and* training data at long lengths.
5. **KV-cache cost scales linearly with context.** A 200K-token prefill on Llama-3-70B takes seconds and gigabytes; production must amortize via prefix caching ([235-inference-compression-scaling](235-inference-compression-scaling.md)).
6. **Long-context training data is scarce.** Most pretraining tokens are short documents. Long-context training requires synthetic concatenation, document-pair retrieval, or repository-level code; quality of these inputs caps quality of long-context behaviour.

## Core idea in one paragraph

A model has two context lengths: **nominal** (architecturally addressable, what the API accepts) and **effective** (the prefix length at which task accuracy stays above a threshold, e.g. 80 % on RULER's 13-task suite). The two diverge because (a) attention with positional encodings is intrinsically biased toward edges of the prefix — Lost-in-the-Middle's U-shape — making mid-prefix recall lossy; (b) training data at long lengths is scarce and synthetic, so the model under-learns long-prefix integration; (c) numerical stability of attention degrades at extreme lengths without engineering interventions (YaRN, LongRoPE, attention-sinks). The effective-vs-nominal gap is large in practice — a "128K model" often has an effective length of 32K on multi-hop tasks. The scaling curve `acc(prefix-length)` is roughly flat up to effective, then degrades steeply. For agent harnesses, this means **trajectory budget and memory-bypass are bounded by effective length**, not nominal — a 200-step trajectory consuming 100K context tokens cannot rely on the model attending to step-1 observations from step 200 on a "200K nominal" model unless it is also "200K effective." The mitigations are: subagent decomposition ([02-subagent-delegation](02-subagent-delegation.md)) to split context, memory tiering ([233-memory-scaling-for-agents](233-memory-scaling-for-agents.md)) to off-load and re-retrieve, U-shape-aware prompt layout (critical info at edges), and prefix caching to amortize prefill cost across calls. The right operational stance is **measure your model with RULER, plan against effective length, design memory and subagent architecture to bypass mid-context decay**.

## Mechanism (step by step)

### (a) RULER's 13-task benchmark

Hsieh-2024 constructed:

- **NIAH variants:** single-key (NIAH-S), multi-key (NIAH-MK1, MK2, MK3), multi-value (MV), multi-query (MQ).
- **Multi-hop:** trace a variable through K assignments scattered across the prefix.
- **Frequent-words / common-words:** count and extract.
- **QA:** synthetic question-answering with relevant context at varying positions.

Each task at lengths 4K, 8K, 16K, 32K, 65K, 128K. Effective length = max length where avg ≥ 80 %.

### (b) The Lost-in-the-Middle U-shape

Liu-2023 §3: place relevant document at positions 1, 5, 10, 15, 20, 25, 30 in 30-doc multi-doc QA. Accuracy:

- Position 1: ~75 %
- Position 5: ~64 %
- Position 10: ~58 %
- Position 15: **~52 %** (minimum)
- Position 20: ~58 %
- Position 25: ~61 %
- Position 30: ~63 %

The U-shape is robust across GPT-3.5, GPT-4, Claude, MPT, Llama. It is a property of the attention-with-positional-encoding family.

### (c) Nominal extension recipes (YaRN, LongRoPE, PI, NTK-aware)

- **Position Interpolation (PI, Chen 2023, arXiv:2306.15595):** Scale RoPE position indices down so a 32K window covers what was previously 4K; needs fine-tuning on long sequences.
- **NTK-aware (LocalLLaMA community).** Scale frequencies non-uniformly; preserves high-frequency components.
- **YaRN.** Combines NTK-aware scaling with dynamic-NTK and attention-temperature; works without finetuning at moderate extension.
- **LongRoPE.** Position-search optimization; finds non-uniform interpolation rules; extends beyond 2M tokens.
- **Attention sinks (Xiao 2023, *StreamingLLM*).** Always-attended initial tokens stabilize attention at extreme lengths.

### (d) Long-context training-data curation

Pretraining data is mostly short. Long-context training requires:

- **Concatenation.** Pack short docs into long sequences with explicit boundary markers.
- **Repository-level code.** GitHub repositories, code-with-imports yield natural long contexts.
- **Multi-document retrieval.** Pair related docs at retrieval time to form long inputs.
- **Long books and academic papers.** Scarce but high quality.

Quality of long-context training data caps long-context capability — a model trained on concatenated short docs handles long context worse than one trained on natively long docs.

### (e) KV-cache and prefill economics

A 200K-token prefill on Llama-3-70B:

- Prefill FLOPs: ~200K × 70B × 2 ≈ 2.8e16 FLOPs per prefill.
- Wall-clock: ~5–10s on A100.
- KV-cache memory: ~200K × 80 layers × 8K dim × 2 bytes ≈ 256 GB (without quantization).

This is why **prefix caching** (KV-cache reuse across calls with same prefix) and **MoE inference** ([235-inference-compression-scaling](235-inference-compression-scaling.md)) are essential at long context.

### (f) Subagent decomposition as context bypass

[02-subagent-delegation](02-subagent-delegation.md), [18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md) — split a long-trajectory into K parallel sub-trajectories with independent context windows; orchestrator integrates results. Avoids U-shape and effective-length limits at the cost of integration overhead.

### (g) Memory tiering as the orthogonal solution

[233-memory-scaling-for-agents](233-memory-scaling-for-agents.md) — instead of expanding context, off-load to persistent memory and retrieve relevant items at each step. Retrieval recall is the rate-limiter, not context length.

## Empirical results (table)

**Table 1 — RULER effective length vs nominal claim (Hsieh-2024 §5 selected models)**

| Model | Nominal | RULER effective (avg) | Gap |
|---|---:|---:|---:|
| Llama-3-70B (orig 8K) | 8K | 4K | 2× |
| Llama-3-70B-128K | 128K | 32K | 4× |
| Llama-3.1-70B | 128K | 32K | 4× |
| GPT-4-Turbo | 128K | ~64K | 2× |
| GPT-4o | 128K | ~64K | 2× |
| Gemini-1.5-Pro (May 2024) | 1M | ~128K | 8× |
| Claude-3-Opus | 200K | ~64K | 3× |
| Mistral-7B-32K | 32K | ~16K | 2× |

(Reconstructed from RULER §5 tables and follow-on community measurements.)

**Table 2 — Lost-in-the-Middle U-shape (Liu-2023 §3, GPT-3.5-Turbo, 30 docs)**

| Position | Multi-doc QA accuracy |
|---:|---:|
| 1 (top) | 75.7 % |
| 5 | 64.4 % |
| 10 | 58.0 % |
| 15 | 52.9 % |
| 20 | 58.0 % |
| 25 | 60.6 % |
| 30 (bottom) | 62.3 % |

**Table 3 — NIAH-multi-key fan-out (RULER §5)**

| Task | 4K | 8K | 16K | 32K | 65K | 128K |
|---|---:|---:|---:|---:|---:|---:|
| Llama-3-70B NIAH-MK3 | 99 % | 98 % | 95 % | 88 % | 72 % | 45 % |
| GPT-4-Turbo NIAH-MK3 | 99 % | 99 % | 97 % | 92 % | 84 % | 64 % |
| Gemini-1.5-Pro NIAH-MK3 | 99 % | 99 % | 99 % | 97 % | 92 % | 85 % |

**Table 4 — Long-context extension recipes (illustrative gains on RULER avg)**

| Recipe | Base model | Extended length | Effective length |
|---|---|---:|---:|
| PI (1.5× scale + finetune) | Llama-2-7B 4K | 32K | ~16K |
| YaRN (NTK + temp + finetune) | Llama-2-7B 4K | 64K | ~32K |
| LongRoPE | Llama-2-7B 4K | 2M | ~256K |

## Variants and ablations

- **Selective attention.** Sparse attention (Longformer, BigBird) reduces compute at long context; trade-off vs full-attention quality.
- **State-space models (Mamba, RWKV).** Linear-time sequence modelling; long-context strong by design but with different inductive biases.
- **Retrieval-augmented context.** Combine short context with on-demand retrieval ([233](233-memory-scaling-for-agents.md), [79-skill-rag](79-skill-rag.md)); often beats raw long-context at lower cost.
- **Context compression.** Compress mid-context content via summarization or learned compression; lossy but extends effective reach.
- **Hierarchical context.** Small recent window + summarized older window; Generative Agents' approach ([233](233-memory-scaling-for-agents.md)).
- **Position-encoding-free architectures.** ALiBi (Press 2022); avoids the explicit RoPE-extrapolation problem.
- **Continual long-context training.** Extend nominal in stages — 4K → 16K → 64K → 256K — each stage training data-curated.
- **Instruction-tuning for long context.** Long-context-specific instruction sets improve task transfer at long lengths.

## Failure modes and limitations

- **Nominal-effective gap is brittle.** Effective length depends on task type; a model strong on NIAH may be weak on multi-hop. Per-task measurement is necessary.
- **Mid-context decay is unfixable architecturally with RoPE.** Can be mitigated by training-data curation and attention-sinks but not eliminated.
- **KV-cache memory is the binding constraint** for long context in production. Prefix caching and MoE help; quantization helps further.
- **Prefill latency at long context is user-facing.** A 200K prefill takes seconds; users notice. Streaming and progressive prefill help.
- **Long-context training is expensive.** A 70B model at 256K context costs ~16× more memory than at 16K; training compute scales similarly.
- **Distillation does not preserve long context.** A long-context teacher distilled to short-context student loses capability; explicit long-context distillation needed.
- **Agentic long context conflates with memory.** Practitioners use "long context" and "memory" interchangeably; they are different scaling axes with different curves.
- **Eval contamination at long context is harder to detect.** Synthetic NIAH leaks; benchmark-specific.
- **Multi-modal long context multiplies cost.** Vision tokens at 1024-per-image add up; 1M-token Gemini contexts are dominated by vision when present.
- **Position-encoding interpolation can lose precision.** Subtle reasoning chains that depend on exact positions degrade.

## When to use, when not

**Push effective context length** when your tasks intrinsically require integrating information across long prefixes (long documents, multi-document QA, code repositories, long-running conversations); when you have budget for long-prefill latency and KV-cache memory; when subagent decomposition does not work (e.g., information across docs cannot be cleanly split); when retrieval-augmented short-context fails on your task type. The strongest case is **long-document analysis** and **repository-level code understanding**.

**Do not** rely solely on long context when retrieval-augmented short-context provides equivalent capability at lower cost; when latency constraints preclude long prefills; when effective length is well below nominal and the gap matters; when subagent decomposition or memory tiering is feasible; or when your prompt layout cannot avoid putting critical information mid-context. Retrieval-augmented short-context is usually cheaper at higher quality.

## Implications for harness engineering

- **Measure RULER-effective per model in your stack.** Don't trust nominal claims; benchmark on tasks like yours.
- **Trajectory budget ([237](237-agent-trajectory-scaling.md)) is bounded by effective context.** A 200-step trajectory using 100K tokens fails on a "128K nominal / 32K effective" model.
- **U-shape-aware prompt layout.** Place critical instructions and key context at start and end; avoid burying essential facts mid-prefix.
- **Subagent decomposition for context bypass.** [02-subagent-delegation](02-subagent-delegation.md), [18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md) — split context across sub-agents with independent windows.
- **Memory tiering as orthogonal lever.** [233-memory-scaling-for-agents](233-memory-scaling-for-agents.md), [05-reasoning-bank](../projects/polaris/docs/blocks/05-reasoning-bank.md), [81-reasoningbank](81-reasoningbank.md) — bypass context decay by retrieving rather than packing.
- **Prefix caching for long-context economics.** [235-inference-compression-scaling](235-inference-compression-scaling.md) — KV cache reuse amortizes prefill across calls; essential for repeat-prefix workloads.
- **Cost router with context-length tier.** [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md), [86-frugalgpt](86-frugalgpt.md), [87-routellm](87-routellm.md) — route by required context length and effective-length availability.
- **Context-aware difficulty router.** [88-confidence-driven-router](88-confidence-driven-router.md) — long-context tasks are inherently harder; route accordingly.
- **HIR observability for context usage.** [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md) — log per-step context size, KV-cache state, prefill latency.
- **Domain shells with bounded ACI context.** [11-domain-shell](../projects/polaris/docs/blocks/11-domain-shell.md), [17-mcp-adapter](../projects/polaris/docs/blocks/17-mcp-adapter.md) — design ACIs that paginate and summarize; do not dump.
- **Skill engine respects context budget.** [09-skill-engine](../projects/polaris/docs/blocks/09-skill-engine.md), [04-skills](04-skills.md) — skills should fit in a context tier; no mega-skill that fills context.
- **Bright-line context gates.** [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md) — escalate when context approaches effective limit, not nominal.
- **Reflexion and self-revision benefit from short context.** [90-reflexion-deep](90-reflexion-deep.md) — short-prefix revision often beats long-prefix continuation.
- **Multi-hop reasoning needs context-or-memory budget.** [199-search-r1-multi-hop](199-search-r1-multi-hop.md), [201-compositionality-gap-canon](201-compositionality-gap-canon.md), [202-multi-agent-multi-hop-reckoning-2026](202-multi-agent-multi-hop-reckoning-2026.md) — hop count vs effective context interplay.

**One-line takeaway for harness designers.** **Nominal context length is marketing; effective context length is engineering — measure with RULER, plan trajectory and memory architecture against effective-not-nominal, and treat subagent decomposition and memory tiering as the *bypass* mechanisms when the effective-length ceiling binds your workload.**
