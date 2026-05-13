# 200 — Graph-Grounded Multi-Hop Retrieval: From MINERVA to AnchorRAG

> **Disambiguation.** This file is the **graph-grounded retrieval** half of a four-part block (198–201) on multi-hop reasoning. It complements [128-knowledge-graphs-as-substrate](128-knowledge-graphs-as-substrate.md) (graphs as memory) and [129-kg-rag-hybrid-retrieval](129-kg-rag-hybrid-retrieval.md) (hybrid retrieval) by focusing specifically on the *multi-hop* case. The dataset canon is in [198-multi-hop-qa-datasets-canon](198-multi-hop-qa-datasets-canon.md), the technique arc in [199-multi-hop-reasoning-techniques-arc](199-multi-hop-reasoning-techniques-arc.md), the formal compositionality lineage in [201-compositionality-gap-canon](201-compositionality-gap-canon.md).

## One-line definition

**Graph-grounded multi-hop retrieval** uses an explicit (or LLM-extracted) knowledge graph as a first-class index — encoding multi-hop relations into the structure once at ingest, so query-time retrieval can return *multi-hop-coherent evidence in a single pass* via graph traversal, Personalized PageRank, community summarisation, or learned KG walks — and as of May 2026 the family contains the highest-momentum open-source RAG repos (LightRAG 34.9k★, GraphRAG 32.8k★, HippoRAG 3.5k★).

## Why this family matters

Vector RAG's core failure on multi-hop is that it *assumes the answer-bearing passage is similar to the question*. For a 2-hop question — "*the spouse of the director of Casablanca*" — the answer-bearing passage may share zero lexical or semantic surface with the question. Graph-grounded retrieval fixes this by encoding the *relation structure* of the corpus into edges that the retriever can traverse: from the question's entity surface to the answer entity through one or more hops, regardless of passage-level semantic distance.

The lineage runs from the **2018 MINERVA / MultiHopKG** RL-walks-on-KG line (pre-LLM, KG-only) through **2024 GraphRAG / HippoRAG / LightRAG** (LLM-extracted KGs over arbitrary corpora) to **2024–2025 Plan-on-Graph / AnchorRAG / R²AG / HopRAG** (agentic, self-correcting, RL-trained graph navigation). Three structural insights recur. First, **encode multi-hop relations once at ingest** (HippoRAG's hippocampal index, GraphRAG's community detection) — query-time becomes a single PPR or community lookup, not iterated retrieval. Second, **the LLM extracts the graph cheaply and well** — Microsoft GraphRAG, LightRAG, HippoRAG all show that schemaless LLM-extracted KGs are competitive with hand-crafted ones. Third, **the agent navigates the graph adaptively** — Plan-on-Graph's backtracking and AnchorRAG's multi-agent traversal beat fixed-breadth BFS by accommodating wrong entity-linking and wrong direction guesses.

Take this family seriously and three things change. (1) For corpora >10M tokens, graph-RAG dominates flat-vector RAG on multi-hop — the gap on global-sensemaking queries is large. (2) The retrieval cost amortises across queries — HippoRAG runs **10–30× cheaper, 6–13× faster** than IRCoT at matching quality because the multi-hop work happens at ingest. (3) Production RAG systems converge on a hybrid: graph index + vector index + reranker, where the graph carries the multi-hop signal and the vector index carries the lexical/semantic one.

## Problem this family solves

- Vector top-k retrieves redundant single-hop evidence; the second-hop bridge entity may never be retrieved at all.
- Iterative retrieve-reason loops (IRCoT, Self-Ask) pay the multi-hop cost *every query*; the cost compounds.
- Global / sensemaking queries — "*what are the main themes across this corpus?*" — have no answer-bearing passage; vector RAG fails by definition.
- Entity linking is unreliable; classical KGQA pipelines collapse when entity mentions are ambiguous or missing.
- Multi-hop chains over heterogeneous sources (KG triples + free text + tables) need a substrate that can fuse them.

## The pre-LLM lineage — KG walks (2018–2022)

### MINERVA (Das et al. 2018)

