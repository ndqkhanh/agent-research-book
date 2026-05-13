# 116 — Adapter Tuning, LoRA, and PEFT for Agents: When Parametric Updates Belong in the Stack

**Sources.** Lakshmanan & Hapke, *Generative AI Design Patterns*, Pattern 15 (Adapter Tuning); Ozdemir, *Building Agentic AI*, Chapter 8 (Fine-Tuning AI for Calibrated Performance); Hu et al. 2021 (*LoRA: Low-Rank Adaptation of Large Language Models*); Houlsby et al. 2019 (*Parameter-Efficient Transfer Learning for NLP*); Liu et al. 2022 (*P-Tuning v2*); plus the QLoRA, IA³, and prefix-tuning literature.

**One-line definition.** Parameter-efficient fine-tuning (PEFT) — the family of methods that update a tiny percentage (0.1–3%) of an LLM's parameters via *low-rank adapters* (LoRA), *bottleneck layers*, or *learned prefixes* — is the right answer when an agent has a narrow, repeated subtask whose accuracy ceiling is bound by the base model's prior; for that subtask, a 7B-parameter LoRA adapter on top of an open-source base often beats GPT-4 with a clever prompt, at 1% of the inference cost.

## Why this matters

The default 2024 narrative was "fine-tuning is dead, prompt is enough." The default 2026 narrative is more nuanced: fine-tuning is *expensive and rarely necessary at the system level*, but PEFT for *narrow components* of an agent stack is often the highest-leverage move available. The Memento paper ([106-memento-paper-theory](106-memento-paper-theory.md)) makes the case that you can avoid fine-tuning the LLM. The Memento paper also makes the case that the *retriever* — a small dual-encoder — should be trained. That retriever is a parametric component. The full picture is: don't fine-tune the planner LLM, but consider fine-tuning the smaller specialist components.

For agent builders, PEFT changes the cost-benefit math. Full fine-tuning of a 70B-param model is gated by hardware, dataset size, and risk of catastrophic forgetting. LoRA fine-tuning of the same model is achievable on a single A100, with a few thousand examples, and the resulting adapter is < 100 MB. The unit economics shift from "fine-tuning is for big labs" to "fine-tuning is for any team with a labelled dataset and a weekend."

## Problem it solves

Five concrete situations where PEFT is the right tool:

1. **Style consistency.** The agent should output in your company's voice. Prompts approximate; LoRA on 1000 examples nails it.
2. **Domain adaptation.** Medical, legal, financial vocabulary the base model gets occasionally wrong. LoRA on domain text fixes it.
3. **Calibrated classification.** Specialist classifier (sentiment, intent, routing) where prompt-based GPT-4 reaches 88% but a LoRA-tuned 7B reaches 95%.
4. **Format reliability.** Even with constrained decoding, *which* fields the model fills with what content depends on prior. LoRA shapes the prior.
5. **Cost reduction.** Replace a $10/1M-token GPT-4 call with a $0.30/1M-token LoRA-tuned 7B call for a narrow task. 30× cost cut at parity quality.

Each is a *narrow* subproblem where the base model is the bottleneck.

## Core idea in one paragraph

Full fine-tuning updates all of a model's weights — billions of parameters — which is expensive, slow, and risks losing prior capabilities (catastrophic forgetting). PEFT methods update a tiny *additional* set of parameters while keeping the base weights frozen. **LoRA** adds low-rank matrices to the attention layers; the rank is typically 8–64, so the added parameter count is < 1% of the base. **Adapter layers** insert small bottleneck networks between transformer layers. **Prefix tuning** prepends learned token embeddings to each input. **IA³** scales activations element-wise. All of these add < 1% parameters, train in hours not days, produce small swappable artifacts (a 50–200MB file), and preserve the base model's general capabilities. For agent stacks, the right pattern is: fine-tune the *narrow specialist* components (classifiers, routers, format-shapers, domain extractors) with PEFT, keep the *general planner* at base-model quality, and stack the adapters at inference time per task.

## Mechanism (step by step)

### 1. LoRA — the dominant PEFT method

LoRA inserts trainable low-rank decomposition matrices into the attention layers:

```text
W' = W + ΔW
ΔW = B · A          where A ∈ R^(r×d), B ∈ R^(d×r), r ≪ d
```

