# 129 — KG + RAG Hybrid Retrieval: Combining Graph Traversal with Vector Similarity for Multi-Hop Reasoning

**Sources.** Devlin, *Building LLM Agents with RAG Knowledge Graphs*, Chapters 3–4; Microsoft GraphRAG paper (Edge et al. 2024); LlamaIndex's KG + vector hybrid patterns; Neo4j's GraphRAG documentation; plus the multi-hop QA literature (HotpotQA, MuSiQue benchmark results).

**One-line definition.** Hybrid retrieval combines vector similarity (for "documents semantically about X") and graph traversal (for "things related to X by typed edges") into a single retrieval step that fuses ranked candidates from both, yielding 30–50% improvements on multi-hop reasoning while keeping single-hop performance — and is the dominant retrieval pattern for production agents in regulated, enterprise, and research domains where relational queries are non-trivial.

## Why this matters

Vector RAG ([25-agentic-rag](25-agentic-rag.md), [120-rag-to-finetuning-spectrum](120-rag-to-finetuning-spectrum.md)) is a single-axis retrieval — semantic similarity over chunks. KG retrieval ([128-knowledge-graphs-as-substrate](128-knowledge-graphs-as-substrate.md)) is a single-axis retrieval — structural traversal over typed graphs. Each excels at one query type and degrades at the other. Hybrid retrieval combines both, using each where it wins, fusing results.

For agent builders in 2026, this is the dominant retrieval pattern in production. It outperforms vector-only on multi-hop tasks, outperforms graph-only on semantic similarity, and degrades gracefully when one substrate has a gap (the other compensates). The engineering cost is real (two indexes, two query paths, a fusion strategy) but the capability lift is large for relational workloads.

This chapter is the integration playbook: when to use, what the architecture looks like, how to fuse results, what failure modes to expect.

## Problem it solves

Five concrete capability gaps hybrid retrieval fills:

1. **Multi-hop reasoning.** "Find papers cited by papers Alice authored that Bob also cited." Pure vector misses; pure graph misses semantic context. Hybrid wins.
2. **Entity-anchored search.** "Documents discussing the impact of [specific person] on [specific project]." Vector retrieves loosely; graph anchors the entity precisely.
3. **Lineage + content.** "Show me the requirements doc and all downstream test specs that depend on it." Graph traverses lineage; vector finds related but un-linked docs.
4. **Provenance-grounded answers.** "What's our policy on X, and where does it say so?" Graph carries the policy hierarchy; vector retrieves the actual text.
5. **Sparse-graph fallback.** Where the KG has gaps, vector retrieval picks up; where vector retrieval fails on rare entities, graph anchors.

## Core idea in one paragraph

A hybrid retrieval system runs two queries in parallel: a **vector search** over embedded chunks for semantic similarity, and a **graph query** over the KG for structural relationships, both anchored on the user's question. The two result sets are then **fused** — by reciprocal rank fusion, by an LLM reranker, or by a learned fusion model — into a single ranked list of context items that the agent's planner consumes. Optionally, the graph query can include **community-summary retrieval** (per GraphRAG): pre-computed summaries of clusters in the graph, retrieved by relevance to the query. The complete retrieval result is heterogeneous (raw chunks + extracted entities + community summaries + traversal paths) and richer than either substrate alone. The fusion strategy and the community-summary investment are the two engineering decisions that make or break the pattern.

## Mechanism (step by step)

### 1. Architecture overview

```text
[user query]
   ├──→ [vector retrieval: embed query, top-K from chunk index]
   │       ↓
   │   [chunk results]
   │
   ├──→ [entity extraction: NER on query]
   │       ↓
   │   [entity-anchored graph traversal]
   │       ↓
   │   [graph results: paths + neighbour entities]
   │
   ├──→ [community summary retrieval] (GraphRAG)
   │       ↓
   │   [community summaries]
   │
   ↓
[fusion: combine all three into ranked context]
   ↓
[planner LLM: synthesise]
```

Three retrieval channels, one fusion step.

### 2. Vector retrieval — the semantic axis

Standard chunked-document RAG:
- Embed query.
- Top-K chunks by cosine similarity.
- Apply re-ranking ([133-reranking-for-agentic-retrieval](133-reranking-for-agentic-retrieval.md)).
- Return top-N final chunks with metadata (source, position).

Strengths: semantic coverage, rare-entity recall, content-relevance ranking.

