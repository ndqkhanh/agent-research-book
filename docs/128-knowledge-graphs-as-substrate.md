# 128 — Knowledge Graphs as Agent Substrate: When Typed Entity Graphs Beat Flat Text for Relational Reasoning

**Sources.** Devlin, *Building LLM Agents with RAG Knowledge Graphs*, Chapter 4 (Knowledge Graphs: Giving Structure to Chaos); Stewart & Huang, *Agentic AI Data Architectures*, Chapter 5 (the broader graph + relational integration); plus the foundational KG literature (Berners-Lee's Semantic Web, RDF/SPARQL, Neo4j's property graph model, Microsoft's GraphRAG paper, LinkedIn's Knowledge Graph at Scale).

**One-line definition.** A knowledge graph (KG) — typed nodes (entities) and typed edges (relationships) — is the right substrate for agent context when the questions you ask are *relational* ("who is connected to whom, by what, when") rather than *textual* ("which document mentions X"); chunked-text RAG flattens these relationships, KG retrieval preserves them, and modern agentic stacks increasingly run hybrid graph + vector retrieval to combine the strengths.

## Why this matters

The dominant 2024 RAG pattern was: chunk documents, embed chunks, retrieve top-K by cosine similarity, prepend to prompt. This works for fact-recall queries ("what does the policy say about X?") but breaks on *relational* queries: "who is the manager of the team that owns the service that depends on the database I'm investigating?" Embedding similarity loses graph structure. A chunked-text RAG can retrieve documents about each entity but cannot follow the relationship chain.

For 2026 agent stacks, a knowledge graph is the natural complement to vector retrieval. Where vector retrieval excels at *semantic similarity* over flat text, KG retrieval excels at *structural traversal* over typed relationships. Hybrid systems (graph + vector — see [129-kg-rag-hybrid-retrieval](129-kg-rag-hybrid-retrieval.md)) are the dominant production pattern in regulated and enterprise contexts where relationships matter.

For agent builders, deciding whether to invest in a KG is a single question: do your queries traverse relationships? If yes, KG belongs in the substrate. If no (pure document Q&A), stick with vector RAG.

## Problem it solves

Six concrete query types where chunked-text RAG fails and KG retrieval succeeds:

1. **Multi-hop relational.** "Find all suppliers of company X's competitors." Vector retrieval can't follow "competitor → company → supplier" reliably.
2. **Constraint queries.** "Find employees who report to a director who reports to a VP." Structural; graph traversal native.
3. **Path queries.** "How is person A connected to person B?" Returns a path; vector retrieval has no path concept.
4. **Aggregation by structure.** "Count tasks per team per quarter." Graph + temporal; chunked text loses these axes.
5. **Lineage queries.** "What downstream services consume this database?" Dependency traversal.
6. **Provenance queries.** "Where did this fact come from, and how was it derived?" KG carries provenance edges natively.

Each is structurally relational; KG is the right substrate.

## Core idea in one paragraph

A knowledge graph models the world as **entities** (nodes with types and properties) and **relationships** (edges with types, directions, and properties). Queries traverse the graph: from a starting node, follow edges, filter by properties, aggregate. This is fundamentally different from vector retrieval over flat text — graph queries operate on *structure*, not *similarity*. For agent builders, the KG sits as a complementary substrate alongside vector indexes: vector for "documents about X", graph for "X's relationships." Modern agents query both, fuse the results, and let the LLM synthesise. Building a KG is non-trivial — entity extraction, relationship extraction, schema design, ingestion pipelines, drift handling — but the payoff for relational query workloads is dramatic. The decision rule is: **profile your queries; if more than 30% are relational, build a KG**.

## Mechanism (step by step)

### 1. The KG model — nodes, edges, properties

```text
Node: {id: "person:alice", type: "Person", name: "Alice", joined: "2020-03-15"}
Node: {id: "team:platform", type: "Team", name: "Platform"}
Node: {id: "service:billing", type: "Service", name: "Billing", lang: "Python"}

Edge: (person:alice) -[:MEMBER_OF {since: "2020-03-15"}]-> (team:platform)
Edge: (team:platform) -[:OWNS]-> (service:billing)
Edge: (service:billing) -[:DEPENDS_ON]-> (service:auth)
```

Two paradigms:
- **RDF (triples)**: subject-predicate-object; verbose, standardised, SPARQL.
- **Property graphs**: nodes/edges with properties; richer, Cypher (Neo4j) or Gremlin.

For agent contexts, property graphs are the dominant choice in 2026 due to richer expressiveness and better tooling.

### 2. Schema design

KG schemas have:
- **Entity types**: Person, Team, Service, Document, Project, etc.
- **Relationship types**: MEMBER_OF, OWNS, DEPENDS_ON, AUTHORED, MENTIONS, etc.
- **Property schemas**: per type, what fields exist, with constraints.

