# 293 — Quanta-Proof × Seven-Layer Stack Apply Plan 2026

**Anchors.** Quanta-Proof — math / formal-proof agent ([projects/quanta-proof](../projects/quanta-proof/)). Companion: [39-ai-and-mathematics-structure](39-ai-and-mathematics-structure.md), [85-alphaevolve](85-alphaevolve.md), [97-qwen-prm](97-qwen-prm.md), [232-rl-on-reasoning-traces-scaling](232-rl-on-reasoning-traces-scaling.md), [156-heavyskill-parallel-reasoning-deliberation](156-heavyskill-parallel-reasoning-deliberation.md).

**One-line definition.** A **per-layer apply plan** for Quanta-Proof — the **mathematical reasoning + formal proof** agent — emphasizing the **thinking-model + tree-search + PRM** stack (DeepSeek-R1 ([232](232-rl-on-reasoning-traces-scaling.md)), Qwen-PRM ([97](97-qwen-prm.md)), HeavySkill ([156](156-heavyskill-parallel-reasoning-deliberation.md)), tree-of-thoughts / LATS ([15-tree-of-thoughts-lats](15-tree-of-thoughts-lats.md))), **formal-verifier integration** (Lean / Coq / Isabelle as the bright-line truth), and **cross-channel verifier ensemble** — staged across four 90-day phases targeting Olympiad-class proof generation.

## Per-layer plan

### L1 Foundation
Standard. Permission Bridge gates external math-tool calls (Wolfram, formal-verifier sessions).

### L2 Capability
**Pretraining**: math-finetuned (R1-Distill-Qwen-32B, AlphaProof / AlphaProver). **TTC**: thinking model with tree-search inner skill. **Trajectory**: long-horizon for multi-step proofs. **Multi-agent**: HeavySkill team (parallel-then-deliberate). **Verifier**: **Lean / Coq formal verifier as hard truth**; PRM ([97](97-qwen-prm.md)) for sub-step evaluation; cross-channel for high-stakes claims.

### L3 Protocol
- **MCP**: Lean MCP, Coq MCP, Isabelle MCP, Sympy MCP, Wolfram Alpha MCP.
- **A2A**: quanta-proof exposes `prove`, `simplify`, `solve`, `verify_proof` capabilities.
- **AGNTCY**: published OASF.
- **SKILL.md**: math skills (proof tactics, lemma libraries) via argus.
- **Routines**: scheduled "weekly review of unsolved Olympiad problems" routines.

### L4 Runtime
LangGraph for stateful proof workflows (decompose → search → verify → integrate). Tree-search inner skill ([84-swe-search-mcts](84-swe-search-mcts.md), [15-tree-of-thoughts-lats](15-tree-of-thoughts-lats.md)) for hard problems.

### L5 Security
- **Prompt injection** less critical (formal verifier is the truth gate).
- **Supply chain**: vendored math skills; signed lemma libraries.
- **Isolation**: container per proof attempt; formal-verifier in dedicated container.
- **Bright-line**: `PUBLISH_PROOF`, `MODIFY_LEMMA_LIBRARY`.

### L6 Operations
- **Observability**: per-step PRM scores; per-search-tree-node metrics.
- **Eval**: AIME-2024/2025, MATH-500, MiniF2F (Lean formal-proof benchmark), AlphaProof's open subset.
- **Durability**: LangGraph + Postgres; long proof attempts checkpointed.
- **SRE**: SLO on proof-validity (Lean-verified) ≥ 99% on attempted; quality before quantity.

### L7 Compliance
**EU AI Act**: limited risk (academic / research math). **Authorship**: clear AI-assistance disclosure when proofs published.

## Phased rollout

| Phase | Scope | Duration |
|---|---|---|
| **P1** | L1-L2 + thinking model + Lean MCP integration | 90 days |
| **P2** | L3-L4 (HeavySkill + tree search) + L5 (formal verifier as truth gate) | 90 days |
| **P3** | L6 Operations + benchmark eval | 90 days |
| **P4** | L7 Compliance + AlphaProof-class deployment | 90 days |

## One-line takeaway

**Quanta-Proof adopts the seven-layer stack as the math + formal-proof agent — thinking model + tree-search inner skill + Qwen-PRM at sub-step + Lean / Coq formal verifier as bright-line truth gate + HeavySkill parallel-then-deliberate, with proof attempts in isolated containers, durable across long search trajectories via LangGraph checkpointer, and AIME / MATH / MiniF2F as canonical evals; the formal verifier is the load-bearing primitive that turns "the model produced a proof" into "the proof is valid".**
