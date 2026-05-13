# 100 — Contextual Agentic Memory Is a Memo, Not True Memory

**Paper.** Binyan Xu*, Xilin Dai*, Kehuan Zhang — *Contextual Agentic Memory is a Memo, Not True Memory* — arXiv:2604.27707v1 — The Chinese University of Hong Kong; Zhejiang University — April 30, 2026 (*equal contribution).

**One-line definition.** Every deployed agentic memory system today (MemGPT, Generative Agents, Reflexion, Voyager, MemoryBank, A-MEM, mem0) implements **C-engineering** — adding to the model's *context* — not **θ-engineering** — updating its *weights*; a Compositional Sample Complexity Separation theorem proves an Ω(k²) vs. O(d) lower bound that no amount of clever retrieval, re-ranking, or context-window expansion can close, so the field has been *engineering filing cabinets while calling them brains*.

## Why this paper matters

Three years of memory papers have stacked retrieval tiers on top of frozen models — and benchmarked progress almost entirely on **recall**: did the agent fetch the right note? The implicit assumption is that *better notes* eventually become *better learning*. Xu, Dai, and Zhang formalize what that assumption costs. Their core construction (a Fano-inequality argument on a composition operator with bounded in-context accuracy) shows there exist task families where any retrieval-only memory must store Ω(k²) examples to reach a target error rate, while a parametric model needs O(d) — with d ≪ k² for any non-trivial concept algebra. The gap is **context-window-independent**: doubling the window only nudges in-context accuracy ᾱ; it does not erase the coverage requirement.

Three downstream consequences fall out immediately. The **frozen-novice problem**: every session begins with identical pre-trained weights, so an agent running Reflexion for a year is still doing `predict(C)` and never `train(θ)` — the filing cabinet grows, the cognition does not. The **persistent-compromise problem**: stateless agents bound per-session injection probability at p₀, but agents with persistent retrieval-based memory accumulate compromise as P_t = 1 − (1 − p₀)^N(t) → 1; MINJA hit 98.2% injection success, PoisonedRAG flipped million-document RAGs with five crafted texts. The **expertise-organization problem** from Chi et al. (1981): expertise emerges from *structural reorganization* of knowledge, which biological systems achieve through neocortical consolidation during sleep — a weight change. Sleep-time-compute proposals that rewrite *context* tokens still don't touch weights.

Take this paper seriously and three things change at once. (1) Benchmarks must measure **compositional generalization over time**, not recall@k. (2) Harness designers must add a *consolidation pathway* from episodic store to weights — the AI analog of sleep — alongside the fast retrieval tier. (3) Security threat models must distinguish C-attacks (injection, poisoning) from θ-attacks (training-time access), because retrieval-based persistence converts the former into the latter for free.

## Problem it solves

Five concrete failures of current designs that the paper formalizes:

1. **Coverage blow-up.** A retrieval system that has seen `(f_i, f_j) → ⊕(f_i, f_j)` for some pairs cannot answer compositionally novel pairs better than the frozen base model's bounded in-context accuracy ᾱ < 1. Achieving 1 − δ accuracy needs Ω(k²) stored compositional examples.
2. **Frozen novice.** No matter how rich the episodic store grows, the agent never reorganizes its representations — surface-feature categorization persists where experts would categorize by deep structural principles.
3. **Capacity mismatch.** Tasks needing integration of m > K mutually dependent facts fail even without compositional novelty (the "lost in the middle" regime; Liu et al. 2023; Paulsen 2026 finds effective utilization saturates at ~20K tokens even on 128K-context models).
4. **Persistent compromise.** Retrieval-augmented agents convert one-shot prompt injections into permanent attack surface; bounded transient compromise becomes unbounded persistent compromise.
5. **Architectural confusion.** The community labels memory by what it *appears* to do (working / episodic / semantic / experiential) without auditing where storage actually lives. Almost every system is "episodic" by storage type even if marketed as semantic — a category error that hides the lack of θ-learning.

## Core idea in one paragraph