Schema design is the critical up-front decision. Three principles:
- **Type things you'll query**, not things that exist. Entity sprawl kills KG quality.
- **Edge directions matter.** Semantic clarity beats neutrality.
- **Properties carry temporal and confidence info.** "since" on edges, "confidence" on extracted facts.

### 3. Construction — entity and relationship extraction

Three approaches:

- **Manual / curated.** High quality, low recall. Right for small canonical KGs (corporate org chart, product taxonomy).
- **Rule-based extraction.** Regex / NER on structured sources. Right for well-formatted inputs.
- **LLM-driven extraction.** Prompt LLMs to extract entities and relationships from unstructured text. Highest recall, variable precision.

Modern hybrid: rules for known patterns + LLM for the long tail + human review for high-stakes additions.

### 4. Querying — traversal over structure

Cypher (Neo4j) example:

```cypher
// Find all employees reporting to anyone reporting to a VP
MATCH (vp:Person {title: 'VP'})<-[:REPORTS_TO*1..2]-(emp:Person)
RETURN emp.name, emp.title

// Find services depending on a given database
MATCH (db:Database {name: $name})<-[:DEPENDS_ON*1..]-(svc:Service)
RETURN svc.name

// Path between two people
MATCH path = shortestPath((a:Person {id: $a})-[*]-(b:Person {id: $b}))
RETURN path
```

These are *structurally typed* queries — they say what relationships to follow, what to filter, what to return. Vector retrieval cannot express these.

### 5. KG + agent integration

Two integration patterns:

**Pattern A: KG as a tool.** The agent has a `query_kg(cypher)` tool; the LLM emits Cypher queries when relational reasoning is needed.
- Pro: flexible; agent decides when to use it.
- Con: LLM must know Cypher; query failures common.

**Pattern B: KG behind retrieval.** A retrieval layer accepts natural-language queries, translates to Cypher (or graph algorithms), executes, returns results.
- Pro: agent doesn't need to know Cypher.
- Con: NL→Cypher is itself a difficult task.

For agent builders: **pattern B for end-user-facing queries; pattern A for technical agents** (e.g. SRE / data exploration).

### 6. GraphRAG — the hybrid pattern

Microsoft's GraphRAG paper (Edge et al. 2024) operationalises KG + vector for RAG:
1. Extract a KG from the corpus.
2. Cluster the KG into communities.
3. Generate per-community summaries.
4. At query time: retrieve community summaries by relevance + traverse graph for structural context + retrieve documents by vector similarity.
5. Fuse all into the prompt.

Reported improvements on multi-hop QA tasks: 30–50% over vanilla vector RAG. Cost: KG construction + community computation up front.

### 7. KG drift and maintenance

KGs go stale. Three drift sources:
- **Source change.** Documents update; KG is stale.
- **Schema evolution.** New entity / edge types needed.
- **Quality drift.** Extraction errors accumulate.

Maintenance:
- **Incremental updates.** When source documents change, re-extract their KG portions.
- **Periodic full rebuild.** Quarterly or annually.
- **Human curation queue.** Flag low-confidence extractions for review.
- **Versioning.** KG snapshots with timestamps; queries can target a specific snapshot.

### 8. Storage and query engines

Major options in 2026:
- **Neo4j**: dominant property-graph DB; mature; AuraDB cloud.
- **Amazon Neptune / GCP Spanner Graph**: cloud-native graph DBs.
- **TigerGraph, Memgraph, ArangoDB**: alternative property-graph engines.
- **JanusGraph**: open-source on top of Cassandra/HBase; scale-friendly.
- **RDF stores (Apache Jena, Stardog)**: where SPARQL and W3C standards matter.
- **In-memory** (NetworkX, igraph): for small KGs, prototyping.

For most agent stacks: **Neo4j or a cloud-managed property graph**.

### 9. KG vs vector store sizing

| Aspect | Vector store | Knowledge graph |
|---|---|---|
| Storage | proportional to chunks × embedding dim | proportional to entities + edges |
| Query cost | similarity search (logarithmic) | traversal (depends on graph structure) |
| Update cost | per-chunk re-embed | per-entity / edge update |
| Maintenance | re-index periodically | extraction + schema evolution |
| Best query type | semantic similarity | structural traversal |

Hybrid systems run both; storage cost is the sum.

## Empirical anchors

