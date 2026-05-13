# 276 — Local-First / Privacy-First Agents: consumer hardware, model selection, privacy primitives

**Anchors.** Ollama — https://ollama.com/. llama.cpp — https://github.com/ggerganov/llama.cpp. MLX (Apple) — https://github.com/ml-explore/mlx. vLLM — https://github.com/vllm-project/vllm. LM Studio — https://lmstudio.ai/. Open-weights frontier: Qwen-3 / Qwen-2.5, Llama-3 / Llama-4, DeepSeek-V3 / R1 + Distill series, Phi-4, Gemma-3, Mistral. Companions: [233-memory-scaling-for-agents](233-memory-scaling-for-agents.md) (Lyra's local-first memory), [253-tailscale-nats-mesh-for-distributed-agents](253-tailscale-nats-mesh-for-distributed-agents.md), [235-inference-compression-scaling](235-inference-compression-scaling.md), [232-rl-on-reasoning-traces-scaling](232-rl-on-reasoning-traces-scaling.md), [275-agent-finetuning-2026](275-agent-finetuning-2026.md), [86-frugalgpt](86-frugalgpt.md), [117-small-language-models](117-small-language-models.md).

**One-line definition.** A 2026 picture of **local-first / privacy-first agents** — running on consumer hardware (M-series Macs with 64-128GB unified memory comfortably hosting 32-70B models, RTX 4090/5090 + workstation, dedicated H100 home rigs) with **OSS frontier-distilled models** (R1-Distill-Qwen-32B / Llama-70B reaching o1-preview-class reasoning, Qwen-2.5-72B for general tasks, Phi-4 for small-fast, Gemma-3 for safety-tuned), **runtime stack** (ollama for ergonomics, llama.cpp for portability, MLX for Apple-silicon optimization, vLLM for serving), and **privacy primitives** (memory storage local, embeddings local via BGE-small, no cloud telemetry, on-device by default, opt-in cloud) — making **personal-assistant-class capability accessible at $0 marginal cost per query** for individuals running their own infrastructure, with the trade-off that capability per dollar lags frontier APIs by 6-12 months but **closes rapidly through distillation**.

## Why this matters (privacy + sovereignty + zero marginal cost are the local-first wins)

The 2024-2026 inflection: **OSS models reached the "good enough for personal use" threshold**. R1-Distill-Qwen-32B beats GPT-3.5 on most reasoning tasks and beats o1-preview on AIME; Qwen-2.5-72B is competitive with GPT-4o on instruction-following; Phi-4 is GPT-4o-mini class at 14B parameters. Combined with **consumer hardware capable of running them** — M-series Macs with 64-128GB unified memory, RTX 4090 24GB, dual-RTX-5090 home rigs, used H100s ~$15K for those who want them — and **runtime stacks that "just work"** (ollama, llama.cpp, MLX, LM Studio), **personal-agent deployments are now feasible at $0 marginal API cost** with the up-front hardware investment.

The wins for local-first are concrete: (1) **privacy** — agent memory ([233-memory-scaling-for-agents](233-memory-scaling-for-agents.md)) never leaves the device; (2) **sovereignty** — no vendor can deprecate, change pricing, or restrict your agent; (3) **zero marginal cost** — once hardware is amortized, queries are free; (4) **latency** — tens of ms first-token vs hundreds for cloud; (5) **offline capable** — works without network. Trade-offs: capability per dollar lags frontier APIs by 6-12 months; setup overhead is real; updates require local effort.

The Lyra-class personal agent is the canonical local-first deployment. Lyra's memory is SQLite + Chroma + .md files on the device; embeddings via BGE-small-en-v1.5 (33M params, CPU-fine); LLM via ollama or local API; no telemetry. With distillation closing the capability gap and consumer hardware getting cheaper, this deployment shape will dominate personal-agent use cases by 2027-2028.

## Core idea

Run the entire agent stack — model + memory + observability + tools — on hardware you control, with cloud opt-in only for explicit user actions. **Hardware tier** matches the **model tier**: M2 Mac 32GB → 7B-14B; M3/M4 Max 64-128GB → 32B-70B; RTX 4090 → 14B-32B; dual-5090 → 70B-110B; H100 → 70B-180B. **Model selection** prefers **distilled-from-thinking** (R1-Distill series) for reasoning, **Qwen-2.5-72B** for general, **Phi-4** for small-fast, with **Gemma-3** for safety-prioritized. **Runtime** picks ollama for ergonomics, llama.cpp for portability, MLX for Apple-silicon performance, vLLM for serving multiple sessions. **Privacy primitives**: SQLite + Chroma local, BGE-small embeddings local, redactor on memory writes, no telemetry by default, opt-in cloud for specific user-approved tools.

## Mechanism (step by step)

### (a) Hardware tier matrix (May 2026)