Memory in cognitive systems is not a single mechanism but a layered architecture spanning a *compression spectrum*: low-compression, high-fidelity traces (episodic) at one end; high-compression, generalizable rules (parametric) at the other. Current AI systems implement only the low-compression end and call it memory; the rule end requires changes to model weights. A compositional task — one whose answers depend on combining base concepts the model has not jointly seen — exposes this asymmetry sharply: the retrieval system either has the answer in its store (and trivially returns it) or doesn't (and falls back on the frozen model's bounded ICL accuracy). There is no third option. Building a memory architecture that achieves true expertise therefore requires adding a *consolidation channel* that periodically distills accumulated experience into weights, asynchronously, like sleep — fast retrieval for the moment, slow consolidation for durable rules.

## Mechanism (step by step)

### Formalism

Let F = {f₁, …, f_k} be base concepts in a domain and ⊕ : F × F → Y a composition operator producing outputs in Y. An agent observes a dataset D of n labeled examples (f_i, f_j, ⊕(f_i, f_j)). A pair (f_i, f_j) is **compositionally novel** w.r.t. D if no entry in D has output ⊕(f_i, f_j).

Two memory architectures sit at the extremes:
- **M_R** — retrieval-based: store (compressed) examples; at query time, retrieve top-k and condition the *frozen* base model on them in-context. Weights θ unchanged.
- **M_P** — parametric: convert experience into weight updates via SFT, RLHF, knowledge editing, or self-distillation. Weights θ change.

### Bounded in-context composition (Assumption 3.2)

