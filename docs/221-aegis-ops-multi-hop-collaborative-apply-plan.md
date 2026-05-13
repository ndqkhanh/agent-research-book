# 221 — Aegis-Ops Apply Plan: System-as-User SRE Agent — the Canonical "No ICPEA" Pattern

> **Disambiguation.** This file is the **Aegis-Ops–specific** deep apply plan, extending the per-project apply-plan family ([203](203-polaris-multi-hop-reasoning-apply-plan.md), [208-210](208-lyra-multi-hop-collaborative-apply-plan.md), [212-214](218-atlas-research-multi-hop-collaborative-apply-plan.md)). Aegis-Ops's role in the portfolio: **production SRE / ops agent** baselined against Bedrock Agents and runbook automation, evaluated on LinuxArena (1,671 legit + 184 safety) and internal incident suite — and the canonical example in the portfolio of a **system-as-user** project where ICPEA personal memory is **deliberately omitted** because the user is a system (an incident, an alert, a runbook), not a person.

## One-line definition

A staged plan to fold multi-hop reasoning + collaborative-AI techniques into Aegis-Ops — the **production SRE agent** baselined to match LinuxArena legit task success at <5% undetected sabotage at 1% FPR — where the apply pattern *inverts* the collaborative-AI canon: ICPEA five-layer personal memory is **off** (the user is a system); per-agent layer access control is **on** (different runbook contexts isolate); per-operator constitution (not per-user) captures the on-call engineer's risk preferences; and the multi-hop substrate is sparingly applied because most ops tasks are short-horizon execution, not deep multi-hop reasoning.

## §1 — Why Aegis-Ops is the canonical "no ICPEA" project

The collaborative-AI canon ([206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md)) treats typed personal memory (ICPEA) as a load-bearing axis. That makes sense for projects where the "user" is a person (Polaris researcher, Lyra developer, Mentat assistant client). Aegis-Ops's user is a **system** — an incident, an alert, a runbook execution context. Applying ICPEA here would either be (a) misleading (the "user" doesn't have stable Identity) or (b) dangerous (Identity-shaped data crossing tenant boundaries in shared infra is a compliance failure).

What replaces ICPEA in Aegis-Ops:

- **Per-runbook context isolation** instead of per-user memory. Different runbook executions are isolated trust contexts; a runbook for tenant A cannot read runbook context from tenant B.
- **Per-operator constitution** instead of per-user constitution. The on-call engineer (the human-in-the-loop) has serialised risk preferences (be conservative, alert before mutating, prefer dry-run); these compose with the runbook's own policy.
- **Trajectory-replay forensic record** instead of personal-memory Activity layer. Every action is logged for incident review; nothing personal is stored.

The multi-hop substrate is also sparingly applied. Most ops tasks ("restart service X," "check disk usage on host Y," "rotate the cert on Z") are short-horizon and single-shot. Multi-hop techniques apply to the subset of ops tasks that *are* multi-hop:
- **Cross-system root-cause analysis** ("CPU spike on host A is correlated with traffic from B which is correlated with deploy at time T") — multi-hop reasoning.
- **Multi-step incident remediation with conditional branches** — agentic, but not multi-hop QA.
- **Runbook-of-runbooks composition** ("execute incident-response runbook then escalation runbook") — composition, not retrieval.

So Aegis's apply plan looks like: pure-function agents + verifier gates Tier-0 (universal); multi-hop substrate *narrowly applied* to root-cause analysis only; collaborative-AI substrate *substantially modified* (no ICPEA, per-operator constitution only, branching as runbook-strategy UX, Pages as runbook documentation, voice/screen-share off-niche).

Take this seriously and three things change. (1) Aegis-Ops becomes the canonical example for *every other system-as-user project* (Cipher-Sec, Quanta-Proof, Vertex-Eval, Gnomon) of how to apply the collaborative-AI canon without conflating system context with personal data. (2) The argus marketplace integration (ops-tool MCPs: k8s, AWS, GCP, etcd) is the largest collaborative-AI lift, not the personalisation lift. (3) LinuxArena sabotage detection at 1% FPR is the central safety target, and the verifier gate surface needed for it is shared with Orion-Code.

