# 291 — Orion-Code × Seven-Layer Stack Apply Plan 2026

**Anchors.** Orion-Code — software-engineering / coding agent ([projects/orion-code](../projects/orion-code/)). Companion: [220-orion-code-multi-hop-collaborative-apply-plan](220-orion-code-multi-hop-collaborative-apply-plan.md), [46-components-of-coding-agent](46-components-of-coding-agent.md), [62-everything-claude-code](62-everything-claude-code.md), [68-atomic-skills-scaling-coding-agents](68-atomic-skills-scaling-coding-agents.md), [84-swe-search-mcts](84-swe-search-mcts.md), [92-chatdev](92-chatdev.md), [145-comparing-coding-harnesses](145-comparing-coding-harnesses.md).

**One-line definition.** A **per-layer apply plan** for Orion-Code — the **SWE-bench-class coding agent** with custom-ACI ([237-agent-trajectory-scaling](237-agent-trajectory-scaling.md), [236-tool-use-and-aci-scaling](236-tool-use-and-aci-scaling.md)), tree-search inner skill ([84](84-swe-search-mcts.md)), and verifier-evaluator loops — emphasizing the **ACI-quality multiplier** (SWE-agent demonstrated 4× capability gain from custom ACI alone), **worktree-per-task isolation** ([18](../projects/polaris/docs/blocks/18-subagent-and-worktree.md), [271](271-agent-isolation-patterns.md)), **bright-line gates on git push / package install / prod-config**, and **multi-agent code review team** (security / performance / coverage reviewers) — staged across four 90-day phases.

## Per-layer plan

### L1 Foundation
Standard. Strict Permission Bridge for `git_push`, `pip_install`, `npm_install`, `delete_file`. Bright-line on `force_push_to_main`, `prod_config_change`.

### L2 Capability
**Pretraining**: code-specialized (Qwen-2.5-Coder, DeepSeek-Coder-V2, Claude Sonnet 4.6 for general coding). **TTC**: thinking model on hard SWE problems; tree-search inner skill ([84-swe-search-mcts](84-swe-search-mcts.md)). **Trajectory**: SWE-agent custom ACI; T_plateau ~50 steps. **Multi-agent**: code review team (Anthropic Agent Teams pattern, [250](250-anthropic-agent-teams.md)).

### L3 Protocol
- **MCP**: git MCP, file-edit MCP, test-runner MCP, lint MCP, search MCP.
- **A2A**: orion-code exposes `code_review`, `feature_implement`, `bug_fix`, `refactor` capabilities.
- **AGNTCY**: published OASF.
- **SKILL.md**: vendored coding skills + marketplace skills via argus.
- **Routines**: scheduled "weekly tech debt sweep", "nightly dependency audit".

### L4 Runtime
**Anthropic Agent Teams** ([250](250-anthropic-agent-teams.md)) for multi-reviewer parallel review. **LangGraph** ([259](259-langgraph-deep-dive.md)) for state-machine SWE workflows (read → plan → edit → test → fix → commit). **Hybrid runtime** is the practical choice.

### L5 Security
- **Prompt injection**: critical — orion reads attacker-controllable code, issues, PR descriptions.
- **Supply chain**: SBOM of installed packages; signature verification on dependency installs.
- **Isolation**: **worktree per task** mandatory; container for code execution; micro-VM for untrusted-input code execution.
- **Bright-line**: `git_push`, `pip_install`, `npm_install`, `force_push`, `branch_delete`, `prod_config_change`, `delete_file_outside_worktree`.

### L6 Operations
- **Observability**: per-step ACI productivity metrics (wasted-step ratio per [236](236-tool-use-and-aci-scaling.md)).
- **Eval**: SWE-bench Verified + custom internal regression suite.
- **Durability**: LangGraph checkpointer; idempotency on git operations; saga for multi-PR workflows.
- **SRE**: SLO on PR-merge-rate, eval-pass-rate, P99 task latency.

### L7 Compliance
**SOC 2** for enterprise deployment. **EU AI Act**: usually limited risk for internal coding; high-risk if used in critical-infrastructure code. **License auditing** in SBOM.

## Phased rollout

| Phase | Scope | Duration |
|---|---|---|
| **P1** | L1-L2 + custom ACI + LangGraph workflow | 90 days |
| **P2** | L3 + L4 (Agent Teams for parallel review) | 90 days |
| **P3** | L5 Security (worktree + container + bright-line) + L6 Operations | 90 days |
| **P4** | L7 Compliance + integration with Polaris / Mentat-Learn | 90 days |

## One-line takeaway

**Orion-Code adopts the seven-layer stack as the SWE-bench-class coding agent — custom-ACI for 4× per-step productivity, worktree-per-task isolation as load-bearing security control, Agent Teams for parallel multi-reviewer code review (security + performance + coverage), LangGraph state-machine for the read→plan→edit→test→fix→commit workflow, and bright-line gates on every git push / package install / prod-config change; the highest-leverage L3 lever is the custom ACI, the highest-leverage L5 lever is worktree isolation.**
