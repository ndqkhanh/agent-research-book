# 133 — Reranking for Agentic Retrieval: Cross-Encoder, Listwise, and Learned-from-Feedback Rerankers

**Sources.** Kar, *Building Multimodal Generative AI*, Chapter 9 (Reranking); Lakshmanan & Hapke, *Generative AI Design Patterns*, Pattern 10 (Postprocessing); Khattab et al. 2020 (ColBERT — late-interaction reranking); Nogueira & Cho 2019 (BERT-based passage reranking); plus Cohere's Rerank, Jina's reranker family, and the open-source cross-encoder ecosystem.

**One-line definition.** Reranking takes a top-K list of candidates from a fast retriever (vector / BM25 / hybrid) and re-orders them with a more powerful but slower scorer (cross-encoder, LLM, or learned model) to produce a higher-quality top-N for the agent — typically a 10–30% precision lift at the cost of one extra forward pass per candidate, and the highest-leverage RAG optimisation per dollar after constrained decoding.

## Why this matters

A retriever that returns 100 candidates and is right on the top-1 is the dream; in practice retrievers return 100 candidates with the *right* answer somewhere in the top 20. Reranking promotes the right answer from rank 17 to rank 2, where the LLM actually sees it. The lift is real and consistent; the engineering cost is small.

For agent builders, reranking is the second-largest reliability lever in RAG pipelines, after constrained decoding. Skipping it is the most common reason RAG quality lags expectation. Adding it is one library call away in most stacks.

This chapter is the playbook: what reranker to use, how to integrate it, when listwise reranking earns its complexity, and when LLM-as-reranker beats classical cross-encoders.

## Problem it solves

Five concrete RAG failures reranking addresses:

1. **Right answer in top-K but not top-3.** Retriever pulls relevant doc to position 12; LLM never sees it.
2. **Surface-similarity bias.** Retriever ranks by semantic similarity; misses passages whose relevance isn't surface-textual.
3. **Diversity-vs-relevance trade-off.** Retriever returns five near-duplicates; reranker can dedupe.
4. **Domain mismatch.** Generic embedding model doesn't capture domain ranking signal; reranker fine-tuned on domain fixes it.
5. **User-feedback ignored.** Click-through and relevance signals don't feed back into retrieval; learned-from-feedback rerankers close the loop.

Each is solved by adding a reranker between the retriever and the LLM.

## Core idea in one paragraph

Retrieval has two phases: a *fast* retriever that scores all corpus items at low compute (BM25, vector, hybrid), and a *slow* reranker that scores only the top-K candidates with a more expensive model. The reranker is typically a **cross-encoder** that takes (query, candidate) as a pair and outputs a relevance score — unlike the retriever's bi-encoder which encodes query and candidate independently. Modern variants extend this: **listwise rerankers** consider the candidate set jointly, producing a ranking rather than independent scores; **LLM-as-reranker** prompts an LLM to rate or re-rank the candidates; **learned-from-feedback** trains the reranker on user signals (clicks, dwell time, explicit ratings). The reranker is a small ML artifact (cross-encoders are typically 100M–500M params) that runs at acceptable latency for agent workloads. Adding it to a RAG pipeline is the highest-leverage optimisation per engineering dollar.

## Mechanism (step by step)

### 1. The two-stage retrieval architecture

```text
[query]
   ↓
[retriever: fast, scores all of corpus]
   ↓ top-K (typically 50–200)
[reranker: slow, scores K pairs (query, candidate)]
   ↓ top-N (typically 4–10)
[LLM: synthesises with N candidates as context]
```

The retriever is recall-optimised; the reranker is precision-optimised. Each stage does what it's good at.

### 2. Cross-encoder reranker — the dominant pattern

A cross-encoder takes two inputs together:

```text
Input:  [CLS] query [SEP] candidate [SEP]
Output: relevance_score (single scalar)
```

The query and candidate attend to each other through all layers of the transformer. This is more expressive than a bi-encoder (which encodes them separately and combines via dot-product) at the cost of compute: cross-encoder cost is O(K × forward-pass) where K is the number of candidates.