The frozen model f_{θ_frozen}, given K demonstrations of ⊕ from D, achieves accuracy α(K) ≤ ᾱ < 1 on held-out compositionally novel pairs. The paper proves this is a theorem (Fano's inequality) for any operator class with log|H| > K log|Y|: when the hypothesis space H is larger than what K demonstrations can identify, error is bounded below regardless of how powerful the base model is in-context.

### Theorem 1 (Compositional Sample Complexity Separation)

Under Assumption 3.2, for target error 1 − δ with δ < 1 − ᾱ:

- **Retrieval lower bound.** Any M_R with frozen weights requires
  n_R  ≥  ((1 − δ − ᾱ) / (1 − ᾱ)) · (k² / 2)  =  Ω(k²)
  stored compositional examples. The proof is direct: for stored pairs, retrieval answers correctly; for the (1 − n/N) novel-pair mass, accuracy is at most ᾱ, so the convex combination gives CGC(M_R) ≤ n/N + (1 − n/N)ᾱ. Setting CGC ≥ 1 − δ yields the bound.
- **Parametric upper bound.** If ⊕ has VC dimension d, fine-tuning on n_P = O((d + log(1/δ))/δ) examples suffices to reach 1 − δ on the full operator.
- **Separation.** n_R / n_P = Ω(k² / d). For structured operators with d = O(k) the gap is Ω(k); for simple operators (group operations, d = O(1)) it is Ω(k²).

The supremum accuracy of M_R is provably below M_P's whenever the data budget falls in the gap n_P ≤ n < n_R, which is non-empty for all k > k₀(d, δ, ᾱ). **This separation is context-window-independent.** Larger windows raise ᾱ marginally and never remove the Ω(k²) coverage demand.

### Mechanistic and empirical support

- Mechanistic interpretability — Meng et al. (2022) localize factual associations to specific MLP layers; Geva et al. (2020) cast FFN layers as key-value memories. Weight-based storage achieves compact, generalizable representations retrieval cannot structurally replicate.
- Yao et al. (2026) compare agents storing reflective experience externally vs. parametrically; parametric storage outperforms, and the gap *widens* on novel compositional transfer — exactly the regime Theorem 1 predicts.
- SCAN (Lake & Baroni 2017) and COGS (Kim & Linzen 2020) compositional benchmarks: weight-based models consistently outperform retrieval-only on systematic splits.
- Ovadia et al. (2024) find RAG excels at rare-entity recall but cannot lift compositional reasoning above the base model. Yang et al. (2026) confirm fine-tuning achieves highest accuracy on multi-hop queries.

### Frozen-novice problem (dynamic argument)

A separate, *non-compositional* failure: even on tasks the model could solve in principle, agents that operate exclusively via C-engineering cannot accumulate expertise. The session-start state is fixed at θ_frozen. Chi et al. (1981) show experts categorize problems by deep structural principles, novices by surface features — a reorganization that requires weight change. MemGPT-style "sleep-time compute" rewrites context tokens, not weights; the agent stays a well-organized novice.

### Security: persistent compromise

For a stateless agent, per-session injection success p₀ bounds compromise.
For an agent with persistent retrieval-based memory, once injected content reaches the store it is retrieved every subsequent session:
P(compromised by t) = 1 − (1 − p₀)^N(t)  →  1  as N(t) → ∞.

Compromising C requires only one successful injection during normal operation; compromising θ requires training-time access or weight editing — capabilities unavailable through normal queries. The asymmetry is structural, not a deployment accident.

### The proposed two-pathway architecture

```text
fast path:    query → embed → retrieve top-k from episodic store → condition LM
slow path:    episodic store -- (offline consolidation: SFT / KE / self-distill) --> θ_t+1
              with: (1) trace provenance, (2) versioned checkpoints, (3) regression guards
```

Three design principles the paper formalizes:
1. Treat agentic memory honestly as **episodic lookup** (don't pretend it's semantic or experiential).
2. Add a **consolidation pathway** from episodic store to weights, run asynchronously like sleep.
3. Make consolidation **safe**: trace provenance lets you audit which experiences became weight changes; versioned checkpoints let you roll back; regression guards prevent silent capability loss.

## Empirical anchors

This is a theory paper. The empirical anchors it cites are:
- Yao et al. 2026 — parametric > retrieval on novel compositional transfer; gap grows with novelty.
- Ovadia et al. 2024 — RAG excels at recall, fails to lift compositional reasoning.
- Yang et al. 2026 — fine-tuning achieves highest multi-hop accuracy.
- Liu et al. 2023 ("Lost in the Middle") + Paulsen 2026 — effective context utilization saturates at ~20K tokens even on 128K-context models.
- MINJA — 98.2% injection success rate with persistent cross-session effect.
- PoisonedRAG — five crafted documents flip RAG answers in million-doc corpora.
- SCAN, COGS — compositional generalization benchmarks where weight-based methods dominate retrieval-only.
- Ye et al. 2025 — 90% of SFT parameter updates contribute nothing to knowledge enhancement (motivates *circuit-aware* fine-tuning rather than full SFT during consolidation).

## Variants and counter-arguments addressed

The paper engages four common defenses of the C-engineering status quo and refutes each:

- **"Context windows will grow large enough."** No: the constraint is context-independent. Larger K raises ᾱ marginally, leaves Ω(k²) coverage intact, and effective utilization saturates well below stated context lengths.
- **"In-context learning is implicit gradient descent."** Akyürek et al. 2023 / Dai et al. 2023 show ICL *behaves* like one-step gradient descent — but base weights at session end are unchanged, so the parametric state recurs next session. No persistence.
- **"Memory, skills, rules lie on a compression spectrum."** Yes (Zhang et al. 2026b) — and that's the point. Low-compression traces belong in fast external stores; high-compression rules must live in weights. Treating all three as context lookup confuses points on the spectrum.
- **"Learned-retrieval (MemRL, Memento) closes the gap."** No: optimizing *which* exemplars to surface still feeds them to the frozen model at inference. Theorem 1 still binds. The gains reflect reduced noise on seen compositions, not generalization to novel ones.

## Failure modes and limitations

1. **Domain pretraining caveat.** The Ω(k²) bound binds when ᾱ < 1. If pretraining gave the model strong coverage of the composition operator (ᾱ → 1), the separation narrows. The argument therefore binds *most strongly* in domain-specific deployments, which is also where persistent agents matter most.
2. **Consolidation is unspecified.** The paper doesn't validate any specific consolidation algorithm. Whether consolidation actually produces compositional generalization — versus rote memorization of distilled examples — is open empirical work.
3. **Consolidation safety.** If poisoned traces are consolidated, a C-attack escalates to a θ-attack. Mitigation depends on engineering (provenance, checkpoints, regression guards) that doesn't exist in shipped systems.
4. **Capacity-bound tasks.** Tasks integrating m > K facts fail even without novelty (App. A). Theorem 1 is the deeper constraint; capacity is a co-occurring failure.

## When to use, when not

**Take the framing seriously when** designing persistent agents that must improve over time, encounter compositionally novel inputs (research agents, scientific discovery, complex reasoning loops), or operate in adversarial deployment. Use it as the lens for choosing between RAG and fine-tuning when knowledge updates are needed. Use it to design compositional-generalization-over-time evaluations, not just recall@k.

**Pure retrieval is still right** for tasks whose distribution is well-covered (FAQ matching, entity search, common recommendation), where ᾱ → 1 holds, where reversibility/auditability matters more than learning, or where you need a stateless agent. The paper's claim is **complementarity**, not replacement.

## Implications for harness engineering

- **Rethink the agent loop.** Most agent loops today are `plan → act → observe → reflect → store`. The paper argues they must become `plan → act → observe → reflect → store → (offline) consolidate`. The fifth arrow is what's missing. See [01-agent-loop-architecture](01-agent-loop-architecture.md) for the canonical loop and [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md) for the verifier extension that should now be paired with consolidation.
- **Two-pathway memory in harnesses.** Harnesses like Claude Code rely on file-based persistent memory (`CLAUDE.md`, `memory/*.md`) — these are episodic memos by the paper's typology. See [09-memory-files](09-memory-files.md) for current practice and [72-claude-mem-persistent-memory-compression](72-claude-mem-persistent-memory-compression.md) for the persistent-memory-compression direction. The paper's thesis says: keep these as the fast path, but add a slow path that consolidates distilled lessons into weights at the platform level (likely a managed-agents responsibility — see [73-multica](73-multica-managed-agents-platform.md)).
- **Reflexion is not learning.** [14-reflexion](14-reflexion.md) and [81-reasoningbank](81-reasoningbank.md) compress trajectories into reusable items, but the base model never updates. They lift accuracy on the seen distribution; they cannot close compositional gaps. ReasoningBank's MaTTS contrast extraction is the *strongest* known C-engineering signal — and is still bounded by ᾱ.
- **Voyager is procedural; consolidation is structural.** [19-voyager-skill-libraries](19-voyager-skill-libraries.md) accumulates *code* skills — externalized procedural memory. That's a strict subset of consolidation: it can call existing capabilities but can't create new compositional rules. The paper would say Voyager's library should be paired with periodic skill→weight distillation.
- **Native consolidation in the model.** [78-ngc-neural-garbage-collection](78-ngc-neural-garbage-collection.md) lets the model RL-learn to evict its own KV cache as a discrete action — collapsing the boundary between "what to remember" and "what to compute." NGC operates inside one trajectory; the paper's consolidation is across trajectories. They are layered, not competing.
- **Security: episodic stores are persistent attack surface.** [82-poisonedrag](82-poisonedrag.md) demonstrated that five documents can flip a million-doc RAG. Combined with the paper's persistent-compromise math, harnesses with retrieval memory need explicit injection isolation: untrusted retrievals enter a quarantined context only, never a consolidation candidate set, and consolidation requires trace provenance + human review for high-impact updates.
- **New evaluation primitive — Compositional Generalization over Time (CGT).** Run an agent for T sessions on isolated concepts, then evaluate on novel combinations. A learning agent shows accuracy strictly increasing in T; a pure-retrieval agent flatlines at the base-model baseline. Existing benchmarks (LoCoMo, LongMem, AgentBench) test the wrong thing. Adding CGT to harness eval suites would expose which "memory" features actually transfer learning.
- **Practical pipeline.** Trace → skill → weight is a three-stage compression: raw episodic traces go into the fast store; periodically distill into Voyager-style skills (medium compression); periodically consolidate skill clusters into weights via circuit-aware fine-tuning (Ye et al. 2025) targeting fact-memory units rather than full SFT.

The one-sentence takeaway for harness designers: **stop optimizing the filing cabinet and start building the sleep cycle.**
