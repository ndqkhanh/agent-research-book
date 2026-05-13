# 188 — Witness / Provenance Memory: Synthesis (May 2026)

**Scope.** Cross-paper + cross-repo synthesis of *provenance-aware* memory techniques: signed writes, structural decorrelation, refusal-as-emission, Byzantine-tolerant consensus, witness/journal architectures. Extends [docs/184](184-strongest-memory-techniques-synthesis-may-2026.md) with the sixth design-space axis — **trust assumption** — that the original synthesis left implicit. Sister chapters: [docs/186 — Mnema](186-mnema-witness-lattice.md) (single-source), [docs/187 — Multi-Agent Shared Memory Landscape](187-multi-agent-shared-memory-landscape.md) (the threat / defense / system landscape).

**One-paragraph thesis.** [Doc 184](184-strongest-memory-techniques-synthesis-may-2026.md) named five structural commitments — retrieval architecture is the binding ceiling, tiered beats flat, PPR-fusion is the strongest retrieval, temporal validity is non-optional, memory ops belong in the trace. Each assumed cooperative authorship. The **sixth commitment**, surfaced by the multi-agent + adversarial-input literature ([Mnema](186-mnema-witness-lattice.md), [PoisonedRAG / AgentPoison / MemoryGraft / A-MemGuard / CP-WBFT](187-multi-agent-shared-memory-landscape.md)), is **provenance is non-optional once shared memory or external content enters the write path**. The strongest convergent technique is *signed provenance + structural decorrelation + refusal-as-emission* — three patterns that compose onto any [docs/183](183-oss-memory-landscape-may-2026.md) substrate in ~200 LOC. The closed-form 1−α detection floor names the engineering limit on redundancy: *no amount of stacked voters beats the shared-root-cause probability α; only structural decorrelation across model families, training corpora, and ingestion paths does*. The 2026 production stack is the [doc 184 §3 stack](184-strongest-memory-techniques-synthesis-may-2026.md#§3) plus this layer when (a) shared memory crosses a trust boundary, (b) external content can enter the write path, or (c) replay must be cryptographically verifiable for compliance.

---

## §1 — The sixth axis: trust assumption

[Doc 184 §2](184-strongest-memory-techniques-synthesis-may-2026.md#§2) spans six axes (substrate, retrieval, tier, write, forget, mode). Multi-agent + adversarial-input concerns add a seventh axis whose granularity is binary in practice and ternary in design:

```
single-trusted-author ← honest-multi-author ← Byzantine-resistant
(mem0 single-user,    (mem0 agent_id scope,  (Mnema, CP-WBFT,
 Letta single-agent,   Letta multi-agent,    Trusted MultiLLMN,
 Cognee local,         Supermemory MCP,       A-MemGuard wired in,
 HippoRAG)              agentmemory leases)    CPA writers)
```

Every shipping system in [docs/183](183-oss-memory-landscape-may-2026.md) sits at *single-trusted-author* or *honest-multi-author*; only the [docs/187](187-multi-agent-shared-memory-landscape.md) literature reaches Byzantine-resistant. The choice of axis position is *the* load-bearing decision — it determines whether write-time signing, decorrelated voting, and refusal-as-emission are non-negotiable or overkill.

Decision rule:

| Trust regime | Axis position | Patterns to ship |
|---|---|---|
| Single user, local agent | single-trusted-author | None of §3; doc 184's stack as-is |
| Multi-user / multi-IDE / chat assistant | honest-multi-author | Pattern 1 (sign at write) + Pattern 3 (refusal) |
| Regulated / compliance / external content / multi-vendor agents | Byzantine-resistant | All three patterns + optional BFT consensus |

---

## §2 — The three convergent techniques

### Technique 1 — Signed provenance at the write path

**Source convergence**: Cryptographic Provenance Attestation (CPA, [docs/187 §3](187-multi-agent-shared-memory-landscape.md#§3)), Mnema witness signing ([docs/186 §2](186-mnema-witness-lattice.md#§2)), A-MemGuard write-time gate.

**Mechanism**. Every memory write attaches a cryptographic signature from the originating agent's private key. Verification is mandatory at read time. The substrate refuses unsigned / invalid-signature rows.

**Variants**:
- **Ed25519 signing** — Mnema default; ~50 µs/write, 64 B/row signature, 32 B pubkey reference. Asymmetric: an attacker without the private key cannot forge.
- **HMAC** — symmetric, adequate for single-tenant clusters; ~5 µs/write, 32 B/row tag. Cheaper but no public verifiability.
- **Witness-per-row signing** (Mnema) — generalises CPA from "one key per agent" to "one key per memory item". Heavier; only justified when individual rows need independent retraction / lifecycle.

**What it closes**: MemoryGraft (no valid signature for externally-injected experience), AgentPoison (backdoor row fails signature on insertion), PoisonedRAG (mass-injection limited to authorised pubkeys), local file tampering, replay forgery.

**What it doesn't close**: a compromised authorised agent. For that, see Technique 2.

**Where this lands in the canon**: Polaris's `ProvenanceLedger` is one config away from CPA — JSONL-shaped, append-only, schema-tolerant for an Ed25519 column. Lyra's `auto_memory` is the same shape with HMAC. See [docs/186 §10–§12](186-mnema-witness-lattice.md#§10).

### Technique 2 — Structural decorrelation across consensus / critic-jury

**Source convergence**: Mnema cross-family critic-jury ([docs/186 §5](186-mnema-witness-lattice.md#§5)), the 1−α detection floor (closed-form bound), CP-WBFT weighted vote, "Can AI Agents Agree?" empirical pushback against naive BFT.

**Mechanism**. When stacking redundancy (N voters / N retrievers / N gates), enforce that voters are decorrelated across **model family**, **training corpus**, and **ingestion path**. Record the decorrelation choices in the audit trail. Measure α (shared-root-cause probability) at orchestration time.

**The closed-form bound** ([docs/186 §6](186-mnema-witness-lattice.md#§6)):

```
P_undetected = α + (1 − α) · β^(1 + q)
              ↓
            α    as q → ∞
```

**Engineering rule**: at least one cross-family voter in any N-way consensus or critic-jury. Three GPT-4 judges with three different prompts have α near 1; one Claude + one Gemini + one GPT has α near 0.

**What it closes**: correlated-failure cascades that redundancy alone cannot fix. The empirically-verified failure shape behind [docs/82 PoisonedRAG](82-poisonedrag.md)'s ~90% success rate against shared-corpus retrievers — α is high because all the rerankers share training data.

**Where this lands in the canon**:
- Polaris v2.2 P29 three-gate AND (ClaimGate × AttributionAlignmentGate × TriangulationGate) is structurally decorrelation-shaped, but the decorrelation isn't *measured*. Annotation step + α-flag is a ~1-week change.
- Lyra's reasoning-bank tournament TTS ([docs/81](81-reasoningbank.md)) defaults to N candidates from one model. Document the rule; recommend cross-family voters when running ≥ 3-way tournaments.
- Skill-side: Polaris's [Argus router cross-encoder rerank](180-argus-skill-router-agent-design.md) is a one-voter step; if you stack a second reranker, make it cross-family.

### Technique 3 — Refusal as a first-class emission

**Source convergence**: Mnema structural refusal ([docs/186 §2](186-mnema-witness-lattice.md#§2)), A-MEM typed-edges (a missing edge is informative, [docs/187 §4](187-multi-agent-shared-memory-landscape.md#§4)), A-MemGuard write-time rejection.

**Mechanism**. Retrievers and gates emit either a result tuple *or* a structured refusal — never silent best-effort top-K when nothing meets the confidence threshold. Refusal carries a typed reason (`no-domain-witness`, `all-below-threshold`, `contradicting-witnesses`, `vendor-degraded`) and the audit trail (candidates inspected, threshold used).

**Concrete shape**:

```python
@dataclass(frozen=True)
class RefusalResult:
    query: str
    reason: Literal[
        "no-domain-witness",
        "all-below-threshold",
        "contradicting-witnesses",
        "vendor-degraded",
        "tombstoned",
    ]
    closest_score: float
    candidates_inspected: int
    threshold_used: float
```

The retriever's return type is `tuple[Result, ...] | RefusalResult`. The caller branches on type and decides escalation, fallback, or HITL — instead of treating low-confidence top-K as ground truth.

**What it closes**: the "retrieval always returns something" failure mode that turns weak signals into hallucinations. Composable with every retriever in [docs/183](183-oss-memory-landscape-may-2026.md).

**Where this lands in the canon**: extend Polaris's `PPRRetriever.retrieve` and Lyra's `AutoMemory.retrieve` to return the union type. ~1-day change per retriever.

---

## §3 — Updated design-space matrix (extends doc 184 §2)

[Doc 184 §2](184-strongest-memory-techniques-synthesis-may-2026.md#§2) named six axes. The seventh:

### Axis 7 — Trust assumption

```
single-trusted-author ← honest-multi-author ← Byzantine-resistant
(mem0 / Letta / Cognee   (mem0 agent_id /        (Mnema / CP-WBFT /
 single-user / Polaris    Letta multi-agent /     A-MemGuard / CPA writers /
 single-program)          Supermemory MCP /        signed-CPA on top of any §1)
                          agentmemory leases)
```

### Axis 8 — Provenance / signing

```
unsigned ← row-level metadata ← row-level signature ← witness-per-item Ed25519 lifecycle
(mem0,    (Letta tool-call      (CPA / signed-CPA   (Mnema)
 Letta,    trace,                wrapper,
 most)     ProvenanceLedger)     A-MemGuard hook)
```

### Axis 9 — Refusal semantics

```
always returns something ← graceful degradation ← typed RefusalResult
(legacy RAG, mem0,        (Letta agent decides,    (Mnema, A-MemGuard,
 Cognee)                   threshold-aware)         pattern 3 wrapper)
```

The 2026 strongest stack from [doc 184 §3](184-strongest-memory-techniques-synthesis-may-2026.md#§3) plus these axes:

- **Trust regime**: choose by deployment risk; set the upper bound on the other axes.
- **Provenance**: row-level signature minimum for any honest-multi-author or Byzantine-resistant deployment.
- **Refusal**: typed `RefusalResult` for any retrieval-driven gate.

---

## §4 — The 1−α floor as a design constraint

The most-stealable single result from the witness-lattice literature is the closed-form bound itself. It applies anywhere you stack redundancy:

| System component | What α is | Lower α how |
|---|---|---|
| Polaris v2.2 P29 three-gate AND | shared-model-family probability | enforce one cross-family judge |
| Lyra reasoning-bank N-way tournament | shared-prompt probability | rotate prompt templates per voter |
| HippoRAG 2 cross-encoder rerank stack | shared-encoder probability | mix dense + cross-encoder + LLM-judge |
| Argus router voter chain | shared-tier-2-embedder probability | add a BM25 voter alongside the embedding voter |
| Multi-agent debate | shared-base-model probability | force one Claude + one Gemini + one GPT voter |

**Rule of thumb**: when designing any N-way redundancy, ask "what's α?" If the answer is "all voters share substrate X", the effective N is much smaller than N — and `P_undetected` floor is much higher than the 1/N you'd hope for.

This is the canon's first quantitative rule for redundancy design. [docs/123 — Robustness Fault Tolerance](123-robustness-fault-tolerance.md) and [docs/49 — Agents of Chaos](49-agents-of-chaos-red-teaming.md) hinted at it empirically; Mnema names the closed form.

---

## §5 — Composability with the doc-184 stack

The three techniques compose cleanly onto the doc-184 strongest 2026 stack:

| Doc 184 layer | Plus provenance layer |
|---|---|
| Tier 0 hot context | unchanged |
| Tier 1 warm vector + BM25 + entity-link | + signed-CPA write wrapper; reject unsigned rows at read |
| Tier 2 cold temporal KG (Graphiti shape) | + witness-per-claim Ed25519 (Polaris, full Mnema) or HMAC (Lyra, lightweight) |
| Cross-cutting: memory ops as tool calls | unchanged; signature emission becomes part of the tool-call trace |
| Cross-cutting: temporal validity | unchanged; signed retraction strengthens the audit chain |
| **NEW Cross-cutting: signed provenance** | per Technique 1 |
| **NEW Cross-cutting: structural decorrelation** | per Technique 2 — measure α at every redundancy point |
| **NEW Cross-cutting: refusal-as-emission** | per Technique 3 — typed `RefusalResult` from every retriever |

No conflict with PPR fusion + scope stratification + Cognee `improve()` + Letta tool-call writes. The provenance layer is *additive*.

---

## §6 — Polaris adoption recipe (v2.6 candidate)

Polaris already has the substrate (`ProvenanceLedger`, append-only JSONL, [v2.5 P43 temporal validity](../projects/polaris/POLARIS_V2_5_MEMORY_PLAN.md)). Three new files for v2.6:

```text
projects/polaris/packages/polaris-core/src/polaris_core/memory/
  signed_provenance.py        # NEW [Technique 1] — Ed25519 signing on
                              # Claim emit; verify on load. Composes with
                              # v2.5 temporal validity. ~5 days.
  refusal_result.py           # NEW [Technique 3] — RefusalResult union
                              # type from PPRRetriever.retrieve when no
                              # claim within threshold. ~3 days.
  family_decorrelation.py     # NEW [Technique 2] — annotates ClaimGate
                              # voters with model_family; emits HIR event
                              # with α-estimate when α > 0.5. ~5 days.
```

Bright-lines added:
- `BL-CLAIM-SIGNED` — every persisted claim carries a valid Ed25519 signature.
- `BL-RETRIEVAL-DECORRELATED` — N-way voter stacks must include ≥ 1 cross-family voter.
- `BL-RETRIEVAL-REFUSAL-TYPED` — retrievers return either result tuple or `RefusalResult`; never a low-confidence pseudo-result.

Effort: ~3 weeks. Composes with the v2.5 memory stack ([docs/184 §5](184-strongest-memory-techniques-synthesis-may-2026.md#§5)).

---

## §7 — Lyra adoption recipe (v3.9 candidate)

Lyra's threat model is local-first; full Ed25519 PKI is overkill. The minimum useful adoption:

```text
projects/lyra/packages/lyra-core/src/lyra_core/memory/
  hmac_auto_memory.py         # NEW [Technique 1, lightweight] — HMAC
                              # signature on every auto_memory entry;
                              # verify on load; reject tampered rows.
                              # ~3 days.
  refusal_result.py           # NEW [Technique 3] — emit when recall has
                              # nothing within threshold; surfaces in HIR
                              # trace. ~2 days.
```

Plus the documentation discipline:
- v3.x doc note recommending cross-family voters when running ≥ 3-way reasoning-bank tournaments. No code change.

Effort: ~1 week.

The v3.8 memory work ([docs/185 §2](185-memory-integration-playbook.md#§2)) — mem0 adapter, Cognee adapter, PPR fusion, Letta toolset — runs in parallel. Provenance layer can land first or last; no ordering dependency.

---

## §8 — What NOT to do

Three traps the convergent literature surfaces:

### Don't mistake "honest-multi-author" for "Byzantine-resistant"

mem0's `agent_id` scope, Letta's multi-agent shared archival, Supermemory MCP cross-IDE — these are all *namespaced multi-tenancy*, not Byzantine-resistance. They protect against accidental cross-contamination (agent A reading agent B's data), not against an adversarial agent A poisoning the shared store. The docs are clear; deployments often miss the distinction.

### Don't stack redundancy without measuring α

Five GPT-4o voters with five different prompts have α ≈ 1. The effective N is ~1, the `P_undetected` floor is high. Five voters across five model families have α near 0; the effective N is real. The instinct to "just add more judges" without recording the model-family / training-corpus / prompt-template mix is the trap.

### Don't sign without verifying

Signing the write path without verifying at the read path is theatre. Make verification mandatory; reject unsigned / invalid-signature rows. Every shipped agent stack today does the first half (audit trails). Few do the second half (refusing unsigned reads).

---

## §9 — Where this leaves the synthesis

Memory in May 2026 has matured enough that the *single-agent* stack is converged ([doc 184 §3](184-strongest-memory-techniques-synthesis-may-2026.md#§3)). The *multi-agent + adversarial-input* stack is still being shipped — Mnema is paper-only, A-MemGuard is research, CP-WBFT is one paper, A-MEM is the closest-to-production cousin and doesn't ship signing. The patterns are convergent across the literature; no system bundles all three yet. **The next six months will see the first OSS memory framework that signs every write by default and emits typed refusals.** The harness that ships first wins the regulated-deployment market segment.

The 1−α floor is the most-quotable single result of the year in agent memory. It generalises beyond memory — wherever redundancy meets correlated failure (model ensembles, multi-encoder rerank, multi-prompt critic-jury), the bound applies. Treat it as a *design rule*, not a curiosity.

---

## §10 — One-paragraph summary

Doc 184's five structural commitments assumed cooperative authorship; the multi-agent + adversarial-input literature ([Mnema](186-mnema-witness-lattice.md), [docs/187](187-multi-agent-shared-memory-landscape.md)) adds a sixth: **provenance is non-optional once shared memory or external content enters the write path**. Three convergent techniques — signed provenance at write, structural decorrelation across consensus voters, typed refusal-as-emission — compose onto any [docs/183](183-oss-memory-landscape-may-2026.md) substrate in ~200 LOC and close the bulk of the documented threat surface (50–90% recall corruption under PoisonedRAG / AgentPoison / MemoryGraft drives near zero with A-MemGuard-shape gates). The closed-form 1−α detection floor — `P_undetected = α + (1−α)·β^(1+q) → α as q → ∞` — names the design rule: redundancy alone never beats the shared-root-cause probability; structural decorrelation across model families / training corpora / ingestion paths does. Polaris's adoption ships in v2.6 (~3 weeks: signed claims + refusal type + family-decorrelation annotation); Lyra's adoption ships in v3.9 (~1 week: HMAC-signed auto-memory + refusal type + tournament-decorrelation discipline). Net payoff: an audit-grade, replay-verifiable, provenance-attested memory layer that composes with the doc-184 stack and closes the regulatory + adversarial-input gap.

---

## §11 — Reading list

### Sister chapters
1. [docs/186 — Mnema](186-mnema-witness-lattice.md) — the single-source absorption.
2. [docs/187 — Multi-Agent Shared Memory Landscape](187-multi-agent-shared-memory-landscape.md) — the threat / defense / system landscape.
3. [docs/184 — Strongest Memory Techniques Synthesis](184-strongest-memory-techniques-synthesis-may-2026.md) — the synthesis this extends.

### Threat / defense canon
4. [docs/82 — PoisonedRAG](82-poisonedrag.md), [docs/35 — Malicious Intermediary Attacks](35-malicious-intermediary-attacks.md), [docs/49 — Agents of Chaos](49-agents-of-chaos-red-teaming.md).
5. [docs/121 — Multi-Agent Coordination Patterns](121-multi-agent-coordination-patterns.md), [docs/123 — Robustness Fault Tolerance](123-robustness-fault-tolerance.md).
6. [docs/175 — Agent Skills Ecosystem & Security](175-agent-skills-ecosystem-and-security.md) — sister synthesis on the skill side.

### Frontier papers (post-canon)
7. [arXiv:2502.12110 — A-MEM](https://arxiv.org/abs/2502.12110) (NeurIPS 2025).
8. [arXiv:2510.02373 — A-MemGuard](https://arxiv.org/abs/2510.02373).
9. [arXiv:2511.10400 — CP-WBFT](https://arxiv.org/abs/2511.10400).
10. [arXiv:2512.16962 — MemoryGraft](https://arxiv.org/abs/2512.16962), [arXiv:2604.02623 — Poison Once Exploit Forever](https://arxiv.org/html/2604.02623), [arXiv:2601.05504 — Memory Poisoning Attack and Defense](https://arxiv.org/abs/2601.05504).
11. [arXiv:2507.14928 — Byzantine-Robust Decentralized Coordination of LLM Agents](https://arxiv.org/abs/2507.14928).
12. [arXiv:2603.01213 — Can AI Agents Agree?](https://arxiv.org/abs/2603.01213).

### Single-agent canon (provenance-aware reading order)
13. [docs/100 — Contextual Memory Is a Memo](100-contextual-memory-is-a-memo.md).
14. [docs/151-153 MEMTIER](151-memtier-why-flat-memory-breaks-at-72-hours.md).
15. [docs/106-109 Memento](106-memento-paper-theory.md), [docs/79 Skill-RAG](79-skill-rag.md), [docs/81 ReasoningBank](81-reasoningbank.md).
16. [docs/181 — Mem0 Deep Dive](181-mem0-deep-dive.md), [docs/182 — Memory Frontiers](182-memory-frontiers-2026.md), [docs/183 — OSS Memory Landscape](183-oss-memory-landscape-may-2026.md).

### Implementation
17. [docs/185 — Memory Integration Playbook](185-memory-integration-playbook.md) — the existing integration recipe.
18. [docs/172 — Polaris 2026 Roadmap](172-polaris-2026-deep-research-roadmap.md), [Polaris v2.5 P42-P44](../projects/polaris/POLARIS_V2_5_MEMORY_PLAN.md) — the existing memory tier.
19. [docs/180 — Argus](180-argus-skill-router-agent-design.md) — sister system design that already uses signed bright-lines.
