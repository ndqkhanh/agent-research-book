# 186 — Mnema / Witness Lattice: Living Memory for Multi-Agent AI

**Source**: [gentic.news/article/mnema-witness-lattice-living-memory-multi-agent-ai](https://gentic.news/article/mnema-witness-lattice-living-memory-multi-agent-ai) · EUMAS 2026 single-blind submission · Springer LNCS 15-page paper.

**Scope.** Single-source absorption of the Mnema architecture — the first published multi-agent memory design that treats individual memory items as **autonomous cryptographic agents** with their own keys, journals, and refusal rights, rather than passive rows in a store. Mnema is *speculative* (no code, pre-registration only, no benchmarks yet) but the structural argument is load-bearing for any system that needs replayability + attributability + poisoning-resistance simultaneously. This chapter folds the architecture into the canon and identifies what Polaris and Lyra should steal.

**One-paragraph thesis.** Every memory system in [docs/183](183-oss-memory-landscape-may-2026.md) — mem0, Letta, Zep/Graphiti, Cognee, HippoRAG 2 — assumes *cooperative authorship*: the agent writes, the store accepts, retrieval ranks. Mnema rejects this assumption. Three documented failure modes are framed as one problem: coordination collapse (41–86.7% across seven SOTA frameworks), memory poisoning (50–90% corrupted recall under MemoryGraft / AgentPoison / PoisonedRAG), and unauditable decisions under EU AI Act / HIPAA / SOC 2. The Mnema thesis is that **passive storage + opaque learned orchestration cannot deliver replayability + attributability + poisoning resistance jointly**. The proposed substitute: replace the store with **living witnesses** (Ed25519-keyed memory units with signed journals and structural refusal capacity), replace the orchestrator with a **fixed nine-step signed protocol** (cross-family critic-jury, constitutional gates, deterministic commitment), and bound redundancy effectiveness with a **closed-form detection floor** — `P_undetected = α + (1−α)·β^(1+q)` — that pins detection at 1−α regardless of how much redundancy you add. Trade-off: protocols are less expressive than learned policies, but verifiability is what regulated deployments need. The structural argument is correct even if the specific architecture turns out not to ship.

---

## §1 — The five-layer architecture

Mnema decomposes into five layers, top to bottom:

| Layer | Role |
|---|---|
| **Substrate** | The underlying data layer — vectors, graphs, raw evidence — that witnesses ingest from. Not exposed to higher layers directly. |
| **Witness** | Autonomous cryptographic memory units. Each carries an Ed25519 public key, a hash-chained signed journal of every event in its life, and *structural capacity to refuse queries outside its domain*. |
| **Lattice** | The evolving knowledge graph in which witnesses interact. Lineage edges (ancestor / descendant), agreement edges (corroboration), conflict edges (contradiction). |
| **Protocol** | A fixed nine-step signed pipeline for every decision. Not learned, not adaptive — verifiable by replay. |
| **Audit** | The cryptographic replay trail. Every `COMMIT` / `ESCALATE` / `DEFER` / `DIE` decision can be re-executed deterministically from the journals. |

The architectural inversion: **memory items are not stored, they are *witnessed into existence*** — a witness is born when peers converge on fresh evidence, not when a row is inserted.

## §2 — The witness as autonomous agent

Each witness ships with three structural properties that make it more than a row:

1. **Cryptographic identity.** Ed25519 public/private key pair. Every emission is signed; verification is cheap; replay is deterministic.

2. **Hash-chained signed journal.** Every event in the witness's life — birth, ingestion, assertion, contradiction, retirement — is appended to a tamper-evident chain. The journal is the *primary* artefact; the substrate is recoverable from the journal but not vice versa. This mirrors the append-only-JSONL pattern that Polaris already uses for `ProvenanceLedger` (see [docs/172](172-polaris-2026-deep-research-roadmap.md)) but lifts it from one log per program to one log per memory item.

3. **Structural refusal capacity.** A witness has a *domain*; queries outside that domain produce signed refusals, not hallucinated answers. This is the architectural counterweight to mem0's "single-pass ADD-only" assumption that any retrieval should return *something*.

A `BIRTH` entry seeds the journal with the lineage edges to the witnesses that gossiped it into existence — provenance is structural, not annotation.

## §3 — The grammar

Witness-to-witness interaction operates over a fixed six-verb vocabulary:

```
ASSERT       — claim a fact within domain
CORROBORATE  — confirm another witness's assertion
CONTRADICT   — flag conflict; emits a signed disagreement
PROPOSE      — offer a candidate fact for adoption
ASSENT       — vote yes on a proposal
DISSENT      — vote no on a proposal
```

Every emission is signed; every receipt is journalled. The grammar is intentionally narrow — Mnema's argument is that *expressive* protocols learn-to-deceive more easily than fixed grammars, and verifiability scales with protocol simplicity.

## §4 — The lifecycle

A witness is not a static record. It moves through five stages:

```
EMBRYONIC → JUVENILE → ADULT → ELDER → PHANTOM
```

Plus four operational transitions:

| Op | When | Effect |
|---|---|---|
| **BIRTH** | Peers converge on fresh evidence | New witness instantiated with lineage edges |
| **SPLIT** | High action variance — the witness disagrees with itself across queries | Partition into children; parent retires |
| **COALESCE** | Two witnesses share canonical claims with non-contradictory journals | Merge into one |
| **PROBE** | Within budget, witness queries substrate for fresh ingestion candidates | Adds to journal; may emit `PROPOSE` |
| **RETIRE** | Lifecycle stage threshold met | Five-stage progression to PHANTOM |

A `PHANTOM` witness still exists in the journal but emits *signed refusal pointing to its successor* on every query. The retraction path is structural — there's no "delete" operation, only a downgrade to PHANTOM that preserves the audit chain.

This is conceptually identical to Polaris's `RED-RETRACTED` tier ([docs/172 §3 Gap 5](172-polaris-2026-deep-research-roadmap.md)) but generalised: every memory item in Mnema is a candidate for retraction, not just citations.

## §5 — The nine-step decision protocol

Every decision (whether to admit a fact, contradict an assertion, escalate to a human) runs through the same fixed pipeline:

1. **Activation** — does the query fall within any active witness's domain?
2. **Speak-or-refuse** — each candidate witness signs either a contribution or a structured refusal.
3. **Cross-family critic-jury** — three different model families review the contributions independently. (The point: structural decorrelation across model lineages, not redundancy with the same model.)
4. **Constitutional gate** — does the proposed action violate the system's bright-line rules?
5. **Veto-council gate** — pre-registered veto holders can block.
6. **Optional debate** — bounded multi-round exchange when contributions conflict.
7. **Saga-style execution** — long-running actions decompose into atomic sub-steps, each commit-able / revert-able independently.
8. **Deterministic commitment function** — emits one of `COMMIT` / `ESCALATE` / `DEFER` / `DIE`.
9. **Journal close** — every step is signed and appended.

Steps 3–4 are the load-bearing novelty: **redundancy is not enough; you need structural decorrelation**. Three GPT-4o votes don't beat one GPT-4o + one Claude + one Gemini, even at higher count, because the failure modes correlate. This is where the 1−α floor (next section) bites.

## §6 — The 1−α detection floor

The single most quotable result in the paper. A closed-form bound on the probability that a poisoned memory remains undetected after q redundant copies:

```
P_undetected = α + (1 − α) · β^(1 + q)
```

Where:
- **α** = probability that the redundant copies share a *root cause* (same model family, same training corpus, same ingestion path).
- **β** = residual per-copy corruption rate after a single check.
- **q** = redundancy factor.

As `q → ∞`, the term `(1 − α) · β^(1 + q)` collapses to zero, leaving:

```
lim_{q→∞} P_undetected = α
```

**Engineering consequence**: detection floor is pinned at α. **No amount of redundancy beats it.** If two-thirds of your "independent" voters are GPT-4-family models trained on overlapping corpora, your α is high and your effective detection ceiling is low — regardless of how many voters you stack.

The Mnema schema is designed to *record* the structural-decorrelation choices (model family, source-graph root, ingestion path) in the witness journal so an auditor can verify α at any point.

This formalises a result the canon has hinted at without proving — see [docs/82 PoisonedRAG](82-poisonedrag.md) (mass-injection attacks succeed at near-90% on shared-corpus retrievers), [docs/49 agents-of-chaos](49-agents-of-chaos-red-teaming.md) (cascade failures correlate by model family), [docs/35 malicious-intermediary-attacks](35-malicious-intermediary-attacks.md). The 1−α floor is the closed-form behind those empirical findings.

## §7 — What problems Mnema targets

Three failure modes Mnema names explicitly:

| Failure | Documented rate | How Mnema addresses |
|---|---|---|
| **Coordination collapse** | 41–86.7% across seven SOTA frameworks (per the paper's review of MAS-Bench / MAST) | Fixed protocol replaces learned orchestration; deterministic commitment function |
| **Memory poisoning** | 50–90% corrupted recall under MemoryGraft, AgentPoison, PoisonedRAG | Signed journals + cross-family critic-jury + 1−α-aware redundancy |
| **Unauditable decisions** | EU AI Act / HIPAA / SOC 2 violations | Cryptographic replay trail; every step signed; journals reproducible |

The argument's force is that these *aren't three problems* — they're one problem under three lenses. Coordination collapses *because* memory poisoning leaks into shared state *because* nobody can replay to find the bad commit. Mnema's claim is that solving one without the others doesn't ship.

## §8 — What's load-bearing vs speculative

Mnema is a EUMAS 2026 single-blind submission. As of May 2026:

| Element | Load-bearing | Speculative |
|---|---|---|
| The 1−α detection floor formula | ✅ — closed-form, verifiable | — |
| The five-layer architecture | ✅ as a *taxonomy* | speculative as a *shipping system* |
| Witness lifecycle + grammar | ✅ as a *discipline* | speculative as a *contract* — no code, no benchmarks |
| Cross-family critic-jury | ✅ — established result (debate / multi-juror) | the specific 3-family choice is a knob |
| Pre-registered demonstration (n=40, α=0.05, β=0.20) | ✅ — pre-registration is the right discipline | the specific demonstration hasn't shipped results |

What you *cannot* steal directly:
- No GitHub repo. No reference implementation. No benchmark numbers.
- The full Springer LNCS paper exists at the link's `/papers/mnema/mnema_eumas2026.pdf`; treat results as forthcoming.

What you *can* steal:
- The 1−α floor as a design constraint on any retrieval+redundancy system.
- Signed-journal-per-memory-item as a generalisation of provenance-per-claim.
- Structural decorrelation across model families as a critic-jury discipline.
- Refusal-as-a-first-class-emission instead of degraded-confidence retrieval.

## §9 — Comparison to mem0 / Letta / Zep / A-MEM / MemGPT

Mnema's framing of the orthodoxy: *those systems treat memory as something the agent stores — a record, a note, a vector, a graph node. Mnema inverts this — memory items become active agents with refusal rights, not inert storage.*

| System | Memory model | Provenance | Refusal | Replay |
|---|---|---|---|---|
| **mem0** | scope-stratified rows | row-level metadata | ❌ | ❌ |
| **Letta** | self-managed tiers | tool-call trace | partial (agent decides) | tool-call trace replay |
| **Zep / Graphiti** | temporal KG | temporal-validity windows | implicit (out-of-window) | event stream replay |
| **A-MEM** ([arXiv 2502.12110](https://arxiv.org/abs/2502.12110), NeurIPS 2025) | Zettelkasten-linked notes with dynamic indexing | linkage edges | ❌ | partial |
| **MemGPT** | OS-paging metaphor | page-level | ❌ | ❌ |
| **Mnema** | Ed25519-keyed witnesses with signed journals | per-witness hash chain | **first-class structural** | **deterministic** |

Mnema's claim isn't that the others are wrong — it's that they're *insufficient* for the deployments where regulators ask "show me why the model said that, and prove it wasn't tampered with." Single-agent assistants don't need this. Multi-agent regulated systems do.

## §10 — What Polaris should steal

Polaris already has the audit substrate (`ProvenanceLedger`, JSONL append-only, [P43 temporal validity](../projects/polaris/POLARIS_V2_5_MEMORY_PLAN.md)). Three Mnema patterns that compose cleanly:

1. **Sign every claim row.** Extend `Claim` with an Ed25519 signature field; every emit signs with the program's key. Cheap (one ed25519 per write), audit-grade (replay verifies). Bright-line: `BL-CLAIM-SIGNED`.
2. **Cross-family critic-jury for ClaimGate.** The v2.2 P29 three-gate AND already does ClaimGate × AttributionAlignmentGate × TriangulationGate. Add: when the three gates disagree, *the critic-jury's structural decorrelation matters*. If all three judges are GPT-4 family, the disagreement signal is weaker than if one is Claude, one is Gemini, one is GPT. Plumb the model family into the gate's record.
3. **Refusal-as-first-class.** When the retriever has no claim within `top_k * threshold` of the query, emit a `RefusalResult` instead of degraded-confidence top-K. The agent then decides whether to escalate, not infer from a low score.

The 1−α floor itself is a *constraint*, not a feature: when designing the v2.2 cross-skill consolidation gate, the redundancy budget should be spent on independent gates (different model families, different prompts, different evidence sources) rather than re-running the same gate.

## §11 — What Lyra should steal

Lyra's surface is smaller (single user, single agent), so the full Mnema overhead doesn't pay off. But two patterns transfer:

1. **Signed `auto_memory` entries.** When the agent writes a `feedback` / `user` entry, sign it with the project's local key (not Ed25519 — HMAC is enough for single-user). Detects local file tampering. Light footprint.
2. **The 1−α discipline at consensus time.** Lyra's tournament TTS ([reasoning_bank MaTTS](81-reasoningbank.md)) uses N candidate trajectories. If the candidates all run the same model with the same prompt, α is high and the tournament discriminates less than its sample count suggests. Document this in the v3.x docs and recommend at-least-one-cross-family voter when running N-way tournaments.

Mnema's full witness lifecycle (BIRTH/SPLIT/COALESCE/PHANTOM) is overkill for Lyra; the discipline is the steal, not the implementation.

## §12 — Integration slots

```text
projects/polaris/packages/polaris-core/src/polaris_core/memory/
  signed_provenance.py      # NEW — Ed25519 signing on Claim emit;
                            # composes with v2.5 P43 temporal validity.
  refusal_result.py         # NEW — RefusalResult emit when retrieval
                            # falls below confidence threshold.
                            # PPRRetriever.retrieve returns either
                            # tuple[RetrievalResult, ...] or RefusalResult.
  critic_jury_decorrelation.py  # NEW — annotates ClaimGate verdicts
                                # with model_family field; flags α-high
                                # configurations.

projects/lyra/packages/lyra-core/src/lyra_core/memory/
  signed_auto_memory.py     # NEW — HMAC signature on every entry
                            # write; verify on load.
```

Estimated effort: ~3 weeks for Polaris signed-provenance + refusal + decorrelation; ~1 week for Lyra signed entries. Net payoff: a measurable α-floor on the v2.2 P29 gate stack.

## §13 — Reading list

In recommended order:

### The Mnema source

1. [gentic.news article](https://gentic.news/article/mnema-witness-lattice-living-memory-multi-agent-ai) — the public summary.
2. The full LNCS paper at `/papers/mnema/mnema_eumas2026.pdf` (link off the article) — 15 pages, EUMAS 2026 single-blind submission.

### The threat-model priors Mnema reacts to

3. [docs/82 — PoisonedRAG](82-poisonedrag.md) — mass-injection attacks on retrieval.
4. [arXiv:2407.12784 — AgentPoison](https://openreview.net/forum?id=Y841BRW9rY) — first backdoor attack targeting agent memory + RAG knowledge bases (NeurIPS 2024).
5. [arXiv:2512.16962 — MemoryGraft](https://arxiv.org/abs/2512.16962) — persistent compromise via poisoned experience retrieval.
6. [docs/35 — Malicious Intermediary Attacks](35-malicious-intermediary-attacks.md), [docs/49 — Agents of Chaos](49-agents-of-chaos-red-teaming.md).

### The defense / consensus priors

7. [arXiv:2510.02373 — A-MemGuard](https://arxiv.org/abs/2510.02373) — proactive defense for LLM-agent memory.
8. [arXiv:2511.10400 — CP-WBFT](https://arxiv.org/abs/2511.10400) — Byzantine-fault-tolerant consensus for LLM agents.
9. [arXiv:2507.14928 — Byzantine-Robust Decentralized Coordination of LLM Agents](https://arxiv.org/abs/2507.14928).

### The single-agent memory canon Mnema diverges from

10. [docs/181 — Mem0](181-mem0-deep-dive.md), [docs/183 — OSS Memory Landscape](183-oss-memory-landscape-may-2026.md), [docs/184 — Strongest Memory Techniques Synthesis](184-strongest-memory-techniques-synthesis-may-2026.md).
11. [arXiv:2502.12110 — A-MEM](https://arxiv.org/abs/2502.12110) — Zettelkasten-inspired single-agent memory (NeurIPS 2025).

### Sister chapters

12. [docs/187 — Multi-Agent Shared Memory Landscape](187-multi-agent-shared-memory-landscape.md) — the broader landscape this absorbs from.
13. [docs/188 — Witness/Provenance Memory Synthesis](188-witness-provenance-memory-techniques-synthesis.md) — folds Mnema into doc 184's design-space matrix.

---

## §14 — One-paragraph summary

Mnema is the first published multi-agent memory architecture that treats memory items as autonomous Ed25519-keyed witnesses with signed hash-chained journals, structural refusal capacity, and a fixed nine-step decision protocol. Its central contribution is a closed-form detection floor — `P_undetected = α + (1−α)·β^(1+q)` — that bounds the effectiveness of any redundancy strategy by the shared-root-cause probability α, regardless of how many copies you stack. The system frames coordination collapse, memory poisoning, and unauditability as one problem (passive storage + opaque orchestration) and proposes the inversion (active witnesses + fixed protocol). No code, no benchmarks yet — EUMAS 2026 single-blind submission with a pre-registered demonstration. The structural argument is correct even if this specific architecture doesn't ship: any system aiming for replayability + attributability + poisoning-resistance jointly will end up with some shape of signed journal + cross-family critic-jury + structural refusal. Polaris should steal signed claims + cross-family decorrelation + refusal-as-emission; Lyra should steal HMAC-signed auto-memory entries + the 1−α discipline at tournament-TTS time. The one-line takeaway: **redundancy is a trap without structural decorrelation, and α is what auditors actually verify**.
