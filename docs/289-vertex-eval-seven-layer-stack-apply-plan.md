# 289 — Vertex-Eval × Seven-Layer Stack Apply Plan 2026

**Anchors.** Vertex-Eval — evaluation framework / benchmarking agent ([projects/vertex-eval](../projects/vertex-eval/)). Companion: [115-evaluating-llm-systems](115-evaluating-llm-systems.md), [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md), [265-agent-evaluation-2026](265-agent-evaluation-2026.md).

**One-line definition.** A **per-layer apply plan** for Vertex-Eval — the **evaluation / benchmarking agent** that runs eval suites against other agents — emphasizing the **provider-role overlay** (vertex-eval is consumed by every other project for their CI eval pipelines), the **benchmark-as-service architecture** (GAIA / OSWorld / GDPval / SWE-bench Verified / MultiAgentBench / MAST as A2A-callable services), and **statistical-rigor SRE** (sample sizing, confidence intervals, paired comparisons) per [265](265-agent-evaluation-2026.md) — staged across four 90-day phases.

## Per-layer plan

### L1 Foundation
Standard. Permission Bridge with eval-specific bright-lines (`EVAL_OVER_BUDGET`, `EVAL_PUBLISH`).

### L2 Capability
**LLM-judge calibration** ([265](265-agent-evaluation-2026.md)) is the load-bearing capability. Cross-channel verifier ensemble (3+ model families) for high-stakes judging.

### L3 Protocol
- **MCP**: benchmark-runner MCP, judge-result MCP, statistical-analysis MCP.
- **A2A**: vertex-eval exposes `eval_suite_run`, `compare_models`, `judge_calibrate` as A2A capabilities.
- **AGNTCY**: published OASF; trust tier `audited`.
- **Routines**: scheduled "weekly eval" + "regression on every dependency update" routines.

### L4 Runtime
LangGraph for state-machine eval workflow (load suite → run model → run judge → analyze → publish). Postgres checkpointer mandatory for reproducibility.

### L5 Security
- **Prompt injection**: critical — eval datasets often contain adversarial inputs by design.
- **Supply chain**: SBOM of judges + benchmarks; signed eval datasets.
- **Isolation**: per-eval-run container.

### L6 Operations
- **Observability**: per-eval-run + per-judgment span; provenance metadata.
- **Eval**: meta-eval (calibrate the judge!).
- **Durability**: durable across long benchmark runs.
- **SRE**: SLO on eval-suite reproducibility (paired runs within CI bound).

### L7 Compliance
**EU AI Act**: vertex-eval as third-party verifier for high-risk systems contributes to conformity assessment evidence. SOC 2 if commercial.

## Phased rollout

| Phase | Scope | Duration |
|---|---|---|
| **P1** | L1-L2 + benchmark library (GAIA, SWE-bench, MultiAgentBench) | 90 days |
| **P2** | L3 (A2A) + L4 (LangGraph) + judge calibration | 90 days |
| **P3** | L6 Operations + statistical methodology + per-project integration | 90 days |
| **P4** | L7 Compliance + production hardening | 90 days |

## One-line takeaway

**Vertex-Eval is the **evaluation provider** in the in-tree ecosystem — A2A-callable benchmark + judge services consumed by every other project's CI pipeline — across four 90-day phases adopting the seven-layer stack with provider-role overlay (high-availability, calibrated judges, SBOM of benchmarks + judges, statistical-rigor SRE), and serving as the third-party verifier whose evidence contributes to EU AI Act conformity assessments for high-risk consumer agents.**