- **Citation.** *Go for a Walk and Arrive at the Answer: Reasoning Over Paths in Knowledge Bases using Reinforcement Learning*. ICLR 2018, arXiv 1711.05851.
- **Mechanism.** A REINFORCE-trained agent walks the KG from a query entity along outgoing edges, conditioned on the relation type being asked about. Variable path length; needs no per-relation training.
- **Why it matters.** First widely cited "**path-as-reasoning**" KG agent. Reasoning paths are interpretable provenance. The conceptual ancestor of every later "let the LLM walk a graph" system.
- **Repo.** [github.com/shehzaadzd/MINERVA](https://github.com/shehzaadzd/MINERVA).

### MultiHopKG (Lin et al. 2018)

- **Citation.** *Multi-Hop Knowledge Graph Reasoning with Reward Shaping*. EMNLP 2018, arXiv 1808.10568.
- **Mechanism.** Uses a pretrained one-hop KG-embedding model to estimate rewards for unobserved facts (countering false-negative supervision); action-dropout via random edge masking counters spurious paths.
- **Why it matters.** Established two techniques every later KG-RL system uses — dense reward shaping from a base model, and stochastic action masking to escape spurious-path attractors.
- **Repo.** [github.com/salesforce/MultiHopKG](https://github.com/salesforce/MultiHopKG).

### SMORE (Ren et al. 2022)

- **Citation.** *Knowledge Graph Completion and Multi-hop Reasoning in Massive Knowledge Graphs*. KDD 2022, arXiv 2110.14890.
- **Mechanism.** Distributed system that scales multi-hop KG reasoning to billion-edge graphs.
- **Why it matters.** Made graph-walk reasoning practical at industrial scale; the missing-link between academic KG-walks and the LLM-era graph-RAG systems that followed.

The pre-LLM line established that **path-as-reasoning is interpretable and learnable**. Modern LLM+KG systems (PoG, AnchorRAG, R²AG) inherit this lineage.

## The 2024 LLM-extracted-KG wave

### GraphRAG (Microsoft; Edge et al. 2024)

- **Citation.** *From Local to Global: A Graph RAG Approach to Query-Focused Summarization*. arXiv 2404.16130.
- **Mechanism.** Two-stage pipeline:
  1. **Indexing.** LLM extracts entity-relation graph from corpus → run **Leiden community detection** → for each community, generate an LLM-written **community summary** at multiple granularities.
  2. **Querying.** For "global" queries, route to the community-summary level matching the query scope; aggregate community-level partial answers into a final answer. For "local" queries, fall back to entity-neighborhood retrieval.
- **Why it matters.** Solves the **global sensemaking** case — questions whose answer needs *the whole corpus*, not specific passages. Vector RAG fails here by construction; GraphRAG was the first practical fix at the 1M-token scale.
- **Variants.** *DRIFT search* (hybrid local+global), *LazyGraphRAG* (defer summaries to query-time, ~700× cheaper indexing).
- **Repo.** [github.com/microsoft/graphrag](https://github.com/microsoft/graphrag) — **32,849★**, very active.

### HippoRAG (Gutiérrez et al. 2024) — single-shot multi-hop

- **Citation.** *HippoRAG: Neurobiologically Inspired Long-Term Memory for Large Language Models*. NeurIPS 2024, arXiv 2405.14831. (OSU NLP.)
- **Mechanism.** LLM extracts a **schemaless KG** (the "hippocampal index"). At query time, query concepts seed **Personalized PageRank** over the KG; the PPR distribution integrates multi-hop evidence in a *single retrieval step*. No iterative retrieve-reason loop.
- **Headline numbers.** Up to **+20% on multi-hop QA**; single-step retrieval **matches IRCoT at 10–30× cheaper, 6–13× faster**.
- **Why it matters.** The most important "single-hop retrieval that gives multi-hop answers" insight. By encoding multi-hop relations into the KG once at ingest, query-time retrieval needs only one PPR pass — the multi-hop work amortises across queries.
- **HippoRAG-2 (2025).** Improves entity linking, adds a learned reranker, scales to billion-token corpora.
- **Repo.** [github.com/OSU-NLP-Group/HippoRAG](https://github.com/OSU-NLP-Group/HippoRAG) — 3,492★.

### LightRAG (Guo et al. 2024) — pragmatic GraphRAG

- **Citation.** *LightRAG: Simple and Fast Retrieval-Augmented Generation*. arXiv 2410.05779, EMNLP 2025 Findings. (HKU DS.)
- **Mechanism.** **Dual-level retrieval**:
  - *Low-level.* Entity / relation lookups over the LLM-extracted KG.
  - *High-level.* Theme / topic lookups over LLM-summarised concepts.
  - **Incremental update algorithm** integrates new docs without full re-indexing — a missing piece in GraphRAG.
- **Why it matters.** GraphRAG-class quality at much lower indexing cost, with online ingest. The "pragmatic GraphRAG" of choice for many production pipelines in 2025–26.
- **Repo.** [github.com/HKUDS/LightRAG](https://github.com/HKUDS/LightRAG) — **34,926★** as of May 2026, currently trending higher than GraphRAG itself.

### KG-RAG family (production-grade)

Not a single paper but a pattern: vector retrieval finds *entry entities* → graph traversal expands to multi-hop neighborhoods → LLM synthesises. Reference implementations:

- **Neo4j neo4j-graphrag-python** — production-grade GraphRAG primitives over Neo4j.
- **VectorInstitute/kg-rag** — academic implementation with structured eval.
- **arXiv 2404.19234** — *Multi-Hop QA over KGs using LLMs*; the canonical formalism for this hybrid pattern.

The standard production shape in 2026: vector index + property-graph index + reranker, with the graph carrying multi-hop signal and the vector index carrying lexical/semantic similarity.

## The 2024–2025 agentic-graph wave

### Plan-on-Graph (Chen et al. 2024)

- **Citation.** *Plan-on-Graph: Self-Correcting Adaptive Planning of LLMs on Knowledge Graphs*. NeurIPS 2024, arXiv 2410.23875.
- **Mechanism.** Three core mechanisms:
  - **Guidance.** Decompose the query into sub-objectives; each sub-objective drives the next graph step.
  - **Memory.** Maintain a working subgraph + visited paths + per-node status (explored / pending / pruned).
  - **Reflection.** Self-correct when a path is wrong — backtrack, prune, restart with a different sub-objective.
- **Headline.** Best published numbers on CWQ / WebQSP / GrailQA at release.
- **Why it matters.** Fixes the fragility of fixed-breadth KG search. The **adaptive breadth + backtracking** pattern is now standard in agent-graph systems.
- **Repo.** [github.com/liyichen-cly/PoG](https://github.com/liyichen-cly/PoG) — 96★.

### AnchorRAG (2025) — multi-agent open-world KGQA

- **Citation.** *Towards Open-World RAG on Knowledge Graphs: A Multi-Agent Collaboration Framework*. arXiv 2509.01238.
- **Mechanism.** Three-agent system:
  - **Predictor** — finds plausible *anchor entities* without predefined entity-linking ground truth.
  - **Retrievers** (parallel) — multi-hop expansion from each anchor.
  - **Supervisor** — selects, merges, answers.
- **Why it matters.** Most KGQA pipelines fail when entity linking is wrong. AnchorRAG drops the assumption of perfect linking — useful in open-world conditions where mention surfaces don't map cleanly to KG nodes.

### R²AG / HopRAG / RELOOP — the 2025 reasoning-retrieval-reasoning-refining lineage

- **R²AG** (OpenReview ey5sQDCWlM) — **RL-trained retrieval steering** over a multi-hop retrieval *tree*; **+24.1% recall, +20.4% accuracy** over naive RAG.
- **HopRAG** (arXiv 2502.12442) — passage graph with **LLM-pseudo-query edges**; retrieve→reason→prune.
- **RELOOP** (arXiv 2510.20505) — recursive retrieval with a multi-hop reasoner-planner stack.

All three are 2025 expressions of the same pattern: **replace fixed-pipeline RAG with a search tree over retrieval/reasoning, then learn to navigate it.**

### Tree-of-Thought-on-Graph and Paths-over-Graph (2024–2025)

- **ToG (Tree-of-Thoughts on Graph)** — beam search over candidate KG paths with LLM-scored pruning at each depth.
- **PoG-Paths** — explicit multi-path enumeration before LLM-side selection; useful when multiple legitimate reasoning paths exist.

## Comparative architecture

| System | Index | Query-time | Multi-hop encoded | Adaptive | Cost @ ingest | Cost @ query |
|---|---|---|---|---|---|---|
| Vector RAG | Embedding ANN | Top-k | No | No | Low | Low |
| IRCoT | Vector | Iterative | Per-query | No | Low | High |
| GraphRAG | Entity-relation graph + community summaries | Community lookup or entity-neighborhood | At ingest (community detection) | No | **High** | Low |
| LightRAG | Dual-level graph | Entity + theme | At ingest, incremental | No | Mid | Low |
| HippoRAG | Schemaless KG | Personalized PageRank from query concepts | At ingest (PPR weights) | No | Mid | **Very low** |
| Plan-on-Graph | KG (existing) | Agent walk + backtrack | Per-query | **Yes** | Low | Mid |
| AnchorRAG | KG | 3-agent collab | Per-query | **Yes** | Low | High |
| R²AG | Multi-hop tree | RL policy | Trained | **Yes** | Mid | Mid |

The sharp split: **graph-as-cache** systems (GraphRAG, LightRAG, HippoRAG) push cost to ingest and amortise it; **agentic graph-walk** systems (PoG, AnchorRAG, R²AG) push cost to query-time but adapt to per-query needs. Production deployments increasingly stack both — graph-as-cache for the *what* (which sub-graph is relevant), agent-walk for the *how* (which path through it answers this exact question).

## Empirical results

Selected headline numbers, comparable-when-the-paper-allows:

- **GraphRAG vs Vector RAG, 1M-token podcast corpus.** Comprehensiveness 50% → 80%; diversity 30% → 70% (GraphRAG paper §6).
- **HippoRAG, MuSiQue F1.** ~58 (single retrieval) vs ~50 IRCoT (iterated) at **10–30× cheaper**.
- **HippoRAG-2, billion-token corpus.** Maintains <5s query latency; multi-hop F1 holds within 2 points of HippoRAG-1.
- **LightRAG, MultiHop-RAG accuracy.** ~0.71, beating GraphRAG-default at materially lower indexing cost.
- **Plan-on-Graph, CWQ Hits@1.** ~58, beating prior best by ~3 points.
- **AnchorRAG, four KGQA benchmarks.** Beats KGQA SOTA on all four; variance highest on benchmarks with poor entity-linking ground truth (where AnchorRAG's anchor predictor matters most).
- **R²AG, WikiMultiHopQA.** +24.1% retrieval recall, +20.4% accuracy vs flat RAG.

## Variants and ablations worth knowing

- **Schemaless vs schema-constrained extraction.** Schemaless (HippoRAG) is more flexible; schema-constrained (Microsoft GraphRAG) is more queryable. Trade quality of extraction against queryability of the resulting graph.
- **Community detection algorithm.** Leiden (Microsoft) > Louvain on stability; spectral clustering occasionally beats both on dense corpora.
- **PPR seeding.** HippoRAG uses query-extracted concepts as seeds; alternatives include retrieval-based seeding (top-k vector hits → PPR) and LLM-suggested seeding (query-conditioned).
- **Edge weighting.** Uniform vs LLM-confidence-weighted edges materially affect PPR rankings; weighted edges are usually worth the cost.
- **Hybrid index.** Graph + vector + BM25 + reranker is the typical 2026 production shape; reranker on the union of all three sets.
- **Online ingest.** GraphRAG requires full re-index for new docs; LightRAG handles online ingest natively. For corpora that drift (news, internal docs), this is a hard constraint.
- **Entity-linking error handling.** AnchorRAG's anchor predictor; PoG's reflection-on-wrong-path; HippoRAG-2's learned reranker — three different stances on the same problem.

## Failure modes and limitations

- **LLM-extracted-graph errors propagate.** A wrong relation extracted at ingest poisons every downstream multi-hop query touching it. Periodic graph-quality audits are essential.
- **Entity coreference.** Same entity surfaced as different nodes (e.g. "JFK", "John F. Kennedy", "President Kennedy") fragments multi-hop chains. Most systems use LLM-side merging at query time; some (HippoRAG-2) use learned canonicalisation at ingest.
- **Graph staleness.** Wikipedia-extracted KGs go stale fast (~6 months for hot topics). LightRAG's incremental ingest is the cheapest mitigation.
- **Privacy / leakage.** A graph index of an enterprise corpus is a higher-leverage attack surface than a vector index — one node link can expose a whole multi-hop chain. See [22-guardrails-prompt-injection](22-guardrails-prompt-injection.md), [82-poisonedrag](82-poisonedrag.md).
- **Cost at ingest.** Microsoft GraphRAG's full pipeline runs **$$ per 1M tokens** because of the multi-pass LLM extraction. LazyGraphRAG and LightRAG are explicit cost-reduction variants.
- **Multi-hop *across* graph + vector.** When the answer needs both KG-side and free-text-side evidence, naive concatenation can confuse the reader; CoK-style fusion ([199-multi-hop-reasoning-techniques-arc](199-multi-hop-reasoning-techniques-arc.md)) is required.
- **Open-world entity drift.** Real-world KGs are always partial; AnchorRAG's anchor-predictor pattern is the right answer to "what if the entity isn't in the graph yet?"
- **Path explosion.** Beyond 3–4 hops, naive BFS over the graph explodes; PoG-style backtracking + community-summary routing keep the search tractable.

## When to use, when not

**Use graph-RAG** when (a) the corpus exceeds ~1M tokens, (b) multi-hop and global-sensemaking queries are common, (c) ingest is a one-time or amortised cost, (d) you have budget for ~$0.05–$0.20/1M tokens of LLM-extracted graph at ingest. The query-time savings dominate for any corpus you query more than ~50 times.

**Use HippoRAG specifically** when query latency is the binding constraint and queries are well-served by entity-grounded multi-hop. For pure factual-retrieval workloads it's hard to beat.

**Use LightRAG specifically** when ingest must be online (corpus updates daily) and the query mix includes both entity-level and theme-level queries.

**Don't use graph-RAG** when (a) the corpus is small (<100k tokens — vector RAG wins), (b) queries are single-hop factual lookups (vector RAG matches at lower cost), (c) ingest cost is hard-capped (use LazyGraphRAG instead), (d) the corpus has unstable entities and no canonicalisation pipeline.

## Implications for harness engineering

- **Treat the graph as long-term memory.** A LLM-extracted KG is durable, queryable, and auditable — it belongs alongside [09-memory-files](09-memory-files.md), [128-knowledge-graphs-as-substrate](128-knowledge-graphs-as-substrate.md), and the broader [185-memory-integration-playbook](185-memory-integration-playbook.md).
- **Index at ingest, not at query time.** The HippoRAG insight applies generally — push multi-hop work to ingest whenever you can amortise. See [134-semantic-indexing](134-semantic-indexing.md).
- **Hybrid index is the production default.** Graph + vector + BM25 + reranker. Cf. [25-agentic-rag](25-agentic-rag.md), [129-kg-rag-hybrid-retrieval](129-kg-rag-hybrid-retrieval.md), [133-reranking-for-agentic-retrieval](133-reranking-for-agentic-retrieval.md).
- **Anchor-predictor before traversal.** Don't trust deterministic entity linking; route through an anchor-prediction step (AnchorRAG) for any open-world deployment. Cf. [180-argus-skill-router-agent-design](180-argus-skill-router-agent-design.md).
- **Path provenance is a free win.** Graph-walk paths are interpretable; surface them as provenance on every multi-hop answer. Cf. [188-witness-provenance-memory-techniques-synthesis](188-witness-provenance-memory-techniques-synthesis.md).
- **Adaptive depth + backtracking.** Plan-on-Graph's pattern beats fixed-depth BFS; bake backtracking into any agent that walks the graph at query time. Cf. [15-tree-of-thoughts-lats](15-tree-of-thoughts-lats.md).
- **Online incremental ingest.** Bake in a LightRAG-style update path; full reindex pipelines silently bottleneck production. Cf. [132-vector-cdc-pipelines](132-vector-cdc-pipelines.md).
- **Graph quality as a regression metric.** Periodic LLM-extraction audits — sample N triples, judge correctness, alarm on drift. Cf. [115-evaluating-llm-systems](115-evaluating-llm-systems.md).
- **Two-axis evidence gates.** Compose graph-walk provenance with a CiteGuard-style attribution gate and a Contradiction-to-Consensus inter-source gate. Cf. [172-polaris-2026-deep-research-roadmap](172-polaris-2026-deep-research-roadmap.md) §3 Gap 5.
- **Domain-specific graph shells.** Per-domain extraction prompts beat generic ones (medical, legal, ML). Cf. [149-sector-use-case-catalog](149-sector-use-case-catalog.md).
- **Cost dashboards.** Graph ingest is sneaky-expensive; instrument $/M-tokens-ingested per release. Cf. [86-frugalgpt](86-frugalgpt.md).
- **Test graph-RAG with the right benchmarks.** MultiHop-RAG, FRAMES, FanOutQA — the dataset canon ([198-multi-hop-qa-datasets-canon](198-multi-hop-qa-datasets-canon.md)) calls out which evaluations actually surface graph-RAG's edge.

**The one-line takeaway for harness designers:** Pay the multi-hop cost once at ingest with a graph-RAG cache (HippoRAG / LightRAG / GraphRAG), navigate it adaptively at query time (PoG / AnchorRAG), and stop iterating retrieval-reason loops at every query.
