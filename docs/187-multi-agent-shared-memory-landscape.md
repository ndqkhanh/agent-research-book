# 187 — Multi-Agent Shared Memory: Landscape, Threats, Defenses (May 2026)

**Scope.** A survey of the *multi-agent* corner of the agent-memory landscape — the systems and papers that explicitly target shared state, cross-agent coordination, attribution, contested writes, and adversarial inputs. Sister to [docs/183](183-oss-memory-landscape-may-2026.md) (single-agent OSS landscape) and [docs/186](186-mnema-witness-lattice.md) (Mnema absorption); closes the gap doc 184 left in the design-space matrix's sixth axis: trust assumption.

**One-paragraph thesis.** Single-agent memory (mem0, Letta, Zep, Cognee, A-MEM, HippoRAG 2 — all of [docs/183](183-oss-memory-landscape-may-2026.md)) assumes cooperative authorship: the agent writes, the store accepts, retrieval ranks. Multi-agent memory cannot make that assumption. Three failure modes break it: (1) **memory poisoning** — an adversary injects facts via shared retrieval / shared experience pools / shared environment and hijacks downstream agents (90% success on PoisonedRAG; 50–90% recall corruption under MemoryGraft / AgentPoison); (2) **coordination collapse** — 41–86.7% across seven SOTA frameworks per MAS-Bench / MAST; (3) **attribution loss** — when N agents write to one store, "who said what when, and on what evidence" is the question regulators ask first. The landscape splits into **attack papers** (PoisonedRAG, AgentPoison, MemoryGraft, Poison-Once-Exploit-Forever), **defense papers** (A-MemGuard, Cryptographic Provenance Attestation, Byzantine-fault-tolerant consensus), and **shared-memory architectures** (A-MEM Zettelkasten, Letta multi-agent, Supermemory MCP, Mnema witness lattice). The strongest convergent technique today is **signed provenance + structural decorrelation**: every shared-memory write is signed by the originating agent; every consensus decision is verified across structurally-decorrelated voters (different model families, different ingestion paths). That's a four-line addition to mem0 / Letta. The α floor in [Mnema's bound](186-mnema-witness-lattice.md) — `P_undetected = α + (1−α)·β^(1+q)` — names what the limit is even with infinite redundancy.

---

## §1 — Why multi-agent memory is a different problem

Single-agent memory in [docs/184](184-strongest-memory-techniques-synthesis-may-2026.md) optimises for: retrieval accuracy, temporal validity, scope stratification, tier discipline, tool-call auditability. Add a second agent and three orthogonal concerns appear:

| Concern | Single-agent | Multi-agent |
|---|---|---|
| **Authorship** | one writer; trivial | N writers; needs attribution per row |
| **Trust** | trust the agent | each row's trust ≤ min(trust(writer), trust(retrieval-path)) |
| **Conflict** | append-only with retraction | active contradiction needs resolution semantics |
| **Replay** | reproduce one trajectory | reproduce *the joint trajectory* — every agent's read order matters |
| **Poisoning surface** | local memory tampering | shared memory + shared retrieval + shared experience pool |
| **Coordination** | n/a | the failure mode that MAS-Bench / MAST instruments |

The first axis (authorship) drives the next four. *No published single-agent memory system handles this directly* — they handle it indirectly via API-side filtering (mem0's `user_id` scope is the closest analogue; it's not real attribution).

---

## §2 — The threat-model layer

### Memory-poisoning attacks (the prior art that defines the threat model)

