# 33 — dnaHNet: Hierarchical Foundation Models as Agent Tools

**Definition.** dnaHNet (arXiv:2602.10603) is a tokenizer-free, autoregressive foundation model for genomic sequence learning. Its key novelty is a *differentiable dynamic chunking mechanism* that hierarchically compresses raw nucleotides into latent tokens, producing roughly a 3× speedup over Transformer baselines and matching or beating StripedHyena2 on zero-shot genomic benchmarks.

## Problem it solves

Most sequence foundation models (text, genomics, proteins) rely on *fixed vocabularies* — BPE tokens, k-mers, codons. For genomes these fragment biologically meaningful units (motifs, exon-intron boundaries) at arbitrary positions, hurting generalization. The natural fix — longer context to capture whole genes — runs into the quadratic cost of vanilla attention. dnaHNet's dynamic chunking handles both: no fixed vocabulary, and hierarchical compression yields quadratic FLOP reductions.

Beyond genomics, the paper is a reference point for the broader pattern: **specialized foundation models as agent tools**. An agent for biology (GPT-Rosalind, [30-gpt-rosalind-domain-specialized.md](30-gpt-rosalind-domain-specialized.md)) benefits from being able to call a native genomic model rather than prompting a text LLM to reason about nucleotide strings.

## Mechanism

Key components of dnaHNet:

1. **Tokenizer-free input.** Raw nucleotides (A, C, G, T, N) feed directly into the model — no BPE or k-mer segmentation.
2. **Dynamic chunking.** A differentiable mechanism learns where to segment the sequence; high-entropy / information-dense regions get smaller chunks, low-entropy regions larger.
3. **Hierarchical stack.** Chunks at one level become tokens at the next; the model operates over progressively coarser representations. This is where the quadratic savings come from.
4. **Autoregressive training on prokaryotic genomes.** Scalable pretraining data; the architecture is designed to scale further.
5. **Zero-shot evaluation on downstream tasks.** Protein variant fitness prediction, gene essentiality assessment — tasks that require genuine biological understanding, not surface statistics.

## Concrete pattern

For agent teams integrating a specialized foundation model as a tool:

```
Expose dnaHNet (or analogous domain FM) via MCP:
  tool: "genomic_embed"
    input:  { sequence: "ACGT...", task: "variant_fitness" }
    output: { embedding: [...], score: 0.87, confidence: 0.92 }
  tool: "genomic_autoregressive_score"
    input:  { context, query }
    output: log-likelihood, per-nt probabilities

Harness orchestrates:
  - text LLM decides what to ask / which sequence to look up.
  - dnaHNet returns domain-grounded scores.
  - text LLM synthesizes the clinician/researcher-facing answer.
```

This is the "agent as dispatcher, foundation model as calculator" pattern. The calculator's correctness is checkable against ground-truth benchmarks; the dispatcher handles context and synthesis.

## Variants & related techniques

- **GPT-Rosalind** ([30-gpt-rosalind-domain-specialized.md](30-gpt-rosalind-domain-specialized.md)) — composes domain models via tool use.
- **RadAgent** ([28-radagent-agentic-radiology.md](28-radagent-agentic-radiology.md)) — analogous agent-plus-tool decomposition for imaging.
- **StripedHyena2, HyenaDNA, Evo** — prior genomic foundation models that dnaHNet benchmarks against.
- **MCP** ([07-model-context-protocol.md](07-model-context-protocol.md)) — distribution format for specialist tools.
- **Tokenizer-free / byte-level language models** (ByT5, MEGABYTE) — same principle in text.

## Failure modes & anti-patterns

- **Letting the generalist model hallucinate biology.** If the agent doesn't route nucleotide reasoning to the specialist, you get textbook-plausible garbage. Fix: strong routing + refusal when the generalist is out of its depth.
- **Ignoring confidence.** The specialist returns a score; the agent uses it as ground truth. Fix: propagate calibrated confidence; require fallback for low confidence.
- **Mismatched training data.** Prokaryotic pretraining doesn't guarantee eukaryotic performance; domain of use must match domain of training.
- **Over-fitting to benchmarks.** Strong zero-shot benchmark performance does not guarantee clinical utility. Fix: clinical validation, domain evaluators.
- **Black-box integration.** The specialist model is as opaque as the generalist; trust is transferred, not earned. Fix: interpretability tools at the specialist boundary, not just the generalist.

## When to use (and when not to)

**Use** specialized foundation models as tools when:

- Generalists empirically underperform on the domain task.
- The task is genuinely sequence-structural (DNA, protein, music, code bytes).
- You can evaluate the specialist against domain ground truth.
- Latency and cost of an extra tool call are acceptable.

**Don't** use them when:

- The generalist already clears the bar — adding a specialist is complexity for marginal gain.
- The domain lacks stable training data or evaluation suites — the specialist's advantage can't be quantified.
- The harness can't maintain a separate model pipeline; operational cost outweighs capability gain.

## References

- arXiv:2602.10603 — "dnaHNet: A Scalable and Hierarchical Foundation Model for Genomic Sequence Learning". <https://arxiv.org/abs/2602.10603>
- Nguyen et al., "HyenaDNA". <https://arxiv.org/abs/2306.15794>
- Poli et al., "StripedHyena". <https://www.together.ai/blog/stripedhyena-7b>
- "Introducing GPT-Rosalind for life sciences research" (OpenAI). <https://openai.com/index/introducing-gpt-rosalind/>
- Yu et al., "MEGABYTE: Predicting Million-byte Sequences". <https://arxiv.org/abs/2305.07185>