| Hardware | RAM/VRAM | Best model tier | Tokens/sec | Cost (USD) |
|---|---:|---|---:|---:|
| M2 Air 16GB | 16GB shared | Phi-4-mini 4B | 50-80 | $1500 |
| M3 Pro 36GB | 36GB shared | Llama-3-8B / Qwen-7B | 30-50 | $2500 |
| M4 Max 64GB | 64GB shared | Qwen-2.5-32B / R1-Distill-32B | 15-30 | $4000 |
| M4 Max 128GB | 128GB shared | Llama-3-70B / Qwen-2.5-72B | 8-15 | $6000 |
| RTX 4090 + workstation | 24GB VRAM | 14B INT4 / 7B FP16 | 60-120 | $4000 |
| Dual RTX 5090 | 64GB combined | 70B INT4 | 30-60 | $8000 |
| H100 80GB | 80GB | 70B FP16 / 110B INT8 | 80-150 | $15K used |
| Dual H100 | 160GB | 180B FP16 | 50-100 | $30K |

### (b) Model selection 2026 (open-weights)

| Use case | Recommended | Notes |
|---|---|---|
| Reasoning (math, code, logic) | R1-Distill-Qwen-32B / R1-Distill-Llama-70B | Beats o1-preview at AIME |
| General assistant | Qwen-2.5-72B-Instruct / Llama-4 | GPT-4o-class |
| Small-fast | Phi-4 (14B) / Qwen-2.5-7B | Sub-1s response |
| Safety-prioritized | Gemma-3-27B / Llama-Guard | Safety-tuned |
| Code-specific | Qwen-2.5-Coder-32B / DeepSeek-Coder-V2 | Beats GPT-4o-Coder on HumanEval |
| Multilingual | Qwen-2.5 / Llama-3 | Strong non-English |
| Embedding | BGE-small-en-v1.5 (33M) / Nomic-Embed | Local CPU |
| Vision | Llama-4-Vision / Qwen-2.5-VL | Multimodal |

### (c) Runtime stack

**Ollama** — most ergonomic; `ollama pull qwen2.5:72b` and `ollama run` works. OpenAI-compatible API. Auto-quantizes; GGUF backend. Easiest entry point.

**llama.cpp** — most portable; CPU + GPU + Apple Silicon + edge devices. GGUF format. Maximum performance on diverse hardware.

**MLX** — Apple-silicon-native; tensor library + LLM stack. Highest M-series performance.

**vLLM** — production serving; PagedAttention for high throughput; multi-tenant. Best for serving multiple users.

**LM Studio** — desktop GUI; ergonomic for non-engineers.

### (d) Privacy primitives — Lyra-class local-first stack

```
┌─────────────────────────────────────┐
│ User interaction (CLI / TUI / GUI)  │
├─────────────────────────────────────┤
│ Lyra agent loop                     │
│   ├── ollama (local LLM)            │
│   ├── BGE embeddings (local CPU)    │
│   └── tools (local MCP servers)     │
├─────────────────────────────────────┤
│ Memory (Lyra Block 07)              │
│   ├── SQLite + FTS5 (local)         │
│   ├── Chroma (local)                │
│   └── .md files (local)             │
├─────────────────────────────────────┤
│ Observability (Phoenix local)       │
├─────────────────────────────────────┤
│ Cloud (opt-in only, per action)     │
│   └── Tailscale + NATS hub (own VPS)│
└─────────────────────────────────────┘
```

No data leaves the device unless user explicitly approves a cloud action.

### (e) Distributed local-first via Tailscale + NATS

[253-tailscale-nats-mesh-for-distributed-agents](253-tailscale-nats-mesh-for-distributed-agents.md) — two devices on a tailnet with NATS leaf nodes give cross-device sync without cloud dependency. User controls all infrastructure.

### (f) Quantization for consumer hardware

INT4 (GPTQ / AWQ / GGUF Q4_K_M) is the practical default; ~50% memory, ~99% capability retention. INT8 / FP8 for higher quality at 2× memory. FP4 (Blackwell) on supported GPUs for highest density.

### (g) Hardware-aware routing

```python
def pick_model_for_task(task_difficulty, latency_requirement):
    if latency_requirement < 1s:
        return "phi-4-14b-int4"  # fast
    elif task_difficulty == "easy":
        return "qwen-2.5-7b-int4"
    elif task_difficulty == "medium":
        return "qwen-2.5-32b-int4"
    elif task_difficulty == "hard":
        return "r1-distill-qwen-32b-int4"  # reasoning
    elif task_difficulty == "very-hard":
        return "anthropic-cloud"  # opt-in cloud
```

Cost router ([15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md)) selects local-first; escalates to cloud only when warranted.

### (h) The capability-cost frontier 2026

Local-first capability lags cloud frontier by ~6-12 months, but the gap narrows. R1-Distill-Qwen-32B (Jan 2025) reaches AIME 72.6%, vs o1-preview's 56.7% (Sept 2024) and Claude 3.5 Sonnet's ~16% (March 2024). The frontier-distillation cycle is the engine.