Trained on (query, relevant-candidate, irrelevant-candidate) triples with a contrastive or pointwise loss.

Open-source options: Cohere Rerank (API), Jina Reranker (API + open weights), `bge-reranker-v2` (open weights), `ms-marco-MiniLM` (open weights, smaller).

### 3. Listwise reranker — joint scoring

Listwise rerankers score the entire candidate list at once:

```text
Input:  [CLS] query [SEP] cand_1 [SEP] cand_2 [SEP] ... [SEP] cand_K
Output: ordering of candidates
```

Captures inter-candidate signals (e.g. "candidate B is more relevant than A given C is also relevant"). More expressive but more expensive; useful when relative ranking among similar candidates matters.

### 4. LLM-as-reranker

Prompt an LLM to rerank:

```text
Given query: "..."
Candidates:
  1. <text>
  2. <text>
  ...
Output: ranked list with rationales.
```

- Pro: leverages LLM reasoning; generalises across domains.
- Con: latency, cost.

Often used at the *final* prompt-boundary truncation: retrieved 8 with cross-encoder; LLM-rerank to pick top 4 for the planner. See [109-memento-results-and-harness](109-memento-results-and-harness.md) for the K=4 sweet spot.

### 5. Learned-from-feedback reranker

Production RAG systems collect user feedback:
- Click-through: did the user click this candidate?
- Dwell time: how long did they read?
- Explicit rating: thumbs up/down.

Aggregate these into (query, candidate, label) tuples; train a reranker on them. The reranker becomes domain-specific over time.

This is the [107-memento-cbr-memory](107-memento-cbr-memory.md) pattern (parametric retriever) applied to reranking. Works exceptionally well when feedback signal is reliable.

### 6. Latency considerations

Cross-encoder reranking adds latency:
- Per-candidate cost: ~10–100ms on cross-encoders (small to medium sized).
- For K = 50 candidates: 500ms–5s.
- Batched inference reduces this dramatically.
- GPU-hosted rerankers run K = 50 in 100–300ms.

For interactive agents: target reranker latency < 500ms. Pick model size accordingly.

### 7. Diversity-aware reranking

Reranking can deduplicate near-duplicates:

```text
1. Cross-encoder rank by relevance.
2. MMR (Maximal Marginal Relevance): iteratively pick the next candidate that maximises (relevance - λ × similarity-to-already-picked).
3. Final list balances relevance and diversity.
```

Useful when corpus has many near-duplicates (news articles on the same event, repeated forum posts).

### 8. Position-aware reranking

For agent prompts, position in the prompt matters (lost-in-the-middle effect). Position-aware rerankers:
- Score candidates by relevance.
- Place top-1 at start of context, top-2 at end (bookending).
- Mid-relevance candidates in the middle, where attention dispersion is fine.

Empirically: 5–10% improvement over naive top-K placement.

### 9. Evaluation

Reranker quality measured by:
- **NDCG@K**: normalised discounted cumulative gain at K.
- **MRR**: mean reciprocal rank of the relevant candidate.
- **Recall@K post-rerank**: did the relevant candidate make it into the top N?

Eval requires a labelled set of (query, candidate, relevance-label) triples.

### 10. Production deployment

```text
[retriever: vector + BM25 hybrid]
   ↓ top-K = 50
[reranker: cross-encoder (cohere-rerank or self-hosted)]
   ↓ top-N = 10
[LLM-rerank for final prompt-boundary trim] (optional)
   ↓ top-4
[planner LLM]
```

Three-stage retrieval. Each stage adds latency and quality.

## Empirical anchors

- **Cross-encoder reranking** lifts NDCG@10 by 10–30% over vector-only retrieval consistently.
- **LLM-as-reranker** lifts another 5–10% on top of cross-encoder, at higher cost.
- **Listwise rerankers** add modest gains over pointwise; complexity rarely justified for most workloads.
- **Self-hosted reranker** (BGE / Jina) at A100 cost: $0.001–0.01 per query; cheap.
- **Adoption** in production RAG is high in 2026; nearly universal at stage-3 maturity.
- **Latency cost**: 100–500ms typical; well within agent latency budgets.