The base weight matrix `W` stays frozen. Only `A` and `B` (the low-rank update) are trained. With *r* = 16, the parameter count of `ΔW` is ~1% of `W`. Training:
- Initialise `A` random Gaussian, `B` zero (so `ΔW = 0` at start, preserving base behavior).
- Backprop through `A` and `B` only; gradients on `W` are skipped.
- Save `A` and `B` as the adapter; total artifact size is small.

At inference: either fold `ΔW` into `W` (one-time cost; no per-call overhead) or keep them separate (allows hot-swapping adapters per request).

### 2. Adapter layers — the older alternative

Houlsby et al. 2019 inserts small bottleneck networks between transformer layers:

```text
h_out = h + Adapter(LayerNorm(h))
Adapter(x) = Down(x) → ReLU → Up(.)     where Down, Up are small (rank 64–256)
```

Less popular today than LoRA because LoRA folds into existing weights cleanly; adapters require additional layers at inference.

### 3. Prefix tuning, P-Tuning v2 — learned prompt vectors

Instead of modifying weights, prepend learned vectors to the input embeddings of every layer:

```text
[learned_prefix_1, ..., learned_prefix_n, input_embeddings...]
```

The learned prefix is the only thing trained. Small parameter count (n × layers × d), competitive with LoRA on some tasks, weaker on others. Useful when you cannot modify weights at all (closed-source models with prefix-tuning APIs).

### 4. QLoRA — fitting bigger models on smaller hardware

QLoRA (Dettmers et al. 2023) combines LoRA with 4-bit quantisation of the frozen base. Lets you LoRA-tune a 65B model on a single 48GB GPU. Quality is nearly identical to full-precision LoRA. The default for self-hosted PEFT in 2026.

### 5. The training pipeline

```text
1. Curate dataset:
   - 1000–10000 examples (input, desired output) pairs.
   - Quality matters more than quantity; 1000 high-quality > 10000 noisy.
2. Choose base model:
   - Open-source 7B–13B for cost (Llama 3, Mistral, Qwen).
   - Larger if specific capability requires it.
3. Choose PEFT method:
   - LoRA + QLoRA is the default.
4. Train:
   - 1–5 epochs typically.
   - LR 1e-4 to 5e-5; rank 8–32; alpha = 2 × rank.
   - Eval on held-out set to catch overfitting.
5. Evaluate:
   - On held-out test set with task-specific metrics ([115-evaluating-llm-systems](115-evaluating-llm-systems.md)).
   - On general-capability benchmarks to detect regression.
6. Deploy:
   - Fold adapter into base weights, OR
   - Keep separate for hot-swap.
```

A complete pipeline runs in 4–24 hours on a single GPU for most narrow tasks.

### 6. Adapter composition — multi-task PEFT

Multiple adapters can be loaded onto the same base model and switched per request:

```text
base_model + lora_classifier   →   classification task
base_model + lora_extractor    →   extraction task
base_model + lora_summariser   →   summarisation task
```

Each adapter is < 200MB; switching is < 1s. This is the production pattern for serving many specialist tasks from one base model. See vLLM's *multi-LoRA* serving feature.

### 7. When PEFT beats prompting

PEFT beats prompting reliably when:

- The task is **narrow** (single well-defined subtask).
- The base model's accuracy is **bound by prior**, not capacity.
- You have **labelled data** (1000+ examples).
- Inference cost matters (you'd save money by switching to a smaller fine-tuned model).
- Format reliability matters (LoRA shapes formatting more reliably than prompts).

PEFT loses to prompting when:

- The task is **broad** or undefined.
- You have **no labelled data**.
- The base model's prior is **already aligned** with the task.
- The deployment is **single-shot** (no amortisation of training cost).

## Empirical anchors

- **LoRA vs full fine-tuning quality.** Within 1–2% on most tasks; sometimes superior because LoRA is less prone to overfitting.
- **LoRA vs prompting on narrow tasks.** Often 5–15% absolute accuracy gain.
- **Cost reduction.** A 7B LoRA replacing a GPT-4 prompt for a narrow task is typically 10–30× cheaper.
- **Catastrophic-forgetting protection.** LoRA preserves base capabilities much better than full FT; multi-task adapters compose without interference.
- **Adapter size.** Typically 50–200MB; trivial to ship, swap, version.
- **Training time.** 4–24 hours on a single A100 for 7B LoRA with 5K examples.

## Variants and counter-arguments addressed

- **"You said don't fine-tune the LLM" ([106](106-memento-paper-theory.md)).** Memento avoids fine-tuning the *planner*. PEFT for narrow specialists is complementary, not contradictory.
- **"Why not full fine-tuning?"** Full FT is more expensive, slower, harder to revert, and prone to catastrophic forgetting. PEFT is a strictly better default.
- **"Why not RAG?"** RAG addresses knowledge gaps; PEFT addresses skill, style, and format. They compose: a PEFT-tuned model with a RAG retriever wins both axes.
- **"What about RLHF?"** RLHF is alignment, not narrow skill. Different problem; different methods.
- **"Open-source models can't match GPT-4."** Often true on broad tasks, *not* true on narrow tasks where 7B LoRA can beat GPT-4 prompting.
- **"Won't the adapter become stale?"** As the base model updates, you re-train the adapter on the new base. Re-training is cheap; the dataset is the durable asset.

## Failure modes and limitations

1. **Insufficient data.** < 500 examples and PEFT under-trains; results worse than prompting. Hard floor on training set size.
2. **Overfitting on small sets.** Tiny datasets + many epochs = memorisation. Use early stopping on held-out validation.
3. **Catastrophic-skill loss.** Aggressive LoRA training can degrade general capabilities. Eval on broad benchmarks, not just task-specific.
4. **Wrong rank.** Rank too small underfits; too large overfits. Default to *r* = 16; sweep on a held-out set.
5. **Adapter staleness on base updates.** Adapter is bound to a specific base; major version updates require re-training.
6. **Evaluation gap.** Training reduces train-set loss; what matters is test-set quality. The eval-set discipline from [115-evaluating-llm-systems](115-evaluating-llm-systems.md) applies here especially.
7. **Composition pitfalls.** Multiple adapters at once can interfere; test combinations, not just individuals.
8. **Deployment complexity.** Multi-LoRA serving is harder than single-model serving; not every inference framework supports it.

## When to use, when not

**Use PEFT when** the task is narrow, you have ≥1000 labelled examples, inference cost matters, and a small specialist model can plausibly do the job. Common fits: classifiers, routers, extractors, format-shapers, domain-specific QA, style adaptation.

**Don't use PEFT when** the task is broad, labelled data is scarce, the base model already handles it well, or the deployment is one-shot (no amortisation).

**Combine PEFT with prompting and RAG** rather than choosing among them. The full stack is: base model + LoRA adapter + retrieved context + structured prompt.

## Implications for harness engineering

- **Inventory your specialist subtasks.** Which subagents in your harness are narrow and high-volume? Those are PEFT candidates.
- **Build a labelling pipeline before training.** The dataset is the durable asset; collect it from production logs with reward signals (the same data feeds [107-memento-cbr-memory](107-memento-cbr-memory.md) and PEFT).
- **Default to QLoRA + 7B base.** Cheapest path; usually sufficient.
- **Version adapters with the base they target.** `llama3-8b-router-v2.lora` is more useful than `router.lora`.
- **Use multi-LoRA serving.** vLLM and friends let you load many adapters on one base. One GPU, many specialists.
- **Pair PEFT with constrained decoding** ([112-constrained-decoding](112-constrained-decoding.md)). LoRA shapes the content; constraints lock the format.
- **Eval before deploy.** Same offline + online + human eval discipline as [115-evaluating-llm-systems](115-evaluating-llm-systems.md). Adapters can regress; treat them as production ML artifacts.
- **Don't fine-tune the planner.** [106](106-memento-paper-theory.md) and [109-memento-results-and-harness](109-memento-results-and-harness.md) make the case; PEFT applies to *specialist* components, not the broad reasoning model.

The one-sentence takeaway: **PEFT changes the cost-benefit of fine-tuning — narrow specialist subagents in an agent harness are often best served by a small base + a small LoRA adapter rather than a clever prompt on a frontier model.**

## See also

- [80-knowrl](80-knowrl.md) — knowledge-aware RL for factuality, complementary to PEFT for skill.
- [93-dspy](93-dspy.md) — DSPy compiles narrow LLM steps; PEFT is the alternative when you have data.
- [106-memento-paper-theory](106-memento-paper-theory.md), [107-memento-cbr-memory](107-memento-cbr-memory.md) — the case-based-reasoning alternative that avoids fine-tuning entirely.
- [115-evaluating-llm-systems](115-evaluating-llm-systems.md) — the eval discipline that makes PEFT decisions data-driven.
- [117-small-language-models](117-small-language-models.md) — SLMs are common PEFT targets.
- [119-agent-ready-llm-selection](119-agent-ready-llm-selection.md) — LLM choice and PEFT strategy interact.