### 3. Graph retrieval — the structural axis

Two substeps:

**Entity extraction**: NER (or LLM-driven) on the query identifies entities (people, organisations, products). Each entity is matched against the KG's canonical entity list (with disambiguation).

**Traversal**: from anchor entities, traverse edges by type and depth budget. Three traversal strategies:
- **Neighbourhood**: 1-hop or 2-hop neighbours of anchor entities.
- **Path search**: shortest paths between two anchor entities.
- **Pattern matching**: subgraphs matching a query template.

Returns: triples (entity, relationship, entity) plus the original entities' properties.

### 4. Community-summary retrieval (GraphRAG)

In GraphRAG (Edge et al. 2024):
- The KG is clustered into communities (Leiden algorithm or similar).
- Each community gets an LLM-generated summary describing what it's about.
- At query time, summaries are retrieved by relevance.
- The summary brings in *contextual* information about the broader graph region, not just the literal entities.

This step costs significant up-front compute (LLM-generated summaries × communities) but is essential for queries that need *aboutness* of a region.

### 5. Fusion strategies

The three result streams must be combined. Four fusion strategies:

**Reciprocal Rank Fusion (RRF)**: simple, robust default.
```text
score(item) = Σ 1 / (k + rank_in_stream_i)
```
With k = 60 typically. Combines disparate ranking systems without calibration.

**LLM-based reranking**: a cross-encoder or LLM scores each candidate's relevance to the query; pick top-N.
- Pro: handles diverse formats well.
- Con: latency, cost.

**Learned fusion**: train a small model on (query, candidate, label) tuples; learn the fusion.
- Pro: highest accuracy.
- Con: needs labelled data; like a parametric retriever ([107-memento-cbr-memory](107-memento-cbr-memory.md)).

**Heuristic weighting**: vector × α + graph × β + community × γ; tune α/β/γ on eval set.
- Pro: simple; explainable.
- Con: lower ceiling.

For most agent builders: **start with RRF; upgrade to LLM-reranking if quality demands**.

### 6. Tool integration patterns

Two integration approaches:

**Pattern A: hybrid retrieval as a tool.**
- The agent's tool list includes `hybrid_retrieve(query, top_k)`.
- The agent calls it like any other tool.
- The retrieval logic (vector + graph + fusion) is inside the tool.

**Pattern B: hybrid retrieval as the planner's context provider.**
- Before calling the planner, the harness calls hybrid retrieval.
- Retrieved context is prepended to the planner's prompt.
- The agent doesn't see retrieval as a tool; it sees pre-loaded context.

Pattern A is more flexible (agent can re-retrieve mid-task); pattern B is simpler. Most production: pattern B for the initial context, pattern A available as a tool for follow-up retrievals.

### 7. Multi-hop traversal budget

A key tunable: how deep to traverse the graph from anchor entities?

- **Hop 1**: direct neighbours. Cheap, narrow.
- **Hop 2**: neighbours of neighbours. Moderate cost; covers most relational queries.
- **Hop 3+**: combinatorial blow-up; needs cardinality-aware filtering.

For most queries: **hop 2 with cardinality filtering** (drop high-degree nodes from expansion).

### 8. Quality evaluation

Hybrid retrieval needs its own eval set:
- Multi-hop QA pairs.
- Entity-anchored queries.
- Lineage queries.
- Aboutness queries.

Per-channel metrics (vector recall, graph precision, summary relevance) and end-to-end metrics (final-answer accuracy after fusion).

### 9. Operational concerns

- **Two indexes**: vector store + graph DB. Both need updating when source data changes.
- **CDC pipeline**: changes flow to both ([132-vector-cdc-pipelines](132-vector-cdc-pipelines.md)).
- **Cost**: hybrid retrieval is 1.5–3× the cost of vector-only.
- **Latency**: parallel queries hide most of the cost; fusion adds <100ms.
- **Caching**: query result caching at the fusion level.

## Empirical anchors

- **30–50% multi-hop QA improvement** vs vector-only (GraphRAG paper).
- **Single-hop performance preserved** (within 1–2% of vector-only).
- **RRF as fusion baseline** within a few points of more complex strategies.
- **LLM-reranker fusion** adds another 5–10% over RRF at higher cost.
- **Community summaries** are the most expensive component; cache and amortise.
- **Adoption in regulated industries** is high; in consumer Q&A, lower.