## Variants and counter-arguments addressed

- **"Better embeddings remove the need for reranking."** Embeddings keep getting better; rerankers stay ahead because they use cross-attention. Both improvements stack.
- **"Reranking is redundant with hybrid retrieval."** Hybrid (BM25 + vector) helps recall; reranking helps precision. Different problems.
- **"Just use Cohere Rerank."** It's good. So is Jina, BGE. Self-hosted gives more control.
- **"LLM-as-reranker is overkill."** Often yes; sometimes the right tool when domain-specific reasoning is needed.
- **"Listwise is the future."** Modest gains; engineering complexity. Pointwise cross-encoder is the default.

## Failure modes and limitations

1. **Reranker mismatch.** Reranker trained on different domain than your corpus; degrades quality. Domain-tune.
2. **Latency spike on cold cache.** First request hits cold reranker; warm-up critical.
3. **Cost on high QPS.** At thousands of QPS, reranker compute is real money. Self-host or cache.
4. **Stale on model swap.** Reranker tuned on old retriever; needs re-tune when retriever changes.
5. **Bias amplification.** Reranker amplifies retriever's biases; eval for fairness.
6. **Over-trust.** Reranker score treated as ground truth; missing the recall floor (irrelevant in top-K won't recover).
7. **Re-rank cost > retrieval gain on small K.** For K = 4, reranker often doesn't help; for K ≥ 10, it does.
8. **Provider rate limits.** Cohere / Jina / OpenAI rerankers have quotas; budget.

## When to use, when not

**Always use a reranker** in production RAG. The lift is consistent; the cost is modest.

**Use cross-encoder by default**; LLM-reranker for the final prompt-boundary cut.

**Add learned-from-feedback** when production feedback is collected and reliable.

**Skip listwise rerankers** unless inter-candidate signals are demonstrably needed.

**Skip reranking for very small candidate sets** (K = 3 or smaller); not enough material to re-order.

## Implications for harness engineering

- **Reranker as a sidecar service.** Not in the agent code; in a service that the retrieval pipeline calls. See [125-system-level-production-patterns](125-system-level-production-patterns.md).
- **Self-host high-volume rerankers.** Cohere/Jina API for low volume; self-hosted for production scale.
- **Learned-from-feedback pipeline.** Click + dwell + rating → (query, candidate, label) → periodic retrain. Same plumbing as [107-memento-cbr-memory](107-memento-cbr-memory.md).
- **Position-aware placement.** After reranking, place candidates with bookending strategy.
- **Diversity / MMR.** For corpora with near-duplicates.
- **Eval the reranker independently.** NDCG, MRR; per-domain breakouts.
- **Cost dashboard.** Reranker spend per agent; budget.
- **Cache reranking results.** Same query × same candidates → same scores.

The one-sentence takeaway: **reranking is the highest-leverage RAG optimisation per dollar — cross-encoder by default, LLM-rerank at the prompt boundary, learned-from-feedback when production signal is reliable, always at production scale.**

## See also

- [25-agentic-rag](25-agentic-rag.md) — agentic RAG with self-critique.
- [120-rag-to-finetuning-spectrum](120-rag-to-finetuning-spectrum.md) — RAG positioning.
- [129-kg-rag-hybrid-retrieval](129-kg-rag-hybrid-retrieval.md), [134-semantic-indexing](134-semantic-indexing.md) — adjacent retrieval patterns.
- [107-memento-cbr-memory](107-memento-cbr-memory.md) — learned ranking generalised.
- [115-evaluating-llm-systems](115-evaluating-llm-systems.md) — eval for retrieval+rerank quality.
- [110-transformer-llm-architecture-for-agent-builders](110-transformer-llm-architecture-for-agent-builders.md) — lost-in-the-middle, positions matter.
- [135-trustworthy-generation](135-trustworthy-generation.md) — what comes after reranking.
