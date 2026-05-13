# 134 — Semantic Indexing & Index-Aware Retrieval Patterns: Beyond Cosine — Hierarchical, Hybrid, Learned

**Sources.** Lakshmanan & Hapke, *Generative AI Design Patterns*, Patterns 7–9 (Semantic Indexing, Indexing at Scale, Index-Aware Retrieval); Kar, *Building Multimodal Generative AI*, Chapter 10 (Retrieval Optimization for Multimodal GenAI); the IR literature (BM25, ColBERT, SPLADE, dense passage retrieval); plus modern hybrid search systems (Weaviate, Qdrant, Elastic + dense).

**One-line definition.** Semantic indexing goes beyond "embed everything in a single vector index" to a *hierarchy of indexing strategies* — chunked vector, sentence-level vector, summary-level vector, lexical (BM25), structured-metadata, learned-sparse — and *index-aware retrieval* picks the right index (or combines several) per query type, yielding 20–40% precision lift over single-strategy baselines and being the foundational layer that reranking ([133-reranking-for-agentic-retrieval](133-reranking-for-agentic-retrieval.md)) and grounding ([135-trustworthy-generation](135-trustworthy-generation.md)) operate on top of.

## Why this matters

The default 2024 RAG indexing was: split documents into 500-token chunks, embed with one model, index in one store, retrieve by cosine similarity. This is a one-line solution that works for first prototypes and breaks at production: long documents lose structure, short queries don't match long chunks, lexical exact-match queries fail to find documents containing the literal term, metadata-heavy queries (filter by date, type, author) get no help from the index.

Production indexing in 2026 is *layered*: multiple indexes, multiple chunk granularities, hybrid sparse+dense, structured metadata. The retriever is *index-aware*: it picks the right index per query and combines results. Done well, this is the foundation under which reranking and grounding deliver their lifts; done poorly, even the best reranker can't recover lost recall.

For agent builders, semantic indexing is the unfashionable engineering work that makes the rest of RAG possible. It is also the layer where most production RAG quality comes from.

## Problem it solves

Six concrete indexing failures:

1. **Lost structure.** Long document split into 500-token chunks; chunks lose section context.
2. **Granularity mismatch.** Query "Who founded the company?" matches a sentence; chunk-level index returns whole sections.
3. **Exact-match miss.** Query "ICD-10 code A09.0" doesn't embed similar to its description; vector misses, BM25 finds.
4. **Metadata-blind.** Query "policy from 2024 about X"; date filter would narrow to 100 docs; vector ignores it.
5. **Single-shot embedding.** Document fits one chunk; some queries need section-level, some sentence-level.
6. **Embedding-model lock.** Re-indexing on model change is painful; multi-version indexes solve it.

Each is a structural limitation of single-strategy indexing.

## Core idea in one paragraph