## Empirical results (table — May 2026)

| Local model (M4 Max 128GB) | Cloud equivalent | Cost ratio | Latency |
|---|---|---:|---:|
| R1-Distill-Qwen-32B-INT4 | o1-preview | ~50× cheaper amortized | ~25 tok/s |
| Qwen-2.5-72B-INT4 | GPT-4o | ~30× cheaper | ~10 tok/s |
| Phi-4-INT4 | GPT-4o-mini | ~20× cheaper | ~80 tok/s |
| BGE-small | OpenAI ada-002 | Free | <50ms |

Up-front hardware ~$6K; amortized over 2 years ≈ $0.35/day.

## Variants and ablations

- **Cloud-LLM + local memory.** Best of both: frontier capability + private memory. Compromise on privacy depending on what's sent to cloud.
- **Encrypted cloud LLM.** Confidential computing for cloud (Apple Private Cloud Compute, Anthropic enterprise tiers).
- **Federated learning.** Train custom adapter locally; never share data.
- **Edge inference.** Phone, tablet, browser via WebGPU.
- **Hybrid serving.** Multiple local users via vLLM on home server.
- **Speculative decoding locally.** [235-inference-compression-scaling](235-inference-compression-scaling.md), [94-eagle3-spec-decoding](94-eagle3-spec-decoding.md) — small draft + large target.

## Failure modes

- **Capability gap with frontier.** Some tasks need cloud frontier.
- **Hardware obsolescence.** Models grow; hardware doesn't.
- **Update friction.** Manual model updates; less automated than cloud.
- **Multimodal lags.** Vision / audio / video local less mature than cloud.
- **Long-context lags.** Effective-context smaller than cloud frontier ([234-context-length-scaling](234-context-length-scaling.md)).
- **Battery / thermal limits.** Sustained inference on laptops drains battery.
- **Setup overhead.** Non-engineers need hand-holding.

## When to use, when not

**Adopt local-first** for **personal-assistant agents** (mentat-learn, lyra), **privacy-sensitive workflows** (health, finance, legal), **offline-required** scenarios, **cost-sensitive high-volume** workloads where amortized hardware beats API. Strongest cases: **Lyra-class personal agents**, **regulated-data workflows**, **research agents on confidential corpora**.

**Skip local-first** for **frontier-capability needs**, **low-volume sporadic use** (cloud cheaper amortized), **multimodal-heavy** workflows (vision/video lag), **users without technical setup tolerance**. Hybrid (cloud LLM + local memory) is often the right answer.

## Implications for harness engineering

- **Lyra-class deployment as canonical pattern.** [233-memory-scaling-for-agents](233-memory-scaling-for-agents.md), [Lyra Block 07](../projects/lyra/docs/blocks/07-memory-three-tier.md).
- **Ollama / llama.cpp / MLX / vLLM as runtime.** `harness_core/runtime/local/`.
- **BGE embeddings local by default.** [233-memory-scaling-for-agents](233-memory-scaling-for-agents.md).
- **Cost router prefers local-first.** [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md), [86-frugalgpt](86-frugalgpt.md), [88-confidence-driven-router](88-confidence-driven-router.md).
- **Tailscale + NATS for cross-device.** [253-tailscale-nats-mesh-for-distributed-agents](253-tailscale-nats-mesh-for-distributed-agents.md).
- **Distillation as deployment pipeline.** [235-inference-compression-scaling](235-inference-compression-scaling.md), [275-agent-finetuning-2026](275-agent-finetuning-2026.md).
- **Privacy primitives.** [233-memory-scaling-for-agents](233-memory-scaling-for-agents.md), [272-agent-compliance-and-audit](272-agent-compliance-and-audit.md).
- **Cloud opt-in per action.** [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md), [07-permission-bridge-research-edition](../projects/polaris/docs/blocks/07-permission-bridge-research-edition.md).
- **Hardware-aware routing.** [88-confidence-driven-router](88-confidence-driven-router.md), [87-routellm](87-routellm.md).
- **Small-language-models specialization.** [117-small-language-models](117-small-language-models.md).
- **Local observability via Phoenix.** [264-agent-observability-stack-2026](264-agent-observability-stack-2026.md) — runs locally.
- **Local eval suites.** [265-agent-evaluation-2026](265-agent-evaluation-2026.md).

**One-line takeaway.** **Local-first / privacy-first agents reached production-ready in 2026 — OSS distilled-thinking models on consumer hardware (M-series Mac with 64-128GB or RTX-class GPU rig) with ollama / llama.cpp / MLX runtimes deliver R1-Distill-Qwen-32B-class reasoning at zero marginal cost, with private SQLite + Chroma memory and BGE embeddings, opt-in cloud for specific actions only; the canonical Lyra-class deployment is the production template, lagging cloud frontier by 6-12 months but closing through distillation, and dominating cost + privacy + sovereignty for personal-assistant use cases.**