- **GraphRAG** outperforms vanilla RAG by 30–50% on multi-hop QA in Microsoft's reported benchmarks.
- **Multi-hop performance** of vector RAG drops sharply at 3+ hops; graph traversal stays robust.
- **Construction cost** dominates KG TCO; LLM-driven extraction at scale costs $0.01–0.10 per document.
- **Quality of LLM-extracted KGs** is mediocre out of the box; human review or rule augmentation lifts quality 20–40%.
- **Adoption** in regulated industries (finance, healthcare, supply chain) is high; in pure-document Q&A, low.

## Variants and counter-arguments addressed

- **"Just chunk smaller and embed better."** Helps modestly; doesn't recover relational structure.
- **"LLM context is big enough."** Effective context is much smaller (see [110-transformer-llm-architecture-for-agent-builders](110-transformer-llm-architecture-for-agent-builders.md)); even with 1M tokens, traversal needs structure.
- **"KGs are too expensive to build."** True for some workloads; check the relational-query share before deciding.
- **"Vector + reranking covers most cases."** Most "easy" cases yes; relational queries no.
- **"GraphRAG is overhyped."** Real gains on the right workloads; the hype is about the framing, the gains are real.
- **"NL to Cypher is too hard."** Modern LLMs are competent; constrained decoding ([112-constrained-decoding](112-constrained-decoding.md)) on graph schema makes it production-grade.

## Failure modes and limitations

1. **Schema sprawl.** Too many entity / edge types; KG becomes unmaintainable. Constrain schema deliberately.
2. **Extraction errors.** Wrong entity merging (Apple Inc ≠ apple fruit), wrong relationships. Invest in disambiguation.
3. **Stale data.** Source documents updated; KG lags. Incremental update infrastructure is essential.
4. **Query cost on dense graphs.** Some traversals fan out exponentially; bound depth, use indexes.
5. **NL→query brittleness.** LLM emits invalid Cypher; constrained decoding helps but doesn't eliminate.
6. **Cold-start KG.** Building from scratch is expensive; consider partial / rolling deployment.
7. **Schema evolution.** Adding new entity types requires re-extraction; design for evolution.
8. **Multi-tenancy.** KGs in multi-tenant deployments need namespace isolation; not all engines support this cleanly.

## When to use, when not

**Use a KG when** ≥30% of your agent's queries are relational, multi-hop, lineage, or structural; or your domain is intrinsically graph-shaped (org charts, supply chains, dependency graphs, knowledge bases).

**Skip the KG when** queries are predominantly fact-recall over documents (vanilla RAG suffices), or when extraction cost exceeds the relational-query value.

**Build hybrid (KG + vector)** when you have both query types — the dominant 2026 pattern.

## Implications for harness engineering

- **Profile queries before deciding.** Don't build a KG without evidence that relational queries justify it.
- **Schema-first.** A small, well-typed KG beats a large, messy one. See [134-semantic-indexing](134-semantic-indexing.md) for the indexing-side analog.
- **Extraction pipeline as code.** Versioned, evaluable, A/B-able. See [115-evaluating-llm-systems](115-evaluating-llm-systems.md).
- **NL→Cypher with constrained decoding.** [112-constrained-decoding](112-constrained-decoding.md). Schema-aware grammar dramatically reduces query errors.
- **GraphRAG as a default for hybrid.** [129-kg-rag-hybrid-retrieval](129-kg-rag-hybrid-retrieval.md) — the integration pattern.
- **Provenance edges.** Every fact in the KG carries a provenance edge to its source. Required for audit and trust ([122-explainability-compliance](122-explainability-compliance.md), [135-trustworthy-generation](135-trustworthy-generation.md)).
- **Incremental update infrastructure.** Source change → KG patch in minutes, not weeks.
- **Bound query depth.** Configurable max-hops; protects against runaway traversals.

The one-sentence takeaway: **a knowledge graph is the right substrate when your queries traverse relationships — invest in one when relational queries dominate, hybrid with vector when both query types matter, and skip it when chunked-text RAG already suffices.**

## See also

- [25-agentic-rag](25-agentic-rag.md) — agentic RAG with self-critique.
- [37-neuro-symbolic-ai](37-neuro-symbolic-ai.md) — KGs are a neuro-symbolic primitive.
- [129-kg-rag-hybrid-retrieval](129-kg-rag-hybrid-retrieval.md) — the integration pattern.
- [120-rag-to-finetuning-spectrum](120-rag-to-finetuning-spectrum.md) — KG as one substrate on the adaptation spectrum.
- [134-semantic-indexing](134-semantic-indexing.md), [135-trustworthy-generation](135-trustworthy-generation.md) — adjacent retrieval-side patterns.
- [122-explainability-compliance](122-explainability-compliance.md) — provenance edges support audit.
- [130-distributed-sql-as-agent-memory](130-distributed-sql-as-agent-memory.md), [131-temporal-bitemporal-tables](131-temporal-bitemporal-tables.md) — alternative structured-data substrates.