Build *multiple indexes* over the same corpus, each tuned to a different query type. **Chunked vector** (500–1000 tokens) for general semantic retrieval. **Sentence-level vector** for fine-grained Q&A. **Summary-level vector** for "aboutness" queries (per-document or per-cluster summaries; this is GraphRAG's community summary). **Lexical (BM25 / SPLADE)** for exact-match and rare-term queries. **Structured metadata index** (SQL or specialised) for date/type/author/tag filters. **Learned-sparse** (SPLADE, ColBERT) for hybrid dense+sparse semantics. At query time, *route* the query to the right index(es), combine results, then rerank ([133-reranking-for-agentic-retrieval](133-reranking-for-agentic-retrieval.md)). The cost is multiple indexes to maintain; the benefit is recall and precision that no single strategy can match. The right index portfolio depends on your corpus and query distribution; profile both before committing.

## Mechanism (step by step)

### 1. The indexing portfolio

A production RAG system typically maintains:

| Index | Granularity | Strength |
|---|---|---|
| **Chunked vector** | 500–1000 tokens | General semantic retrieval |
| **Sentence vector** | 1–3 sentences | Fine-grained Q&A |
| **Summary vector** | per-doc or per-cluster | Aboutness queries |
| **BM25 / lexical** | per-chunk or per-doc | Exact-match, rare terms |
| **SPLADE / ColBERT** | per-chunk | Learned sparse + dense |
| **Metadata index** | per-doc | Filter by date, type, author, tag |

A query rarely touches all six; the routing decides which.

### 2. Chunking strategies

Chunking is where semantic indexing succeeds or fails:

- **Fixed-size token chunks.** Simple; often suboptimal because chunk boundaries cut sentences/sections.
- **Sentence-bounded chunks.** Don't split mid-sentence; respect natural boundaries.
- **Section-aware chunks.** Use document structure (headings, paragraphs) to set boundaries.
- **Recursive chunking.** Hierarchical: doc → sections → paragraphs → sentences; index multiple levels.
- **Overlap.** Adjacent chunks share 10–20% to preserve cross-chunk context.

For most production systems: **section-aware with overlap**, recursing for documents above a length threshold.

### 3. Multi-granularity indexing

Index the same corpus at multiple granularities:

```text
Document
   ├── chunk-level (500 tokens) → vector index A
   ├── sentence-level             → vector index B
   └── summary-level              → vector index C
```

Per query:
- "What is X?" → summary index first (aboutness).
- "How does X happen?" → chunk index (procedural detail).
- "What did Alice say about X?" → sentence index (specific attribution).

A learned router (small classifier) decides which index to query.

### 4. Hybrid sparse + dense — the dominant pattern

Combine BM25 (sparse, lexical) and vector (dense, semantic) retrieval:

```text
[query]
   ↓
   ├──→ BM25: top-K_lex
   └──→ vector: top-K_sem
   ↓
[fusion: RRF or weighted]
   ↓
top-K combined
```

BM25 catches exact matches (proper nouns, codes, rare terms); vector catches semantic similarity. Fusion (Reciprocal Rank Fusion) combines them.

Open-source: ElasticSearch / OpenSearch with dense plugin, Weaviate hybrid, Qdrant hybrid, Vespa.

### 5. SPLADE & ColBERT — learned sparse + late interaction

**SPLADE** (Formal et al. 2021): produces sparse embeddings (most dimensions zero); each non-zero dimension corresponds to a term. Combines lexical-like behavior with learned ranking.

**ColBERT** (Khattab & Zaharia 2020): late interaction — each query token's embedding interacts with each document token's embedding at scoring time, not encoding time. More expressive than dot-product; more expensive but tractable with optimisations (PLAID, BCAI).

Both lift retrieval quality 5–15% over vanilla vector + BM25.

### 6. Metadata index

Documents have structured properties: date, author, type, tags, jurisdiction. Index them in a relational store or specialised search engine (ElasticSearch).

Query rewriting: the query "policy from 2024 about pricing" parses to:
- Filter: `date >= 2024-01-01`, `tag = "pricing"`.
- Vector search: "policy about pricing".
- Combine: filter narrows; vector ranks within.

LLM-driven query parsing extracts filters; falls back to embedding-only if parse fails.

### 7. Index-aware retrieval — the routing decision

A small router (rules + LLM or learned classifier) picks the right index(es) per query:

```text
[query]
   ↓
[router]
   ├── exact match terms? → BM25
   ├── filter terms (date, type)? → metadata index + vector
   ├── aboutness query? → summary index
   ├── specific Q&A? → sentence index
   └── general → chunk index + hybrid sparse/dense
   ↓
[combined retrieval]
```

The router is light (rules cover 80%, LLM handles ambiguous 20%).

### 8. Index versioning

Indexes are versioned by:
- **Embedding model version.** Switching embedding models invalidates the index.
- **Chunking strategy version.** Re-chunking invalidates.
- **Schema version** for metadata.

Practical pattern: blue/green indexes (build new alongside old; switch when ready). See [125-system-level-production-patterns](125-system-level-production-patterns.md).

### 9. Index maintenance

- **Backfill on new corpus.** New docs flow in via CDC ([132-vector-cdc-pipelines](132-vector-cdc-pipelines.md)).
- **Re-embed on model change.** Replay historical CDC through new embedding worker.
- **Re-chunk on strategy change.** More expensive; usually only annually.
- **Compaction.** Vector indexes grow; periodic compaction maintains performance.

### 10. The complete production pattern

```text
[query]
   ↓
[router]
   ↓
[indexes: chunk, sentence, summary, BM25, SPLADE, metadata]
   ↓ (parallel queries to selected indexes)
[fusion: RRF over multi-index results]
   ↓ top-K (50–200)
[reranker: cross-encoder]
   ↓ top-N (10)
[LLM-rerank for prompt boundary]
   ↓ top-4
[planner LLM]
```

Each layer adds quality; the indexing portfolio is the foundation.

## Empirical anchors

- **Hybrid sparse + dense** lifts NDCG@10 by 5–15% over vector-only.
- **Multi-granularity indexing** lifts another 5–10% by matching query granularity.
- **SPLADE / ColBERT** lift another 5–10% over hybrid.
- **Metadata filtering** can drop candidate sets by 100×, dramatically improving precision when applicable.
- **Index portfolio cost** is 2–3× single-index storage.
- **Adoption** in production RAG is increasing; full multi-index portfolios at stage-3+ are common.

## Variants and counter-arguments addressed

- **"Just embed better."** Better embeddings help; they don't replace exact-match or filtered retrieval.
- **"Multi-index is over-engineering."** Until your single-index RAG is failing at production quality.
- **"BM25 is dead."** BM25 is alive and excellent at what it does. Hybrid is the answer.
- **"ColBERT is too expensive."** PLAID / late-interaction optimisations make it tractable in 2026.
- **"Vendor RAG handles all this."** Increasingly yes, but the principles apply regardless.

## Failure modes and limitations

1. **Chunking pathology.** Bad chunk boundaries destroy retrieval; iterate on the chunking.
2. **Index portfolio drift.** New indexes added; old ones never deprecated. Maintenance cost grows.
3. **Router brittleness.** Router mis-classifies queries; wrong index used. LLM fallback for confident-but-uncertain cases.
4. **Stale indexes.** Without CDC ([132-vector-cdc-pipelines](132-vector-cdc-pipelines.md)), staleness compounds.
5. **Metadata schema drift.** Source schema changes; metadata index breaks.
6. **Storage cost.** Multi-index multiplies storage; tier (hot vs cold) and compress.
7. **Query latency on parallel paths.** Hitting six indexes adds latency; budget.
8. **Eval gap.** Without per-index eval, you don't know which is contributing what.

## When to use, when not

**Use multi-index portfolio for** any production RAG at stage-3+ scale.

**Skip for** small static corpora; single-vector index suffices.

**Add hybrid (BM25 + vector) early.** It's high-leverage, low-engineering.

**Add learned sparse (SPLADE/ColBERT)** when domain-specific retrieval matters.

**Add metadata index when** structured filters appear in queries.

## Implications for harness engineering

- **Indexing pipeline as production infra.** Not application code; platform-level.
- **CDC-driven freshness.** [132-vector-cdc-pipelines](132-vector-cdc-pipelines.md). All indexes updated through one stream.
- **Per-index eval.** [115-evaluating-llm-systems](115-evaluating-llm-systems.md). Recall@K per index per query type.
- **Router as a small ML artifact.** Rules + LLM fallback; iterate on real queries.
- **Index versioning.** Blue/green; deprecate old explicitly.
- **Section-aware chunking by default.** Worth the engineering.
- **Metadata extraction at ingest.** Once at ingest cheaper than at query time.
- **Cache index queries aggressively.** Same query → same retrieval; cache hits dominate at scale.
- **Reranker on top.** [133-reranking-for-agentic-retrieval](133-reranking-for-agentic-retrieval.md) — the layer above this one.

The one-sentence takeaway: **production RAG indexing is a portfolio — chunked, sentence, summary, BM25, learned-sparse, metadata — with an index-aware router picking the right combination per query, and the foundational layer that makes reranking and grounding deliver their lifts.**

## See also

- [25-agentic-rag](25-agentic-rag.md) — agentic RAG.
- [120-rag-to-finetuning-spectrum](120-rag-to-finetuning-spectrum.md) — RAG positioning.
- [128-knowledge-graphs-as-substrate](128-knowledge-graphs-as-substrate.md), [129-kg-rag-hybrid-retrieval](129-kg-rag-hybrid-retrieval.md) — KG as another retrieval substrate.
- [132-vector-cdc-pipelines](132-vector-cdc-pipelines.md) — keeping indexes fresh.
- [133-reranking-for-agentic-retrieval](133-reranking-for-agentic-retrieval.md) — the layer above.
- [135-trustworthy-generation](135-trustworthy-generation.md) — citation and grounding.
- [115-evaluating-llm-systems](115-evaluating-llm-systems.md) — per-index evaluation.
- [136-multimodal-rag](136-multimodal-rag.md) — extending to multimodal corpora.