| Attack | Year | Target | Headline result |
|---|---|---|---|
| **PoisonedRAG** ([arXiv:2402.07867](https://arxiv.org/abs/2402.07867)) | 2024 | RAG knowledge base | ~90% steer-to-attacker-chosen-answer with a handful of injected docs |
| **AgentPoison** ([NeurIPS 2024](https://openreview.net/forum?id=Y841BRW9rY)) | 2024 | Agent long-term memory + RAG KB | First backdoor attack on agent memory; near-total compromise on triggered queries |
| **MemoryGraft** ([arXiv:2512.16962](https://arxiv.org/abs/2512.16962)) | 2025 | Long-term experience pool | Indirect injection via benign-looking external content (README files); agent learns the malicious experience as its own |
| **Poison Once, Exploit Forever** ([arXiv:2604.02623](https://arxiv.org/abs/2604.02623)) | 2026 | Web agents | Environment-injected memory poisoning persists across sessions; one injection → permanent compromise |
| **Memory Poisoning Attack and Defense** ([arXiv:2601.05504](https://arxiv.org/abs/2601.05504)) | 2026 | Memory-based LLM agents | Survey + defense framework |

Common shape: **the poisoning surface is the *write path*, not the *read path***. Every attack succeeds because the agent / store accepts the poisoned write at face value and ranks it later as if it were genuine. This is the load-bearing reason Mnema and A-MemGuard reach for cryptographic write-time discipline.

### Coordination collapse (the second pillar)

Documented across seven frameworks: **41–86.7% failure rate** on coordination tasks (per MAS-Bench / MAST, summarised in [docs/186](186-mnema-witness-lattice.md)). The collapse correlates with shared-memory contention: agents reading partially-updated state, agents updating based on stale reads, agents accepting another agent's poisoned output as truth.

### Unauditability (the regulatory pillar)

EU AI Act Article 13 / HIPAA §164.312 / SOC 2 CC7 all demand replayable provenance. Single-agent stores can replay one trajectory; multi-agent stores must replay the *interleaving*. Without per-row signatures + ordered journals, replay is an approximation, not a proof.

---

## §3 — The defense layer (papers)

### A-MemGuard — proactive defense for agent memory

[arXiv:2510.02373](https://arxiv.org/abs/2510.02373). Frames memory poisoning as a structural problem: the agent cannot distinguish self-generated experiences from externally-injected artefacts. A-MemGuard interposes a check at write-time + retrieval-time and reports SOTA results on AgentPoison / MemoryGraft / PoisonedRAG with attack success rate driven near zero in many settings.

Take-aways:
- **Write-time check is necessary.** Retrieval-only filtering is insufficient because the corrupted row still ranks well on similarity.
- **Lifecycle isolation.** Self-generated and externally-fed memories travel through structurally distinct paths; mixing them is the failure mode.

### Cryptographic Provenance Attestation (CPA)

The defense pattern proposed in the MemoryGraft response. Core: every memory insertion runs through a *trusted insertion protocol* where the agent holds a private signing key. Memories without a valid signature cannot enter the experience pool. Compromise model: an attacker who controls external content but not the agent's private key cannot forge memories.

This is exactly the Mnema witness shape ([docs/186 §2](186-mnema-witness-lattice.md)) reduced to "every write is signed." Mnema generalises CPA from "one key per agent" to "one key per memory item" — but the engineering minimum is CPA, and CPA alone closes a large fraction of the threat surface.

### Byzantine-fault-tolerant LLM consensus (the consensus tier)

A cluster of 2025–26 papers attacks the consensus problem from BFT first principles:

| Paper | Headline |
|---|---|
| **CP-WBFT** ([arXiv:2511.10400](https://arxiv.org/abs/2511.10400)) | Confidence-probe-weighted BFT consensus; works under 85.7% Byzantine fault rate |
| **Byzantine-Robust Decentralized Coordination of LLM Agents** ([arXiv:2507.14928](https://arxiv.org/abs/2507.14928)) | Decentralized + leaderless BFT for agent coordination |
| **Trusted MultiLLMN with WBFT consensus** ([arXiv:2505.05103](https://arxiv.org/abs/2505.05103)) | Weighted-vote BFT mediates between multiple LLMs; suppresses voting power of malicious nodes |
| **A BFT Approach towards AI Safety** ([arXiv:2504.14668](https://arxiv.org/abs/2504.14668)) | Treats AI-safety guarantees as BFT properties |
| **Can AI Agents Agree?** ([arXiv:2603.01213](https://arxiv.org/abs/2603.01213)) | Empirical study: classical BFT guarantees don't translate cleanly to stochastic prompt-driven agents |
| **FREE-MAD** ([arXiv:2509.11035](https://arxiv.org/abs/2509.11035)) | Consensus-free multi-agent debate — questions whether BFT is the right hammer |

The **CP-WBFT 85.7% fault tolerance** number is the headline; the **"Can AI Agents Agree?" empirical pushback** is the load-bearing caveat. Classical BFT assumes deterministic-process voters; stochastic LLM voters break the assumption. Mnema's response is the cross-family critic-jury (structural decorrelation) rather than higher fault tolerance via larger N.

---

## §4 — The shared-memory architectures (systems / repos)

### A-MEM — Zettelkasten-inspired agentic memory

Paper: [arXiv:2502.12110](https://arxiv.org/abs/2502.12110), NeurIPS 2025. Repos:

- [`WujiangXu/A-mem`](https://github.com/WujiangXu/A-mem) — official research repo
- [`WujiangXu/A-mem-sys`](https://github.com/WujiangXu/A-mem-sys) — production system
- [`agiresearch/A-mem`](https://github.com/agiresearch/A-mem) — community fork
- [`tobs-code/a-mem-mcp-server`](https://github.com/tobs-code/a-mem-mcp-server) — MCP integration; ChromaDB + NetworkX DiGraph with explicit typed edges

**Architecture.** Memory notes form a Zettelkasten — each note is a Markdown blob with a unique slug + typed edges (`elaborates`, `contradicts`, `cites`, `derives_from`) to other notes. The agent dynamically *reorganises* notes (adds edges, splits, merges) at write-time; retrieval traverses the graph rather than ranking flat rows.

**What's novel for multi-agent**: the typed-edge schema makes attribution structural — `note_42 cites note_17` is a fact, not a hint. Multi-agent extension is straightforward (add a `signed_by` field per note); the paper doesn't formalise it but the shape is right.

**Best for**: research-shaped + structured-claim memory. Polaris's Program Graph is a degenerate A-MEM (typed nodes + typed edges, but without the dynamic-rewrite operation).

### Letta multi-agent

[`letta-ai/letta`](https://github.com/letta-ai/letta), 22.5K★. The MemGPT-successor's primary surface is single-agent, but two patterns generalise:

1. **Shared archival memory**: multiple agents can mount the same archival store; reads see writes from any agent, attribution preserved by tool-call trace.
2. **Memory-as-tool-call**: every read/write is in the reasoning trace of the *originating* agent. The interleaving across agents is reconstructable from the cross-agent trace.

Limitation: Letta does not sign rows. An attacker who can post a `write_to_archival` tool-call on any participating agent can poison the shared store.

### Supermemory MCP cross-IDE memory

[`supermemoryai/supermemory`](https://github.com/supermemoryai/supermemory), 22.5K★. Multi-IDE / multi-agent via MCP server: same memory mounted into Claude Code, Cursor, ChatGPT, Perplexity, etc. **Solves the "user wants memory continuity across tools" problem; does not solve the "agents distrust each other" problem.** Authorship is by `user_id` not by IDE.

### mem0 with `agent_id` scope

[`mem0ai/mem0`](https://github.com/mem0ai/mem0), 55K★. The 2026-04 redesign added `agent_id` as a peer of `user_id` and `session_id`. Reads / writes filter by triple (`user_id`, `session_id`, `agent_id`). This is **scope-stratified attribution** — closer to "namespaced multi-tenancy" than to "real attribution" — but it's the pragmatic baseline for chat-shaped multi-agent (e.g. CrewAI agents writing into a shared mem0 instance).

### agentmemory (rohitg00/agentmemory)

Discussed in detail relative to Lyra's stack at [docs/183 — see also](183-oss-memory-landscape-may-2026.md). Multi-agent properties:

- **Leases / signals / actions / routines** — explicit coordination primitives.
- **Shared memory across Claude Code / Cursor / Gemini CLI / etc.** via MCP + REST.
- **12 hooks** auto-capture from any participating agent.

Limitation: trust assumption is shared-trust ("all agents in the cluster trust each other"). Not Byzantine-resistant.

### Mnema (no code yet)

[docs/186](186-mnema-witness-lattice.md). Witness-per-memory-item, Ed25519-signed journals, fixed 9-step protocol, the 1−α detection floor. The strongest *theoretical* candidate; not shipped.

---

## §5 — Multi-agent memory matrix

| System | Per-row signing | Attribution | Conflict resolution | Byzantine-tolerance | Maturity |
|---|---|---|---|---|---|
| mem0 | ❌ | scope (`user_id`/`session_id`/`agent_id`) | retrieval ranking | ❌ | production |
| Letta multi-agent | ❌ | tool-call trace | agent-decided | ❌ | production |
| Supermemory MCP | ❌ | `user_id` | API-side | ❌ | production (hosted) |
| agentmemory | ❌ | hooks + lease/signal | contradiction detection at write | ❌ | early production |
| A-MEM | ❌ | typed edges (`cites`, etc.) | dynamic rewrite | ❌ | research |
| **Mnema** | **Ed25519 per witness** | **per-witness journal** | **CONTRADICT verb + protocol** | **structural (1−α floor)** | **proposal (no code)** |
| CP-WBFT (when wired in) | ✅ | implicit by node | weighted BFT vote | **✅ to 85.7% fault** | research |

**Reading the matrix**: every shipping system is optimised for the cooperative-trust regime; only Mnema and the BFT papers explicitly engineer for the adversarial regime. The gap is fill-able with a small composition of existing primitives — see §7.

---

## §6 — The patterns to steal

Three load-bearing patterns from the threat / defense / shared-memory literature, in priority order.

### Pattern 1 — Sign every shared-memory write

**Source**: CPA (MemoryGraft response), Mnema, BFT papers. Mechanism: each agent holds a private signing key (Ed25519 for full PKI; HMAC for single-tenant clusters). Every write to shared memory is signed; every read verifies.

**Cost**: ~50µs/write (Ed25519 sign) + 100µs/read (verify); negligible against retrieval latency. Storage: 64 bytes/row signature + 32 bytes pubkey reference.

**Closes**: MemoryGraft (no valid signature for externally-injected experience), AgentPoison (backdoor row fails verification on insertion), PoisonedRAG-style mass injection (signature pubkey is restricted to authorised agents). Does **not** close: a compromised authorised agent. For that, see Pattern 2.

### Pattern 2 — Cross-family decorrelation when you stack redundancy

**Source**: Mnema's 1−α floor, "Can AI Agents Agree?" empirical pushback.

When you run N voters / N retrievers / N gates, *measure α*. If all N share a model family, training corpus, or ingestion path, the effective independence is far below N. The closed-form bound:

```
P_undetected = α + (1 − α) · β^(1 + q)   →   α as q → ∞
```

Engineering rule: **at least one cross-family voter** in any N-way consensus or critic-jury. This applies to Polaris's v2.2 P29 three-gate AND, Lyra's reasoning-bank tournament, and any agent-debate architecture.

### Pattern 3 — Refusal as a first-class emission

**Source**: Mnema witness refusal rights; A-MEM typed-edges (a missing edge is informative); A-MemGuard write-time gate.

When retrieval has nothing within confidence threshold, *emit a structured refusal* instead of returning low-confidence top-K. The agent can then choose escalation, fallback, or human-in-the-loop. This breaks the failure mode where retrieval always returns *something* and the model treats "something" as ground truth.

Concrete shape:

```python
@dataclass
class RefusalResult:
    query: str
    reason: Literal["no-domain-witness", "all-below-threshold",
                    "contradicting-witnesses", "vendor-degraded"]
    closest_score: float
    # Audit trail:
    candidates_inspected: int
    threshold_used: float
```

A retriever returns either `tuple[Result, ...]` or `RefusalResult`. The caller branches on type.

---

## §7 — Composing the patterns into a minimum viable adversarial-tolerant memory

Assemble the three patterns onto an existing single-agent memory:

1. **Substrate**: any of the [docs/183](183-oss-memory-landscape-may-2026.md) systems — start with Letta or mem0.
2. **Sign at write**: wrap the substrate's write API with an Ed25519 / HMAC signer (CPA pattern).
3. **Verify at read**: wrap the substrate's read API to drop unsigned / invalid-signature rows.
4. **Refusal type**: extend the read API to emit `RefusalResult` when no row passes confidence + signature.
5. **Decorrelation discipline**: when stacking N readers / voters, *enforce* at least one cross-family voter at the orchestration layer. Record the family choices in the trace.
6. **Optional BFT consensus** (for safety-critical systems): mediate writes via CP-WBFT or a Trusted-MultiLLMN-style weighted vote.

Layer 1–4 is ~200 lines of Python. Layer 5 is a discipline + a check at the orchestration layer. Layer 6 is the heavy hammer for systems where Pattern 1 is insufficient.

---

## §8 — Polaris adoption recipe

Polaris already has the substrate — `ProvenanceLedger` is single-agent CPA (signed JSONL is a config away). Three additions for v2.6+:

```text
projects/polaris/packages/polaris-core/src/polaris_core/memory/
  signed_provenance.py       # NEW [CPA / Mnema] — Ed25519 sign on append;
                             # verify on load. Composes with v2.5 P43.
  refusal_result.py          # NEW [Mnema] — RefusalResult emit when
                             # PPRRetriever has nothing within threshold.
  family_decorrelation.py    # NEW [Mnema 1−α] — annotates ClaimGate
                             # voters with model_family; flags α-high
                             # configurations at gate orchestration time.
```

Effort: ~3 weeks. Composes cleanly with the v2.5 memory stack ([docs/184 §5](184-strongest-memory-techniques-synthesis-may-2026.md)).

## §9 — Lyra adoption recipe

Lyra's threat model is local (single user, local agent), so full Ed25519 PKI is overkill. The minimum useful adoption:

```text
projects/lyra/packages/lyra-core/src/lyra_core/memory/
  hmac_auto_memory.py        # NEW — HMAC sign every auto_memory entry;
                             # verify on load; reject tampered rows.
  refusal_result.py          # NEW — emit when recall has nothing
                             # within threshold; surfaces in HIR trace.
```

Effort: ~1 week. Closes the *local-tampering* attack surface (an attacker who edits `~/.lyra/memory/*/entries.jsonl` cannot forge entries without the project key).

The 1−α decorrelation rule applies at MaTTS / tournament-TTS time: prefer cross-family voters in N-way tournaments. Document in v3.x; no code change today.

---

## §10 — What's missing from the landscape

Honest gaps in May 2026:

| Gap | Status |
|---|---|
| **A signed-by-default OSS memory framework** | None. Letta, mem0, Cognee, Zep, A-MEM all default to unsigned. Add as a feature request. |
| **Cross-family decorrelation built into orchestration frameworks** | None at the framework level. AutoGen / CrewAI / smolagents offer no primitive for "force one Claude voter, one Gemini voter, one GPT voter." |
| **Production-shipped Mnema-shape architecture** | None. Mnema is paper-only; A-MEM is the closest production-ready cousin and doesn't include the cryptographic layer. |
| **Memory-poisoning benchmark portable across systems** | Partial. Each attack paper bundles its own bench; A-MemGuard reports on three. No equivalent of MemoryBench for poisoning resistance. |
| **Replay tooling for joint multi-agent traces** | Partial. Letta replays per-agent traces; reconstructing the interleaving is manual. |

These gaps are the obvious places for new contributions over the next six months.

---

## §11 — One-paragraph summary

Multi-agent memory is the perpendicular axis to single-agent memory: same substrate decisions, different trust assumptions. The threat layer (PoisonedRAG, AgentPoison, MemoryGraft, Poison-Once-Exploit-Forever) has documented 50–90% recall corruption attacks; the defense layer (A-MemGuard, Cryptographic Provenance Attestation, Byzantine-fault-tolerant consensus) has matured to the point where structurally-decorrelated cross-family critic-juries with signed write paths can drive attack success rates near zero. The strongest convergent technique today is signed provenance + structural decorrelation + refusal-as-emission — three patterns that compose onto any of the [docs/183](183-oss-memory-landscape-may-2026.md) single-agent stacks in ~200 LOC. The closed-form 1−α detection floor from [Mnema (docs/186)](186-mnema-witness-lattice.md) names the limit: **redundancy alone never beats the shared-root-cause probability**; structural decorrelation does. Polaris should adopt signed-claim emission + family-decorrelation annotation in v2.6 (~3 weeks); Lyra should adopt HMAC-signed auto-memory entries (~1 week). The gaps in the landscape are obvious and fill-able: a signed-by-default OSS memory framework, cross-family voter primitives in orchestration frameworks, and a portable memory-poisoning benchmark are all worth shipping in the next six months.

---

## §12 — Reading list

### Attack papers
- [PoisonedRAG (arXiv:2402.07867)](https://arxiv.org/abs/2402.07867) — the prior that defined the threat.
- [AgentPoison (NeurIPS 2024)](https://openreview.net/forum?id=Y841BRW9rY) — first agent-memory backdoor.
- [MemoryGraft (arXiv:2512.16962)](https://arxiv.org/abs/2512.16962) — persistent compromise via experience retrieval.
- [Poison Once, Exploit Forever (arXiv:2604.02623)](https://arxiv.org/html/2604.02623) — environment-injected memory poisoning on web agents.
- [Memory Poisoning Attack and Defense (arXiv:2601.05504)](https://arxiv.org/abs/2601.05504) — survey + framework.

### Defense papers
- [A-MemGuard (arXiv:2510.02373)](https://arxiv.org/abs/2510.02373) — proactive defense framework.
- Cryptographic Provenance Attestation — discussed in the MemoryGraft response.

### BFT / consensus
- [CP-WBFT (arXiv:2511.10400)](https://arxiv.org/abs/2511.10400).
- [Byzantine-Robust Decentralized Coordination of LLM Agents (arXiv:2507.14928)](https://arxiv.org/abs/2507.14928).
- [Trusted MultiLLMN (arXiv:2505.05103)](https://arxiv.org/abs/2505.05103).
- [A BFT Approach towards AI Safety (arXiv:2504.14668)](https://arxiv.org/abs/2504.14668).
- [Can AI Agents Agree? (arXiv:2603.01213)](https://arxiv.org/abs/2603.01213).
- [FREE-MAD (arXiv:2509.11035)](https://arxiv.org/abs/2509.11035).

### Shared-memory architectures
- [A-MEM (arXiv:2502.12110)](https://arxiv.org/abs/2502.12110) + [WujiangXu/A-mem](https://github.com/WujiangXu/A-mem) + [a-mem-mcp-server](https://github.com/tobs-code/a-mem-mcp-server).
- [Letta](https://github.com/letta-ai/letta), [mem0](https://github.com/mem0ai/mem0), [Supermemory](https://github.com/supermemoryai/supermemory), [agentmemory](https://github.com/rohitg00/agentmemory).
- [Mnema](https://gentic.news/article/mnema-witness-lattice-living-memory-multi-agent-ai) — see [docs/186](186-mnema-witness-lattice.md).

### Sister chapters
- [docs/183 — OSS Memory Landscape](183-oss-memory-landscape-may-2026.md) — single-agent landscape.
- [docs/184 — Strongest Memory Techniques](184-strongest-memory-techniques-synthesis-may-2026.md) — the synthesis this extends.
- [docs/186 — Mnema](186-mnema-witness-lattice.md) — single-source absorption.
- [docs/188 — Witness/Provenance Memory Synthesis](188-witness-provenance-memory-techniques-synthesis.md) — the cross-paper synthesis.
- [docs/121 — Multi-Agent Coordination Patterns](121-multi-agent-coordination-patterns.md), [docs/123 — Robustness Fault Tolerance](123-robustness-fault-tolerance.md).
- [docs/35 — Malicious Intermediary Attacks](35-malicious-intermediary-attacks.md), [docs/49 — Agents of Chaos](49-agents-of-chaos-red-teaming.md), [docs/82 — PoisonedRAG](82-poisonedrag.md).