## §2 — The Aegis-Ops × technique mapping table

Aegis-Ops's existing module structure is `src/aegis_ops/`, with vendored `harness_core/`, `skills/`, `tests/`, `docs/`. The mapping table reflects the system-as-user posture:

| Technique | Source | Aegis module | Tier | Effort | Payoff |
|---|---|---|---|---|---|
| **Pure-function + safety substrate (Aegis's primary axis)** | | | | | |
| Pure-function agents | [202](202-multi-agent-multi-hop-reckoning-2026.md) §4 | `harness_core/orchestration/pure_function.py` | 0 | S | Replayable, sabotage-detection prerequisite |
| Per-runbook context isolation (replaces per-user memory) | new | `src/aegis_ops/context/runbook_isolation.py` | 0 | M | Tenant A's runbook context invisible to Tenant B |
| Per-agent layer access control on runbook context | [205](205-lobehub-collaborative-teammate-platform.md) §2.1 (adapted) | `src/aegis_ops/context/access.py` | 0 | M | Runbook subagents see only declared context layers |
| LinuxArena sabotage-detection eval at 1% FPR | [projects/README.md](../projects/README.md), [26-linuxarena-production-agent-safety](26-linuxarena-production-agent-safety.md) | `tests/benchmarks/linuxarena_sabotage.py` | 0 | M | Critical safety eval |
| Verifier-evaluator loop (composed gates) | [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), shared with Orion | `src/aegis_ops/verifier/` | 0 | M | Multi-axis gate (dry-run + diff + policy + permission) before any mutation |
| Trajectory-replay forensic record | [188-witness-provenance-memory-techniques-synthesis](188-witness-provenance-memory-techniques-synthesis.md) | `src/aegis_ops/forensic/replay_record.py` | 0 | M | Every action replayable for incident review |
| Per-operator constitution | [206](206-collaborative-ai-canon-2026.md) §5 (adapted) | `src/aegis_ops/constitution/` | 1 | M | On-call engineer's risk preferences serialised |
| **Multi-hop substrate (narrow application)** | | | | | |
| HippoRAG-2 over runbook + log + alert graph | [200](200-graph-grounded-multi-hop-retrieval.md), promoted from `polaris-core` | `harness_core/multi_hop/hipporag.py` (vendored) | 1 | M | Cross-system root-cause analysis substrate |
| Self-Ask / IRCoT for multi-step root-cause | [199](199-multi-hop-reasoning-techniques-arc.md) Phase 1 | `src/aegis_ops/reasoning/{self_ask,ircot}.py` | 1 | S | Externalises bridge entity in cross-system causation chains |
| Chain-of-Note per-log-source filter | [199](199-multi-hop-reasoning-techniques-arc.md) Phase 3 | `harness_core/gates/chain_of_note.py` | 1 | S | Filter noisy log sources before reasoning |
| BELLE-style ops-task router | [202](202-multi-agent-multi-hop-reckoning-2026.md) §1 | `src/aegis_ops/routing/ops_task_router.py` | 1 | S | Single-shot vs multi-step vs root-cause routing |
| **Collaborative-AI substrate (modified for system-as-user)** | | | | | |
| ICPEA personal memory | NOT APPLIED (system-as-user) | — | — | — | Off-niche; use per-runbook context instead |
| Branching as runbook-strategy UX | [206](206-collaborative-ai-canon-2026.md) §4 | `src/aegis_ops/conversation/branching.py` | 1 | M | "Three remediation options for this incident" |
| Lobe Pages-style co-authored runbook | [205](205-lobehub-collaborative-teammate-platform.md) §2.2 | `src/aegis_ops/output/pages.py` | 1 | M | Co-authored runbook with timeline; post-mortem document |
| MCP-first marketplace via argus trust gate | [205](205-lobehub-collaborative-teammate-platform.md) §2.3, [209](209-argus-multi-hop-collaborative-apply-plan.md) | `src/aegis_ops/tools/mcp_marketplace.py` | 1 | S | Trust-gated ops-tool MCP installation (k8s, AWS, GCP, etcd, Datadog, etc.) |
| Voyager-line runbook auto-creation | [167](167-autoskill-experience-driven-lifelong-learning.md), [197](197-argus-omega-vol-3-recursive-skills-curator.md) | `src/aegis_ops/skills/runbook_auto_creator.py` (submits to argus.curator) | 1 | M | Successful incident-response patterns become reusable runbooks |
| Voice + screen-share | NOT APPLIED (system-as-user; CLI-first) | — | — | — | Off-niche |
| **Cross-cutting discipline** | | | | | |
| Equal-budget benchmark harness | [202](202-multi-agent-multi-hop-reckoning-2026.md) §3 | `harness_core/evals/equal_budget.py` | 0 | S | Honest baselines vs Bedrock Agents |
| TTC curve plotter | [199](199-multi-hop-reasoning-techniques-arc.md) | `harness_core/evals/ttc_curve.py` | 0 | S | Detect plateau-or-decline on noisy alerts |
| LinuxArena legit task suite + safety suite + internal incident suite | [projects/README.md](../projects/README.md) | `tests/benchmarks/{linuxarena_legit,linuxarena_safety,internal_incidents}.py` | 0 | S | Design-target regression triple |
| gnomon HIR emit | [211](211-cross-project-power-up-plan-with-tradeoffs.md) §4 | `src/aegis_ops/observability/hir_emit.py` | 0 | S | Cross-harness eval signal |

## §3 — Tier 0 (days 1-14): the safety substrate

Aegis's Tier 0 is shaped by the LinuxArena sabotage-detection target. Pure-function agents + verifier + per-runbook context isolation + trajectory-replay record are non-negotiable.

### 3.1 — Pure-function agents

**File.** `harness_core/orchestration/pure_function.py` (shared per [211](211-cross-project-power-up-plan-with-tradeoffs.md) §4).

**What.** Yenugula's pattern. Every agent decision is a pure function of (input, context, tool-results). Side-effecting tools (any tool that mutates state — k8s apply, AWS CLI mutation, Bash command on a target host) go through gated APIs that log + require explicit BL-* permission.

**Why critical for Aegis.** LinuxArena sabotage detection at <5% undetected at 1% FPR requires replay-comparison; non-deterministic agents make this impossible.

### 3.2 — Per-runbook context isolation

**File.** `src/aegis_ops/context/runbook_isolation.py`.

**What.** Each runbook execution gets an isolated context object containing:
- The runbook itself (the ops procedure being executed).
- The target system (host, cluster, service, tenant).
- The trigger (alert, scheduled run, manual invocation).
- The executor (the on-call engineer, if any).
- The trust scope (what credentials this runbook is allowed to use).

Cross-context reads are **forbidden by default**; explicit cross-context permissions require BL-* gate authorization. This is the structural answer to the "no ICPEA" stance — context is per-runbook, not per-user.

### 3.3 — Per-agent layer access control on runbook context

**File.** `src/aegis_ops/context/access.py`.

**What.** Within a runbook execution, sub-agents (the executor, the verifier, the rollback agent) declare which context layers they read. The verifier sees the runbook + the proposed action + the dry-run output, but *not* the credentials. Layered access enforced via PermissionBridge-style runtime checkpoints (cf. [54-semaclaw-general-purpose-agent](54-semaclaw-general-purpose-agent.md)).

### 3.4 — Verifier composing multi-axis gates

**File.** `src/aegis_ops/verifier/`.

**What.** Every mutation goes through a multi-axis gate before execution:
- **Dry-run gate.** Action runs in dry-run mode; output is captured.
- **Diff gate.** Diff between current state and projected post-action state is computed.
- **Policy gate.** Diff is checked against tenant policy (no unauthorised resource creation, no quota violations, no compliance violations).
- **Permission gate.** The runbook's trust scope authorises the action.
- **HITL gate.** For high-risk actions, the on-call engineer must approve explicitly.

Composed gate failure → action aborted; trajectory rolled back; alarm raised. Cf. [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [18-chain-of-verification-self-refine](18-chain-of-verification-self-refine.md).

### 3.5 — Trajectory-replay forensic record

**File.** `src/aegis_ops/forensic/replay_record.py`.

**What.** Every action — agent decision, tool call, gate check, mutation — is logged with full context. The forensic record supports:
- **Replay.** Re-run a past trajectory deterministically (because pure-function).
- **Comparison.** Compare the current trajectory against past trajectories on similar incidents.
- **Audit.** External auditor can step through every action with full reasoning trace.

This is the ops-shaped equivalent of [188-witness-provenance-memory-techniques-synthesis](188-witness-provenance-memory-techniques-synthesis.md)'s witness lattice.

### 3.6 — LinuxArena + internal incident suite in CI

**Files.** `tests/benchmarks/{linuxarena_legit,linuxarena_safety,internal_incidents}.py`.

**What.** Three benchmark suites in nightly CI:
- **LinuxArena legit (1,671 tasks).** Task success rate baseline.
- **LinuxArena safety (184 tasks).** Sabotage-detection rate at 1% FPR.
- **Internal incident suite.** Synthesised or anonymised real incidents from Aegis's deployment history.

Equal-budget + TTC + active-params accounting on every run.

### 3.7 — Equal-budget + TTC + gnomon HIR emit

Cross-cutting per [211](211-cross-project-power-up-plan-with-tradeoffs.md). Especially important for Aegis to compare honestly against Bedrock Agents (which has its own opaque compute model).

## §4 — Tier 1 (days 15-45): multi-hop root-cause + branching + Pages

After Tier 0, Aegis has the safety substrate. Tier 1 adds the *narrow* multi-hop application (root-cause analysis only) + branching + Pages.

### 4.1 — HippoRAG-2 over runbook + log + alert graph

**File.** `harness_core/multi_hop/hipporag.py` (vendored from shared library).

**What.** Aegis's KG nodes are: runbooks (typed by domain), historical incidents (each with a resolved-cause attribution), service dependency graph (k8s services, AWS resources), alert taxonomy. HippoRAG-2 PPR fusion runs over this graph at root-cause analysis time. Single-shot retrieval gives the multi-hop chain "alert X → caused by service Y → which depends on resource Z → which had a recent change at time T."

**Narrow application.** HippoRAG is *only* invoked for root-cause analysis tasks. Single-shot ops tasks ("restart service X") bypass the graph.

### 4.2 — Self-Ask / IRCoT + Chain-of-Note + BELLE router

**Files.** `src/aegis_ops/reasoning/{self_ask,ircot}.py`, `harness_core/gates/chain_of_note.py`, `src/aegis_ops/routing/ops_task_router.py`.

**What.** Standard from [199](199-multi-hop-reasoning-techniques-arc.md) Phase 1 / Phase 3 / [202](202-multi-agent-multi-hop-reckoning-2026.md) §1, applied narrowly to root-cause analysis:
- BELLE router types incoming tasks: *single-shot ops* (~80% of tasks, bypass multi-hop) vs *multi-step incident remediation* (~15%, use Self-Ask) vs *root-cause analysis* (~5%, use IRCoT + HippoRAG).
- Chain-of-Note filters log sources before reasoning (especially valuable when alert traffic is noisy).
- Self-Ask / IRCoT externalise the causation chain ("the bridge entity is the failing service; the next hop is its upstream dependency").

### 4.3 — Branching as runbook-strategy UX

**File.** `src/aegis_ops/conversation/branching.py`.

**What.** When Aegis's BELLE router identifies a root-cause-analysis task with multiple plausible remediations, present them as branches the on-call engineer picks from:
- Branch A: restart the service (cheap; risks recurrence).
- Branch B: roll back the deploy (more disruptive; addresses likely root cause).
- Branch C: scale up upstream dependency (mitigates symptom; doesn't fix root cause).

Each branch is a separate dry-run trajectory. The engineer picks; the chosen branch executes through the verifier gate.

### 4.4 — Lobe Pages-style co-authored runbook

**File.** `src/aegis_ops/output/pages.py`.

**What.** Two use cases:
- **Pre-incident.** A new runbook is co-authored with the SRE team in a Pages-style document.
- **Post-incident.** A post-mortem is co-authored after each incident; timeline preserves the trajectory + decisions + outcomes.

The post-mortem Pages doc feeds back into the runbook auto-creator (§4.6).

### 4.5 — argus trust-gated MCP marketplace

**File.** `src/aegis_ops/tools/mcp_marketplace.py`.

**What.** Consume `argus.HostAdapter.install_mcp(...)`. Aegis's typical MCP needs: kubernetes, AWS, GCP, Azure, etcd, Datadog, Prometheus, Grafana, PagerDuty, Slack, GitHub Actions. Each MCP server gets an argus trust verdict before installation.

Aegis is one of the largest MCP marketplace consumers in the portfolio because ops is tool-heavy.

### 4.6 — Voyager-line runbook auto-creation

**File.** `src/aegis_ops/skills/runbook_auto_creator.py`.

**What.** Successful incident-response patterns (incident resolved + post-mortem published + similar pattern recurred ≥2 times) submit to `argus.curator.submit_trajectory(...)`. Argus's curator gates promotion via held-out incident-replay eval + surrogate verifier; promoted runbooks land in `skills/runbooks/auto/` with full provenance.

### 4.7 — Per-operator constitution

**File.** `src/aegis_ops/constitution/`.

**What.** ICAI-style 3-principle constitution per on-call engineer:
- "Always dry-run before mutating production."
- "Never auto-mutate during business hours; require HITL."
- "Default to roll-back over forward-fix when in doubt."

Composed with runbook policy; runbook policy wins on conflict (because runbooks encode tenant-specific compliance).

## §5 — Tier 2 (days 46-90): trained executor + advanced safety

### 5.1 — Search-R1 RL with ops-grounded reward

**File.** `src/aegis_ops/rl/search_r1.py` (deferred — Tier 2 only if data permits).

**What.** Reward shaping for ops:
- Incident resolved within SLA = +1
- Mean-time-to-resolution improvement vs baseline runbook = bonus
- Verifier gate violation = strong penalty
- BL-SABOTAGE flag = strong penalty
- Cost overrun (compute, API, downtime) = penalty
- Operator-rolled-back action = penalty

Backbone: small ops-focused fine-tune of a coding model (Qwen2.5-Coder-7B). Caveat: real ops data is often sensitive; training data must be heavily anonymised or synthesised.

### 5.2 — Cross-tenant safety hardening

**File.** `src/aegis_ops/security/cross_tenant.py`.

**What.** When Aegis is deployed multi-tenant, additional gates ensure tenant A's actions cannot affect tenant B's resources, credentials, or context. Confidential VM (SEV-SNP / TDX) deployment for the most sensitive tenants.

## §6 — Five sequencing decisions worth defending

### Decision 1 — ICPEA personal memory NOT applied

This is the canonical "no ICPEA" project. The user is a system; applying ICPEA would either be misleading or dangerous. Per-runbook context isolation replaces it.

### Decision 2 — Multi-hop substrate Tier 1, narrowly applied

~80% of ops tasks are single-shot. HippoRAG / Self-Ask / IRCoT only fire for root-cause analysis (~5%) and multi-step remediation (~15%). Defaulting to multi-hop for everything wastes compute.

### Decision 3 — Pure-function agents Tier 0 (sabotage-detection prerequisite)

LinuxArena sabotage detection at 1% FPR is impossible without replay-able trajectories. This is the same Tier-0 lift as Orion-Code; the verifier surface is shared.

### Decision 4 — Verifier composes dry-run + diff + policy + permission + HITL

A single-axis gate is insufficient for production ops. Multi-axis composition catches more failure modes.

### Decision 5 — Voice + screen-share NOT applied

Aegis's user (the on-call engineer) is at a CLI / dashboard, not in a voice conversation. Voice is off-niche; CLI + Pages is the right surface.

## §7 — Risks and mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| ICPEA accidentally introduced (someone copy-pastes from another project) | Med | High | Lint rule rejecting `from harness_core.memory.icpea import` in Aegis; documented in CONTRIBUTING.md |
| Cross-tenant context leak | Low | Critical | Per-runbook context isolation + audit log + cross-tenant pen-testing in CI |
| Verifier gate over-blocks legitimate actions | High | Med-High | Tunable per-runbook thresholds; HITL override path; weekly false-positive review |
| Search-R1 reward-hacks SLA metric | Med | High | Held-out adversarial-incident eval + sample-based human review |
| HippoRAG over-applied to single-shot tasks | Med | Low-Med | BELLE router routes single-shot bypass; per-task latency monitoring |
| Argus marketplace SPOF on tool installation | Low | High (ops cannot run without tools) | Local cache; offline install path; fallback to vendored tool list |
| Per-operator constitution conflicts with runbook policy | Med | Med | Documented precedence: runbook policy wins; constitution flags but doesn't override |
| Trajectory-replay record fills storage | Med | Low-Med | Per-tenant retention policy; lossless compression; cold-storage tier for older traces |
| LinuxArena safety eval regresses with verifier changes | Med | Critical | Per-PR mandatory benchmark; alarm on >0.5pt FPR regression |
| Equal-budget benchmark misrepresents human-in-the-loop pauses | High | Low-Med | Wall-clock vs compute-only metrics reported separately; HITL pauses logged |

## §8 — Concrete day-by-day Tier 0 checklist

A 14-day Tier 0 plan:

- **Day 1-2** — `harness_core/orchestration/pure_function.py` adopted; agent loop refactored.
- **Day 3-4** — `src/aegis_ops/context/{runbook_isolation,access}.py`; first cross-tenant pen-test.
- **Day 5-7** — `src/aegis_ops/verifier/` extended with dry-run + diff + policy + permission + HITL gates.
- **Day 8-9** — `src/aegis_ops/forensic/replay_record.py`; first replay test on a synthesised incident.
- **Day 10-11** — `tests/benchmarks/{linuxarena_legit,linuxarena_safety,internal_incidents}.py`; first benchmark baseline.
- **Day 12** — `harness_core/evals/{equal_budget,ttc_curve,active_params}.py`.
- **Day 13** — `src/aegis_ops/observability/hir_emit.py` to gnomon.
- **Day 14** — Tier 0 retro: LinuxArena legit task success + safety FPR + replay-comparison success rate; sign-off for Tier 1.

## §9 — How Aegis-Ops sets the system-as-user pattern for Cipher / Quanta-Proof / Vertex / Gnomon

Aegis-Ops is the canonical example of the system-as-user pattern. Other system-as-user projects in the portfolio adapt the same shape:

| Project | User shape | ICPEA | Per-X context isolation | Per-X constitution |
|---|---|---|---|---|
| **Aegis-Ops** | System (incident, runbook) | OFF | per-runbook context | per-operator |
| **Cipher-Sec** | System (security target, scope) | OFF | per-engagement scope | per-operator + per-engagement |
| **Quanta-Proof** | System (theorem, proof goal) | OFF | per-proof context | per-mathematician (still per-user, but very thin) |
| **Vertex-Eval** | System (benchmark, suite) | OFF | per-suite context | none (eval platform is operator-driven) |
| **Gnomon** | System (harness, trace bundle) | OFF | per-harness adapter | none |

The pattern: **replace ICPEA with per-context isolation; keep per-operator/user constitution where there's a stable HITL human; otherwise omit constitution entirely.**

## §10 — Cross-references

- [203](203-polaris-multi-hop-reasoning-apply-plan.md), [208-210](208-lyra-multi-hop-collaborative-apply-plan.md), [212-214](218-atlas-research-multi-hop-collaborative-apply-plan.md) — sibling apply plans.
- [211-cross-project-power-up-plan-with-tradeoffs](211-cross-project-power-up-plan-with-tradeoffs.md) — strategy doc.
- [26-linuxarena-production-agent-safety](26-linuxarena-production-agent-safety.md) — LinuxArena baseline.
- [38-claw-eval](38-claw-eval.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md) — verifier lineage.
- [54-semaclaw-general-purpose-agent](54-semaclaw-general-purpose-agent.md) — PermissionBridge inspiration.
- [40-harness-engineering-principles](40-harness-engineering-principles.md), [41-product-control-plane](41-product-control-plane.md), [44-four-pillars-harness-engineering](44-four-pillars-harness-engineering.md), [124-agent-level-production-patterns](124-agent-level-production-patterns.md), [125-system-level-production-patterns](125-system-level-production-patterns.md), [150-temporal-durable-execution-substrate](150-temporal-durable-execution-substrate.md) — production-pattern context.
- [188-witness-provenance-memory-techniques-synthesis](188-witness-provenance-memory-techniques-synthesis.md) — forensic-record substrate.
- [180-argus-skill-router-agent-design](180-argus-skill-router-agent-design.md) — argus integration.

## §11 — One-paragraph summary

Aegis-Ops's multi-hop + collaborative-AI subsystem ships in three tiers: Tier 0 (14 days) hardens the **safety substrate** with pure-function agents (sabotage-detection prerequisite) + per-runbook context isolation (replaces ICPEA personal memory) + per-agent layer access control + verifier composing dry-run/diff/policy/permission/HITL gates + trajectory-replay forensic record + LinuxArena legit/safety/internal-incident benchmarks + equal-budget + TTC + gnomon HIR emit; Tier 1 (30 days) **narrowly applies** the multi-hop substrate (HippoRAG-2 over runbook+log+alert graph for root-cause analysis only, ~5% of tasks) plus Self-Ask/IRCoT for multi-step remediation, BELLE ops-task router, branching as runbook-strategy UX (three remediation options for the on-call engineer to pick from), Lobe Pages co-authored runbook + post-mortem, argus trust-gated MCP marketplace (one of the largest consumers in the portfolio), Voyager runbook auto-creation, and per-operator constitution; Tier 2 (60 days) optionally trains an ops-grounded Search-R1 (data-permitting) and adds cross-tenant safety hardening with confidential VMs. The five sequencing decisions defended in §6 keep ICPEA off (system-as-user), apply multi-hop substrate narrowly (~5% of tasks), keep pure-function agents Tier-0 (LinuxArena prerequisite), compose multi-axis verifier gates, and omit voice/screen-share (off-niche). Aegis-Ops sets the canonical pattern for Cipher-Sec / Quanta-Proof / Vertex-Eval / Gnomon — every system-as-user project in the portfolio inherits this shape.

**The one-line takeaway for harness designers:** Aegis-Ops is the **system-as-user** sibling of Polaris/Atlas — apply pure-function agents + verifier gates Tier-0, replace ICPEA with per-runbook context isolation, narrow the multi-hop substrate to root-cause analysis only, and let LinuxArena sabotage detection at 1% FPR be the central safety target the verifier composes around.
