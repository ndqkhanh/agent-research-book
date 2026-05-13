# 220 — Orion-Code Apply Plan: Autonomous Coding Agent with Multi-Hop Code-Search + Collaborative Surface

> **Disambiguation.** This file is the **Orion-Code–specific** deep apply plan, extending the per-project apply-plan family ([203](203-polaris-multi-hop-reasoning-apply-plan.md), [208](208-lyra-multi-hop-collaborative-apply-plan.md), [209](209-argus-multi-hop-collaborative-apply-plan.md), [210](210-mentat-learn-collaborative-apply-plan.md), [218](218-atlas-research-multi-hop-collaborative-apply-plan.md), [219](219-helix-bio-multi-hop-collaborative-apply-plan.md)). The cross-project matrix is [207](207-cross-project-collaborative-multi-hop-apply-plan.md); the executable strategy doc is [211](211-cross-project-power-up-plan-with-tradeoffs.md). Orion-Code's role in the portfolio: **autonomous coding agent** baselined against Claude Code / Devin / Cursor / SWE-agent, with the design target of +5–15pp on SWE-Bench Verified at similar cost and <10% undetected sabotage at 1% FPR on LinuxArena.

> **Compare with Lyra** ([208](208-lyra-multi-hop-collaborative-apply-plan.md)). Both are coding agents; they differ on **autonomy posture**. Lyra is a **CLI-native interactive coding harness** (4-mode router, 80+ slash commands, the developer at the terminal). Orion is an **autonomous task agent** baselined against Devin / SWE-agent (long-horizon, less interactive, more autonomous). The multi-hop substrate is shared; the UX layer and autonomy budget differ.

## One-line definition

A staged plan to fold multi-hop reasoning + collaborative-AI techniques into Orion-Code — the **autonomous coding agent** with a long-horizon trajectory, hardened safety surface (LinuxArena sabotage detection at 1% FPR), and SWE-Bench-Verified delta as the design target — where the multi-hop substrate (HippoRAG over symbol graph + DSPy multi-hop program + Plan-on-Graph backtracking) is the cross-file refactor primitive and the collaborative-AI substrate (ICPEA + branching + Pages design doc + per-user constitution) makes the agent feel like a teammate rather than a one-shot codegen pipeline.

## §1 — Why apply these techniques to Orion-Code

Orion-Code's distinctive position in the portfolio:

- **Long-horizon autonomous coding.** Unlike Lyra (developer-at-terminal), Orion is shaped for multi-hour autonomous trajectories — Devin-class workloads.
- **Trajectory quality is the win condition.** Per [projects/README.md](../projects/README.md), Orion's design target is **+5–15pp on SWE-Bench Verified at similar cost** and **<10% undetected sabotage at 1% FPR**. The first requires multi-hop code-search + reasoning; the second requires hardened pure-function discipline + verifier loops.
- **Existing scaffolding.** Orion ships with Read / Grep / Edit / Write / Bash-allowlist tools + Plan Mode gate; it's the most production-shaped of the in-tree coding projects in terms of tool surface.

What's missing relative to the [206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md) maturity matrix and [202-multi-agent-multi-hop-reckoning-2026](202-multi-agent-multi-hop-reckoning-2026.md):

- **Multi-hop substrate for cross-file reasoning.** Orion's existing tools are single-shot. Cross-file refactors and "where-does-this-error-come-from" investigations are inherently multi-hop.
- **ICPEA personal memory.** Orion has no typed personal memory layer for the developer.
- **Branching as alternative-implementation UX.** Long-horizon autonomous trajectories benefit from "show three approaches" forking.
- **Pages-style design doc surface.** Orion's output is code; for non-trivial tasks the user wants to co-author the design before implementation.
- **Pure-function agents with replayability.** Critical for sabotage-detection — non-deterministic agents can't be audited at 1% FPR.

Take this seriously and three things change. (1) Orion's SWE-Bench Verified delta is achievable because cross-file refactor — the canonical SWE-Bench failure mode — gets a real multi-hop substrate. (2) Sabotage detection at 1% FPR becomes feasible because pure-function agents make trajectories replayable for forensic comparison. (3) Orion graduates from "autonomous codegen pipeline" to "teammate that grows with the developer" without losing autonomy.