## Variants and counter-arguments addressed

- **"Vector retrieval with better embeddings is enough."** It isn't, for multi-hop. Better embeddings raise the floor; they don't bridge the structural gap.
- **"GraphRAG is just KG + RAG with marketing."** The marketing is real; the engineering (community detection, summarisation, fusion) is also real.
- **"This is too complex for our team."** Start with vector-only; add graph when query analysis shows the need. Don't over-engineer day one.
- **"Aren't there hybrid vector engines that do this?"** Some (e.g. Weaviate's hybrid) do BM25+vector; that's not the same as graph+vector. Graph traversal is structurally different.
- **"What about BM25 + vector?"** That's a different hybrid (lexical + semantic). Some agents combine all three: BM25 + vector + graph.

## Failure modes and limitations

1. **Anchor-entity miss.** NER misses an entity in the query; graph retrieval contributes nothing. Pair NER with fuzzy-matching to KG.
2. **Disambiguation errors.** Anchor matches the wrong entity (Apple the fruit vs Apple Inc); downstream answers are wrong. Invest in disambiguation.
3. **Fusion miscalibration.** Vector and graph scores on incompatible scales; naive fusion biases toward one. Use RRF or learned fusion.
4. **Cardinality blow-up.** A high-degree anchor entity expands to thousands of neighbours; budget exceeds.
5. **Stale graph.** Graph data lags source; graph results are wrong while vector results are right.
6. **Cost surprise.** Two retrieval paths plus reranker plus LLM = 3–5× single-vector-RAG cost.
7. **Over-retrieval.** Too much context dilutes the planner; cap each channel's contribution.
8. **Eval gap.** Multi-hop eval sets are expensive to curate; without them, you're flying blind.

## When to use, when not

**Use hybrid retrieval when** queries are mixed (some semantic, some relational), or when relational queries are >30% of traffic.

**Use pure vector** for purely semantic Q&A workloads.

**Use pure graph** for highly structural workloads (data lineage, dependency graphs) where text content is secondary.

**Skip community summaries** if your KG is small or queries are entity-anchored rather than aboutness-driven.

## Implications for harness engineering

- **Build the eval set first.** Multi-hop and relational query coverage is the discipline. See [115-evaluating-llm-systems](115-evaluating-llm-systems.md).
- **Two indexes, one query path.** Hide the two-index complexity behind a single retrieval API; the agent shouldn't care.
- **CDC pipeline updates both.** [132-vector-cdc-pipelines](132-vector-cdc-pipelines.md). Both stay in sync with sources.
- **NER + canonicalisation as a service.** Reusable across queries.
- **RRF default; reranker upgrade.** Start simple.
- **Cache aggressively.** Same query → same retrieval; result caches dramatically cut cost.
- **Per-channel observability.** Track vector hit rate, graph hit rate, fusion-changed-the-answer rate. Each tells you something.
- **Provenance preservation.** Each retrieved item carries source, type, score. Downstream synthesis ([135-trustworthy-generation](135-trustworthy-generation.md)) needs it.
- **Re-rank with LLM at the prompt boundary.** A final pass that drops low-relevance items before the planner sees them.

The one-sentence takeaway: **hybrid retrieval is vector + graph + community summaries fused, with reciprocal rank fusion as the default — 30–50% multi-hop gains, 1.5–3× the cost, the dominant retrieval pattern for relational workloads in 2026.**

## See also

- [25-agentic-rag](25-agentic-rag.md) — agentic RAG with self-critique.
- [128-knowledge-graphs-as-substrate](128-knowledge-graphs-as-substrate.md) — the KG side.
- [120-rag-to-finetuning-spectrum](120-rag-to-finetuning-spectrum.md) — RAG positioning.
- [133-reranking-for-agentic-retrieval](133-reranking-for-agentic-retrieval.md), [134-semantic-indexing](134-semantic-indexing.md) — retrieval-side depth.
- [132-vector-cdc-pipelines](132-vector-cdc-pipelines.md) — keeping both indexes fresh.
- [135-trustworthy-generation](135-trustworthy-generation.md) — citing the heterogeneous retrieved context.
- [115-evaluating-llm-systems](115-evaluating-llm-systems.md) — multi-hop eval discipline.
- [37-neuro-symbolic-ai](37-neuro-symbolic-ai.md) — neuro-symbolic framing.