## §2 — The Orion-Code × technique mapping table

Orion-Code's existing module structure is `src/orion_code/`, with vendored `harness_core/`, `skills/`, `tests/`, `docs/`. The mapping table:

| Technique | Source | Orion module | Tier | Effort | Payoff |
|---|---|---|---|---|---|
| **Multi-hop substrate (code-shaped)** | | | | | |
| HippoRAG-2 over tree-sitter symbol graph | [200](200-graph-grounded-multi-hop-retrieval.md), [208](208-lyra-multi-hop-collaborative-apply-plan.md) §3.1 | `src/orion_code/code_graph/hipporag.py` (vendored from `harness_core.multi_hop.hipporag`) | 0 | M | Cross-file refactor via single-shot retrieval |
| Tree-sitter symbol graph as substrate | shared with [208](208-lyra-multi-hop-collaborative-apply-plan.md) | `src/orion_code/code_graph/tree_sitter_graph.py` | 0 | S | Code-aware extraction |
| LightRAG incremental ingest on file save | [200](200-graph-grounded-multi-hop-retrieval.md) | `src/orion_code/code_graph/incremental.py` | 0 | S | Online graph update |
| BELLE-style code-question router | [202](202-multi-agent-multi-hop-reckoning-2026.md) §1, [208](208-lyra-multi-hop-collaborative-apply-plan.md) | `src/orion_code/routing/code_question_router.py` | 0 | S | Refactor / debug / explain / generate routing |
| Self-Ask / IRCoT for cross-file reasoning | [199](199-multi-hop-reasoning-techniques-arc.md) Phase 1 | `src/orion_code/reasoning/{self_ask,ircot}.py` | 0 | S | Externalises bridge entity |
| Chain-of-Note per-file filter | [199](199-multi-hop-reasoning-techniques-arc.md) Phase 3 | `harness_core/gates/chain_of_note.py` | 0 | S | Per-file quality gate |
| Decomposition cache | [199](199-multi-hop-reasoning-techniques-arc.md) | `src/orion_code/memory/decomposition_cache.py` | 0 | S | Latency cut on recurring sub-queries |
| DSPy multi-hop program for SWE-Bench-shaped tasks | [199](199-multi-hop-reasoning-techniques-arc.md) Phase 2 | `skills/research/dspy_swe_program.py` | 1 | M | Compiled-against-evals SWE-Bench pipeline |
| Plan-on-Graph backtracking on call-graph walks | [200](200-graph-grounded-multi-hop-retrieval.md) | `src/orion_code/code_graph/plan_walk.py` | 1 | M | Recovers from wrong-path call traversal |
| Reason-in-Documents on retrieved code | [199](199-multi-hop-reasoning-techniques-arc.md) Phase 4 | `src/orion_code/reasoning/reason_in_code.py` | 1 | S | Denoise retrieved code chunks |
| Search-R1 RL with code-grounded reward | [199](199-multi-hop-reasoning-techniques-arc.md) Phase 4 | `src/orion_code/rl/search_r1_code.py` | 2 | L | Trained code-search policy |
| **Collaborative-AI substrate** | | | | | |
| ICPEA five-layer personal memory | [205](205-lobehub-collaborative-teammate-platform.md) §2.1 | `src/orion_code/memory/icpea/` | 1 | M | Per-developer style + project-convention layer |
| Async memory extractor on task-completion | [206](206-collaborative-ai-canon-2026.md) §1 | `src/orion_code/memory/icpea/extractor.py` | 1 | S | Decouples memory write from chat latency |
| Per-agent layer access control | [205](205-lobehub-collaborative-teammate-platform.md) §2.1 | `src/orion_code/memory/icpea/access.py` | 1 | S | TDD subagent declared layers; verifier subagent declared layers |
| Branching as alternative-implementation UX | [206](206-collaborative-ai-canon-2026.md) §4 | `src/orion_code/conversation/branching.py` (using existing git-worktree subagent infra) | 1 | M | "Show 3 approaches" with separate worktrees |
| Lobe Pages-style co-authored design doc | [205](205-lobehub-collaborative-teammate-platform.md) §2.2 | `src/orion_code/output/pages.py` | 1 | M | Co-authored architecture / RFC document |
| MCP-first marketplace via argus trust gate | [205](205-lobehub-collaborative-teammate-platform.md) §2.3, [209](209-argus-multi-hop-collaborative-apply-plan.md) | `src/orion_code/tools/mcp_marketplace.py` | 1 | S | Trust-gated coding-tool installation |
| Voyager-line skill auto-creation | [167](167-autoskill-experience-driven-lifelong-learning.md), [197](197-argus-omega-vol-3-recursive-skills-curator.md) | `src/orion_code/skills/auto_creator.py` (submits to argus.curator) | 1 | M | Successful refactor patterns become reusable |
| Per-user constitution | [206](206-collaborative-ai-canon-2026.md) §5 | `src/orion_code/constitution/` | 2 | M | Developer style + project conventions |
| **Verifier / safety surface (Orion's distinctive surface)** | | | | | |
| Verifier-evaluator loops | [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [18-chain-of-verification-self-refine](18-chain-of-verification-self-refine.md) | `src/orion_code/verifier/` (extend existing) | 0 | M | Test-pass + lint + type-check + security-scan as composed gate |
| LinuxArena sabotage-detection eval at 1% FPR | [projects/README.md](../projects/README.md), [26-linuxarena-production-agent-safety](26-linuxarena-production-agent-safety.md) | `tests/benchmarks/linuxarena_sabotage.py` | 0 | M | Critical safety eval |
| Trajectory-replay forensic comparator | [188-witness-provenance-memory-techniques-synthesis](188-witness-provenance-memory-techniques-synthesis.md) | `src/orion_code/forensic/replay_compare.py` | 1 | M | Detect sabotage by replaying past trajectories |
| **Cross-cutting discipline** | | | | | |
| Pure-function agents | [202](202-multi-agent-multi-hop-reckoning-2026.md) §4 | `harness_core/orchestration/pure_function.py` | 0 | S | Replayable, sabotage-detection prerequisite |
| Equal-budget benchmark harness | [202](202-multi-agent-multi-hop-reckoning-2026.md) §3 | `harness_core/evals/equal_budget.py` | 0 | S | Honest baselines vs Devin / Cursor / Claude Code |
| TTC curve plotter | [199](199-multi-hop-reasoning-techniques-arc.md) | `harness_core/evals/ttc_curve.py` | 0 | S | Detect plateau-or-decline |
| SWE-Bench Verified + LinuxArena nightly | [projects/README.md](../projects/README.md) | `tests/benchmarks/{swebench_verified,linuxarena}.py` | 0 | S | Design-target regression triple |
| MuSiQue-Code + FRAMES-Code (code-multi-hop) | [198](198-multi-hop-qa-datasets-canon.md) adapted | `tests/benchmarks/multi_hop_code/` | 0 | S | Code-multi-hop regression |
| gnomon HIR emit | [211](211-cross-project-power-up-plan-with-tradeoffs.md) §4 | `src/orion_code/observability/hir_emit.py` | 0 | S | Cross-harness eval signal |

## §3 — Tier 0 (days 1-14): the multi-hop code-search + safety substrate

Orion's Tier 0 is shaped by two non-negotiables: **SWE-Bench Verified** as the throughput target and **LinuxArena sabotage detection at 1% FPR** as the safety target.

### 3.1 — Pure-function agents (sabotage-detection prerequisite)

**File.** `harness_core/orchestration/pure_function.py` (shared per [211](211-cross-project-power-up-plan-with-tradeoffs.md) §4).

**What.** Yenugula's pattern ([202-multi-agent-multi-hop-reckoning-2026](202-multi-agent-multi-hop-reckoning-2026.md) §4): every agent decision is a pure function of (input, context, tool-results). Side-effecting tools (Edit, Write, Bash-mutation) go through gated APIs that log + require explicit BL-* permission. Replayable trajectories.

**Why critical for Orion.** Sabotage detection at 1% FPR requires *replay-comparison* against historical trajectories; non-deterministic agents make this impossible.

### 3.2 — HippoRAG-2 over tree-sitter symbol graph

**Files.** `src/orion_code/code_graph/{tree_sitter_graph,hipporag,incremental}.py`.

**What.** Same pattern as Lyra ([208](208-lyra-multi-hop-collaborative-apply-plan.md) §3.1). Tree-sitter extracts a typed symbol graph (functions, classes, methods, callers, imports); HippoRAG-2 PPR fusion runs on the graph. At query time, multi-hop-coherent code passages return in a single retrieval pass.

**Cross-link to Lyra.** Orion and Lyra **share the tree-sitter graph extractor**. Vendor it from a shared library to avoid duplication.

### 3.3 — BELLE code-question router + Self-Ask / IRCoT + Chain-of-Note

**Files.** `src/orion_code/routing/code_question_router.py`, `src/orion_code/reasoning/{self_ask,ircot}.py`, `harness_core/gates/chain_of_note.py`.

**What.** Standard from [208](208-lyra-multi-hop-collaborative-apply-plan.md) §3.3-3.5; same operators wired into Orion's autonomous trajectory loop instead of Lyra's interactive 4-mode router.

### 3.4 — Verifier-evaluator loop with composed gates

**File.** `src/orion_code/verifier/` (extend existing).

**What.** Orion ships with a verifier; this Tier-0 work composes it with multi-axis quality gates:
- Test pass / fail (existing).
- Lint clean (Ruff / Pylint / ESLint).
- Type-check clean (mypy / tsc).
- Security scan clean (Bandit / npm audit).
- BL-* gates: no unauthorised filesystem writes, no unauthorised network egress.

Composed gate: any axis fail → trajectory rolled back; multi-axis fail → BL-SABOTAGE flag triggered.

Cf. [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [18-chain-of-verification-self-refine](18-chain-of-verification-self-refine.md).

### 3.5 — SWE-Bench Verified + LinuxArena + multi-hop-code in CI

**Files.** `tests/benchmarks/{swebench_verified,linuxarena,multi_hop_code/}.py`.

**What.** Orion's design target is +5–15pp on SWE-Bench Verified and <10% undetected sabotage at 1% FPR on LinuxArena. Both ship in nightly CI from day one. Plus MuSiQue-Code + FRAMES-Code adapted from the multi-hop triple ([198](198-multi-hop-qa-datasets-canon.md)).

### 3.6 — Equal-budget + TTC + active-params + gnomon HIR emit

Cross-cutting Tier 0 discipline per [211](211-cross-project-power-up-plan-with-tradeoffs.md). Equal-budget is critical for honest comparisons against Devin / Cursor / Claude Code (those products' published numbers don't always control for compute).

## §4 — Tier 1 (days 15-45): user-aware coding + branching + Pages

After Tier 0, Orion has multi-hop substrate + verifier + safety eval. Tier 1 makes the agent user-aware and exposes branching / Pages.

### 4.1 — DSPy multi-hop program for SWE-Bench

**File.** `skills/research/dspy_swe_program.py`.

**What.** Compose Tier 0's operators into a DSPy `Module`. Compile against held-out SWE-Bench Verified subset. The compiled program is the canonical Orion code-research pipeline.

### 4.2 — Plan-on-Graph backtracking on call-graph walks

**File.** `src/orion_code/code_graph/plan_walk.py`.

**What.** When Orion's trajectory wants to find "the function that calls X that handles Y" and the first walk hits a wrong subgraph, the Reflection mechanism backtracks. The visited subgraph is logged in the trajectory.

### 4.3 — Reason-in-Documents on retrieved code

**File.** `src/orion_code/reasoning/reason_in_code.py`.

**What.** Search-o1's denoising step adapted for code chunks. Refines retrieved code before injection into the reasoning context.

### 4.4 — ICPEA five-layer personal memory

**File.** `src/orion_code/memory/icpea/`.

**What.** Standard ICPEA tuned for coding:
- **I**dentity — "I'm a senior backend engineer working on a Python monorepo with strict typing."
- **C**ontext — current task, current branch, recent files, project conventions.
- **P**reference — preferred libraries (`httpx` over `requests`), code style, test framework, commit-message format.
- **E**xperience — what worked (refactor patterns the user accepted), what failed (subagent strategies the user rolled back).
- **A**ctivity — tool invocations, edited files, tests run.

### 4.5 — Branching as alternative-implementation UX

**File.** `src/orion_code/conversation/branching.py`.

**What.** "Show me three approaches to this refactor" → fork at the message, three parallel agent runs in separate git worktrees, present as branches the user picks from. Pure-function agents (Tier 0) make this safe.

Implementation note: each branch is a git worktree (Orion already has worktree infra for subagents); diff between branches uses `git diff` natively.

### 4.6 — Lobe Pages-style co-authored design doc

**File.** `src/orion_code/output/pages.py`.

**What.** Before any non-trivial implementation, the user and the agent co-author a design doc / RFC. Notion-style timeline preserves prior versions. The implementation then references the design doc as ground truth.

This pattern is high-value for Orion's autonomous trajectories — pinning the design before the agent runs for hours reduces the "you implemented the wrong thing" failure mode.

### 4.7 — argus trust-gated MCP marketplace

**File.** `src/orion_code/tools/mcp_marketplace.py`.

**What.** Consume `argus.HostAdapter.install_mcp(...)`. Orion's typical MCP needs: linters, formatters, security scanners, type-checkers, package-managers, deployment tools.

### 4.8 — Voyager-line skill auto-creation

**File.** `src/orion_code/skills/auto_creator.py`.

**What.** Successful refactor patterns (test pass + user accept + ≥2 occurrences) submit to `argus.curator.submit_trajectory(...)`. Orion pulls promoted skills back from argus, including skills promoted by Lyra (cross-project commons).

### 4.9 — Trajectory-replay forensic comparator

**File.** `src/orion_code/forensic/replay_compare.py`.

**What.** Given a current trajectory and a corpus of past trajectories on similar tasks, compute pairwise distance metrics (action-sequence similarity, tool-use distribution). Sabotage-detection signal: a trajectory that looks unlike all past trajectories on similar tasks is a candidate for review.

Pure-function agents (Tier 0) are the prerequisite — non-deterministic trajectories can't be replay-compared.

## §5 — Tier 2 (days 46-90): training surface + per-user constitution

### 5.1 — Search-R1 RL with code-grounded reward

**File.** `src/orion_code/rl/search_r1_code.py`.

**What.** Fork Search-R1 ([199](199-multi-hop-reasoning-techniques-arc.md) Phase 4) for code. Reward function:
- Test pass = +1
- Test fail = penalty
- Edited-file-not-imported-anywhere = penalty (catches "fixed" code that doesn't run)
- Lint / type-check / security-scan fail = penalty
- BL-SABOTAGE flag = strong penalty
- User-accepted edit (ICPEA Experience layer) = bonus
- User-rolled-back edit = penalty

Backbone: Qwen2.5-Coder-7B initial; 32B once recipe validates.

### 5.2 — Per-user constitution

**File.** `src/orion_code/constitution/`.

**What.** ICAI-style 3-principle constitution per developer. Examples:
- "Always prefer explicit type hints in Python; reject inference-heavy code."
- "Avoid functions over 40 lines; refactor before exceeding."
- "Default to `httpx` and `pytest`; use library-author conventions when contributing upstream."

Composed with project-level `.orion/constitution.md` (project conventions); project overrides developer on conflict.

## §6 — Five sequencing decisions worth defending

### Decision 1 — Pure-function agents Tier 0 (sabotage-detection prerequisite)

LinuxArena sabotage-detection at 1% FPR is impossible without replay-able trajectories. Pure-function agents must ship in days 1-14.

### Decision 2 — Tree-sitter symbol graph shared with Lyra

Orion and Lyra both need code-aware HippoRAG. Shared library extractor avoids the duplication tax. Per [211](211-cross-project-power-up-plan-with-tradeoffs.md) §6 tradeoff #1.

### Decision 3 — Branching via git worktrees, not in-memory forks

Orion already uses worktrees for subagents. Branching as alternative-implementation UX rides the same machinery. In-memory forks would be cheaper but lose the "real working tree" property.

### Decision 4 — Design doc Pages co-authored *before* implementation

For non-trivial autonomous tasks, pinning the design before the agent runs for hours dramatically reduces the "wrong implementation" failure mode. This is Orion-specific value Lyra doesn't need (Lyra's user is at the terminal interactively).

### Decision 5 — Verifier composes multi-axis gates Tier 0

A single test-pass gate is insufficient for SWE-Bench Verified. Lint + type-check + security-scan + BL-* gates must compose from day one.

## §7 — Risks and mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Search-R1 reward-hacks tests | Med | High | Test mutation testing in eval pipeline; user-accept signal weight > test pass; held-out adversarial-eval set |
| Tree-sitter graph misses dynamic dispatch / metaclasses | Med | Med | Hybrid graph: tree-sitter primary + LLM-extracted fallback for dynamic edges |
| Pure-function refactor blocks existing Orion users | Low | Med | Phased rollout; existing trajectories migrated lazily |
| Sabotage-detection false-positives | Med | Med | Replay-comparator threshold tunable; HITL review on uncertain calls |
| LinuxArena sabotage-detection regresses with verifier changes | Med | High | Per-PR mandatory benchmark; alarm on >1pt FPR regression |
| ICPEA cross-contaminates personal data into git history | Low | High | ICPEA stored in `~/.orion/icpea/`, never in repo |
| Branching exhausts CI / runner budget | Med | Med | Per-branch budget caps; auto-cap parallel branches at 3 |
| Design doc Pages becomes stale as implementation diverges | Med | Med | Periodic re-sync prompt; flag drift in trajectory logs |
| Argus marketplace SPOF on tool installation | Low | Med | Local cache for installed tools; graceful degradation |
| HippoRAG ingest cost on monorepos | Med | High | LightRAG-style incremental + lazy ingest of unaccessed subtrees |
| Equal-budget benchmark misrepresents subagent dispatch | High | Low-Med | Report per-subagent cost separately; document multi-subagent dispatch in eval reports |

## §8 — Concrete day-by-day Tier 0 checklist

A 14-day Tier 0 plan:

- **Day 1-2** — `harness_core/orchestration/pure_function.py` adopted; agent loop refactored.
- **Day 3** — `src/orion_code/code_graph/tree_sitter_graph.py`; benchmark ingest time on a sample monorepo.
- **Day 4-5** — `src/orion_code/code_graph/{hipporag,incremental}.py`; first end-to-end multi-hop code retrieval.
- **Day 6** — `src/orion_code/routing/code_question_router.py` (BELLE-style).
- **Day 7** — `src/orion_code/reasoning/{self_ask,ircot}.py` operators.
- **Day 8** — `harness_core/gates/chain_of_note.py` wired.
- **Day 9-10** — `src/orion_code/verifier/` extended with lint + type-check + security-scan composition.
- **Day 11** — `src/orion_code/memory/decomposition_cache.py`.
- **Day 12** — `tests/benchmarks/{swebench_verified,linuxarena,multi_hop_code/}.py`.
- **Day 13** — `harness_core/evals/{equal_budget,ttc_curve,active_params}.py` + `src/orion_code/observability/hir_emit.py`.
- **Day 14** — Tier 0 retro: SWE-Bench Verified baseline + LinuxArena sabotage rate + multi-hop-code metrics; sign-off for Tier 1.

## §9 — How Orion-Code relates to Lyra and the rest of the portfolio

| Project | Niche | Autonomy posture | Multi-hop substrate | Distinctive surface |
|---|---|---|---|---|
| **Orion-Code** | Autonomous coding | Long-horizon autonomous (Devin-class) | Tree-sitter graph + HippoRAG-2 + DSPy + PoG + Search-R1 RL | Verifier + LinuxArena sabotage detection + design-doc Pages |
| **Lyra** | CLI coding | Interactive (developer at terminal) | Same multi-hop substrate | 4-mode router + 80+ slash commands + 5-layer context |
| **Polaris** | Deep research | Long-horizon autonomous | Same | Twenty BL-* gates + typed trust tiers |
| **Mentat-Learn** | Personal assistant | Conversational | Sparingly applied | ICPEA + SOUL.md + multi-channel gateway |

Orion and Lyra **share the tree-sitter graph extractor + HippoRAG-2 multi-hop substrate**. They diverge on **autonomy posture and UX surface**: Orion is shaped for long-horizon autonomous tasks against SWE-Bench / LinuxArena; Lyra is shaped for interactive developer collaboration. Both consume the same shared `harness_core/` library; both contribute trajectories to argus's curator.

## §10 — Cross-references

- [208-lyra-multi-hop-collaborative-apply-plan](208-lyra-multi-hop-collaborative-apply-plan.md) — sibling coding plan (interactive).
- [203-polaris-multi-hop-reasoning-apply-plan](203-polaris-multi-hop-reasoning-apply-plan.md), [218-atlas-research-multi-hop-collaborative-apply-plan](218-atlas-research-multi-hop-collaborative-apply-plan.md), [219-helix-bio-multi-hop-collaborative-apply-plan](219-helix-bio-multi-hop-collaborative-apply-plan.md) — sibling research / domain plans.
- [211-cross-project-power-up-plan-with-tradeoffs](211-cross-project-power-up-plan-with-tradeoffs.md) — strategy doc.
- [26-linuxarena-production-agent-safety](26-linuxarena-production-agent-safety.md) — LinuxArena safety baseline.
- [38-claw-eval](38-claw-eval.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [18-chain-of-verification-self-refine](18-chain-of-verification-self-refine.md) — verifier lineage.
- [29-dive-into-claude-code](29-dive-into-claude-code.md), [62-everything-claude-code](62-everything-claude-code.md), [46-components-of-coding-agent](46-components-of-coding-agent.md), [144-build-your-own-harness](144-build-your-own-harness.md), [145-comparing-coding-harnesses](145-comparing-coding-harnesses.md) — coding-agent canon.
- [180-argus-skill-router-agent-design](180-argus-skill-router-agent-design.md) — argus integration.

## §11 — One-paragraph summary

Orion-Code's multi-hop + collaborative-AI subsystem ships in three tiers: Tier 0 (14 days) hardens the **multi-hop code-search + safety substrate** with pure-function agents (sabotage-detection prerequisite) + tree-sitter symbol graph + HippoRAG-2 + LightRAG incremental ingest + BELLE code-question router + Self-Ask/IRCoT + Chain-of-Note + verifier composing multi-axis gates (test + lint + type + security + BL-*) + SWE-Bench Verified + LinuxArena sabotage-detection + multi-hop-code benchmarks + equal-budget + TTC + gnomon HIR emit; Tier 1 (30 days) makes the agent user-aware with DSPy multi-hop program + Plan-on-Graph backtracking + Reason-in-Documents + ICPEA five-layer memory + branching as alternative-implementation UX (via existing git-worktree subagent infra) + Lobe Pages-style co-authored design doc (high-value for autonomous tasks) + argus trust-gated MCP marketplace + Voyager skill auto-creation + trajectory-replay forensic comparator (sabotage-detection signal); Tier 2 (60 days) trains the executor with code-grounded Search-R1 + per-user constitution. The five sequencing decisions defended in §6 keep pure-function agents Tier-0 (sabotage-detection prerequisite), share the tree-sitter graph with Lyra, branch via git worktrees, co-author design doc *before* implementation, and compose multi-axis verifier gates from day one.

**The one-line takeaway for harness designers:** Orion-Code is the **autonomous-task** sibling of Lyra — same multi-hop code-search substrate, but harden pure-function agents Tier-0 for LinuxArena sabotage-detection, add design-doc Pages co-authoring before implementation, and use git worktrees as the substrate for both subagent orchestration *and* branching alternative-implementation UX.
