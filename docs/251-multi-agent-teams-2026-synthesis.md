# 251 — Multi-Agent Teams 2026 Synthesis: from MetaGPT SOPs to MAST failure taxonomy

**Anchor papers (2025–2026, non-canonical, complementing existing corpus).** Mert Cemri, Melissa Pan, Shuyi Yang, Lakshya Agrawal, Bhavya Chopra, Rishabh Tiwari, Kurt Keutzer, Aditya Parameswaran, Dan Klein, Kannan Ramchandran, Matei Zaharia, Joseph E. Gonzalez, Ion Stoica — *Why Do Multi-Agent LLM Systems Fail?* (MAST) — arXiv:2503.13657 — UC Berkeley — NeurIPS 2025; repo: https://github.com/multi-agent-systems-failure-taxonomy/MAST. Kunlun Zhu et al. — *MultiAgentBench / MARBLE* — arXiv:2503.01935 — ACL 2025. *Talk Isn't Always Cheap: Failure Modes in Multi-Agent Debate* — arXiv:2509.05396 — Sep 2025. *Stay Focused: Problem Drift in Multi-Agent Debate* — arXiv:2502.19559 — Feb 2025. Yan Dang, Ying Qian et al. — *Puppeteer: Multi-Agent Collaboration via Evolving Orchestration* — arXiv:2505.19591 — May 2025. *AgentOrchestra (TEA Protocol)* — arXiv:2506.12508 — Jun 2025. *A Taxonomy of Hierarchical Multi-Agent Systems* — arXiv:2508.12683 — Aug 2025. Kai Mei et al. — *AIOS: LLM Agent Operating System* — arXiv:2403.16971 — COLM 2025. *OrgAgent: Organize Your Multi-Agent System like a Company* — arXiv:2604.01020 — early 2026. *RL for MAS through Orchestration Traces* — arXiv:2605.02801 — May 2026. *LLM-based Multi-Agent Blackboard System* — arXiv:2510.01285 — Oct 2025. Companion: existing canonical files — [91-metagpt-deep](91-metagpt-deep.md), [92-chatdev](92-chatdev.md), [98-diversity-collapse-mas](98-diversity-collapse-mas.md), [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md), [164-crewai-multi-agent-framework](164-crewai-multi-agent-framework.md), [187-multi-agent-shared-memory-landscape](187-multi-agent-shared-memory-landscape.md), [189-recursive-multi-agent-systems](189-recursive-multi-agent-systems.md), [191-onemancompany-skills-to-talent](191-onemancompany-skills-to-talent.md), [202-multi-agent-multi-hop-reckoning-2026](202-multi-agent-multi-hop-reckoning-2026.md), [224-multi-agent-parallel-scaling](224-multi-agent-parallel-scaling.md), [225-agent-era-scaling-synthesis](225-agent-era-scaling-synthesis.md), [250-anthropic-agent-teams](250-anthropic-agent-teams.md).

**One-line definition.** A 2025–2026 synthesis of where the multi-agent-team field landed: **lead-and-spokes hierarchical architectures dominate** (OrgAgent, AgentOrchestra, Anthropic Agent Teams, [250](250-anthropic-agent-teams.md)), **free-chat debate is empirically harmful past short rounds** (Talk Isn't Always Cheap, Stay Focused; failures rates **41–86.7 %** on SOTA OSS multi-agent systems per MAST's 1,600-trace study), **blackboard architectures outperform point-to-point messaging** on token cost and context-overwrite failure rate (arXiv:2510.01285), and **the consensus production architecture** is structured-channel comms (shared task list + targeted messages + plan approvals + cross-channel verifier) layered on **isolated worktrees** with **per-member token budgets** — exactly the shape Claude Code's Agent Teams ([250](250-anthropic-agent-teams.md)) productized in Feb 2026.

## Why this synthesis matters (the field consolidated; debate is dead, hierarchy won)

Multi-agent teams went through three distinct generations between 2023 and 2026, and the corpus captures that arc piecewise — MetaGPT's SOPs ([91](91-metagpt-deep.md)), ChatDev's waterfall ([92](92-chatdev.md)), MoA's ensembling ([224](224-multi-agent-parallel-scaling.md)), debate (Du-2023), recursive multi-agent ([189](189-recursive-multi-agent-systems.md)), the LobeHub / Mentat-Learn / Hermes collaborative-AI canon ([205](205-lobehub-collaborative-teammate-platform.md), [206](206-collaborative-ai-canon-2026.md)). What was missing was the **failure-taxonomy** generation (MAST), the **dynamic-orchestration** generation (Puppeteer, TEA, RL-on-orchestration-traces), and the **architectural-consensus** moment (Anthropic Agent Teams, OrgAgent). This file is that synthesis.

The headline empirical finding from 2025–2026 is brutal and clarifying. MAST's 1,600-trace audit across the most-used open-source multi-agent systems (CrewAI, AutoGen, LangGraph examples, AgentVerse) finds **failure rates of 41–86.7 %** on real tasks, decomposed into 14 failure modes across three categories: (1) **system design failures** — bad role specification, missing verifier, monolithic-context overrun; (2) **inter-agent misalignment** — agents pursuing different sub-goals, context-overwrite cascades, anchoring on wrong early hypotheses; (3) **task verification failures** — no terminal verifier or weak verifier passes through wrong-answer outputs. The taxonomy is now the standard lens for any new multi-agent system; new papers report MAST-category-rate breakdowns alongside accuracy.

The complementary finding from "Talk Isn't Always Cheap" (arXiv:2509.05396) and "Stay Focused" (arXiv:2502.19559) is that **multi-agent debate — the dominant 2023–2024 multi-agent pattern — actively harms accuracy past 2–3 rounds**. Agents prefer agreement over challenge; problem drift causes the team to quietly redefine the question and converge on the wrong target. Together with [98-diversity-collapse-mas](98-diversity-collapse-mas.md), this is the empirical death of free-chat debate as a *primary* coordination pattern. Debate survives only as a short, gated, cross-model-pair pattern (e.g. one round of structured critique with a separate adjudicator) — not as the team's main loop.

What replaced debate is **structured channels and hierarchy**. OrgAgent (arXiv:2604.01020) makes the company analogy explicit: governance / execution / compliance layers with role-typed artifacts moving between them. AgentOrchestra (arXiv:2506.12508) and Puppeteer (arXiv:2505.19591) train an *orchestrator* that decides spawn / delegate / aggregate decisions — RL on orchestration traces (arXiv:2605.02801) treats those decisions as the action space. The Linux Foundation's A2A protocol (https://a2a-protocol.org) — 150+ orgs as of 2026, on major clouds — is the cross-vendor shape this consensus expects.

Take this synthesis seriously and three things change. **First**, you build agent teams as **lead-and-spokes hierarchies with structured channels** — shared task list, targeted messaging, plan-approval gates — not free-chat GroupChat. **Second**, you adopt a **MAST-aware audit pipeline**: every multi-agent trace tagged for failure-mode replay; failures harvested into per-role skill memory ([81-reasoningbank](81-reasoningbank.md), [233-memory-scaling-for-agents](233-memory-scaling-for-agents.md)). **Third**, you understand the **consensus architecture is filesystem-cheap and protocol-light**: shared filesystem for task list (lock files), structured messages between named members, isolated worktrees for blast-radius containment, cross-model verifier at integration — none of these need a heavyweight protocol; A2A enters only when crossing vendor boundaries.

## The failure taxonomy (MAST, arXiv:2503.13657)

MAST's three clusters of 14 failure modes:

### Cluster 1 — system design failures (5 modes)

- **F1: Role under-specification.** Roles defined too vaguely; agents drift into overlapping or null behaviour.
- **F2: Missing verifier.** No terminal correctness check; wrong outputs propagate.
- **F3: Tool / capability mismatch.** Agent assigned a task it lacks tools to complete.
- **F4: Context-window overrun.** Shared context grows past effective length ([234-context-length-scaling](234-context-length-scaling.md)).
- **F5: Schedule deadlock.** Circular dependencies; no agent can proceed.

### Cluster 2 — inter-agent misalignment (5 modes)

- **F6: Goal divergence.** Agents pursuing inconsistent sub-goals.
- **F7: Context overwrite cascade.** Agent A's output overwrites Agent B's working state.
- **F8: Anchoring on wrong early hypothesis.** First agent's incorrect framing locks subsequent agents.
- **F9: Debate collapse / agreement bias.** Talk-Isn't-Always-Cheap pattern ([98-diversity-collapse-mas](98-diversity-collapse-mas.md)).
- **F10: Problem drift.** Stay-Focused pattern; team redefines the task.

### Cluster 3 — task verification failures (4 modes)

- **F11: Verifier passes wrong output.** Weak verifier; calibration failure ([97-qwen-prm](97-qwen-prm.md), [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md)).
- **F12: Verifier-generator monoculture.** Same model family; correlated errors.
- **F13: No terminal commit gate.** Output declared done without verification.
- **F14: Silent retry past bright-line.** Daemon retries past a gate it should escalate ([13-daemon-and-scheduler](../projects/polaris/docs/blocks/13-daemon-and-scheduler.md), [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md)).

**Failure rates on production OSS systems** (MAST §4): CrewAI ~52 %, AutoGen ~47 %, LangGraph examples ~41 %, AgentVerse ~86.7 %. The rate inversely correlates with how structurally-constrained the comms channels are — AgentVerse's free-form town-hall pattern fails most often.

## The communication-protocol shift

| Pattern | Era | Tokens / round | Failure modes (MAST) |
|---|---|---|---|
| Free-chat GroupChat (AutoGen pre-2025) | 2023–2024 | High (each agent re-reads all) | F4, F7, F8, F9, F10 |
| Sequential SOP (MetaGPT, ChatDev) | 2023–2025 | Medium (typed artifacts) | F1, F2, F8 |
| Targeted point-to-point messaging | 2024–2026 | Medium-low | F4 (less), F7 |
| Blackboard (arXiv:2510.01285) | 2025–2026 | Low (single shared store, claim-based) | Reduces F4, F7, F8 |
| Lead-and-spokes shared task list (Anthropic Agent Teams [250](250-anthropic-agent-teams.md)) | 2026 | Low (filesystem state, targeted messages) | Lowest cluster-2 rates reported |
| Hierarchical / OrgAgent (arXiv:2604.01020) | 2026 | Low (multi-layer, scoped) | Strong on F1, F6, F12 |

Blackboard architectures revive the classic 1980s AI pattern: agents pull from a shared posting wall instead of point-to-point. The 2510.01285 paper shows this **cuts tokens** (no per-pair re-reading) and **reduces context-overwrite failures** (single source of truth). Anthropic Agent Teams is essentially a blackboard pattern (`~/.claude/tasks/{team-name}/`) with optional targeted messaging on top.

## Hierarchical orchestration: the dominant 2026 architecture

| Paper | Layers | Innovation |
|---|---|---|
| OrgAgent (arXiv:2604.01020) | Governance / Execution / Compliance | Three-layer corporate analog; compliance layer is the verifier-cluster |
| AgentOrchestra (arXiv:2506.12508) | Hierarchical Planning Agent → Browser Agent + Deep Researcher Agent | TEA (Tool-Environment-Agent) protocol; orchestrator-worker reference |
| Hierarchical MAS Taxonomy (arXiv:2508.12683) | Hierarchies / teams / coalitions / holarchies / markets | Quantifies efficiency-vs-robustness tradeoff |
| Puppeteer (arXiv:2505.19591) | Single trained orchestrator + N workers | RL-trained dynamic routing; transitions exploration → synergistic |
| RL on Orchestration Traces (arXiv:2605.02801) | Single orchestrator | Treats spawn/delegate/aggregate as RL action space; Kimi K2.5 PARL is first deployed example |
| AIOS (arXiv:2403.16971) | Kernel: scheduler / context manager / memory manager / tool service | 2.1× throughput on multi-agent serving |

The consensus production shape is **single lead orchestrator** + **3–6 specialist workers** + **structured-artifact handoffs** + **terminal verifier on a different model family**. The orchestrator is either prompt-engineered (CrewAI, Anthropic Agent Teams, Ruflo-style Claude Code swarms) or RL-trained (Puppeteer, Kimi K2.5 PARL); the trained variant is the 2026 frontier.

## The repo landscape (high-star, May 2026)

| Repo | Stars | Distinctive |
|---|---:|---|
| [crewAIInc/crewAI](https://github.com/crewAIInc/crewAI) | ~50.8 k | Declarative role-spec API (Agent + Task + Crew + Process); 2 B agent runs reported |
| [microsoft/autogen](https://github.com/microsoft/autogen) | ~42 k | Conversational MAS, GroupChat, code-execution agents |
| [langchain-ai/langgraph](https://github.com/langchain-ai/langgraph) | ~12.8 k | Stateful graph runtime; checkpoints, audit trails, rollback |
| [FoundationAgents/MetaGPT](https://github.com/FoundationAgents/MetaGPT) | ~50 k+ | PM/Architect/Engineer/QA SOPs; structured artifact handoffs |
| [OpenHands/OpenHands](https://github.com/OpenHands/OpenHands) | top-4 OSS agent | Multi-agent SWE platform; SDK + CLI + GUI + cloud |
| [openai/openai-agents-python](https://github.com/openai/openai-agents-python) | rapidly growing | Production successor to Swarm; Agents/Tools/Handoffs/Guardrails |
| [google/adk-python](https://github.com/google/adk-python) | growing | Code-first multi-language ADK; native A2A integration |
| [a2aproject/A2A](https://github.com/a2aproject/A2A) | ~22 k+ | LF A2A protocol; 150+ orgs, on major clouds |
| [agntcy](https://github.com/agntcy) | 46 repos | Cisco-led; OASF schema + ACP; complements A2A |
| [agiresearch/AIOS](https://github.com/agiresearch/AIOS) | growing | Agent kernel reference impl |
| [VoltAgent/awesome-claude-code-subagents](https://github.com/VoltAgent/awesome-claude-code-subagents) | community | 100+ specialist Claude Code subagents |
| [wshobson/agents](https://github.com/wshobson/agents) | community | 44 production specialists across SWE lifecycle |
| [ruvnet/ruflo](https://github.com/ruvnet/ruflo) | GitHub page reports ~49.5 k | Claude Code orchestration/swarm platform with 98 agents, 60+ commands, 30 skills, MCP server, hooks, daemon, self-learning memory, RAG/GraphRAG plugins, and federation; treat scale/performance claims as repo-reported |

## Empirical results (table — composite from MAST + MultiAgentBench)

**Table 1 — MAST failure rates by system (1,600-trace audit, March 2025)**

| System | Failure rate | Dominant cluster |
|---|---:|---|
| AgentVerse | 86.7 % | Cluster 2 (inter-agent) |
| CrewAI examples | ~52 % | Cluster 1 (system design) |
| AutoGen GroupChat | ~47 % | Cluster 2 + 3 |
| LangGraph multi-agent | ~41 % | Cluster 1 |
| MetaGPT SOPs | ~38 % | Cluster 3 (verification) |

Lower failure rates correlate with **structured-channel** comms and **terminal verifier** presence.

**Table 2 — MultiAgentBench topology comparison (composite)**

| Topology | Best-fit task | Performance |
|---|---|---:|
| Star (lead + workers) | Software dev, parallel review | High |
| Chain (sequential) | SOP-driven workflows | Medium |
| Tree (hierarchical) | Research, multi-domain | High |
| Graph (mesh) | Open-ended exploration | Highest on research tasks |

**Table 3 — Token cost vs team size (Anthropic Agent Teams disclosed)**

| K (teammates) | Token multiplier (plan mode) | Wall-clock vs single |
|---:|---:|---:|
| 1 (solo) | 1× | 1× |
| 3 | ~5× | ~0.4× |
| 5 | ~7× | ~0.3× |
| 8 | ~11× | ~0.25× |

ROI is positive when wall-clock saved exceeds dollars spent — true for parallel review, false for sequential reasoning.

## Variants and ablations across the field

- **Trained orchestrators** — Puppeteer (RL), Kimi K2.5 PARL (production); the frontier of 2026.
- **Blackboard vs point-to-point** — blackboard wins on token cost and overwrite-prevention.
- **Cross-model verifier pair** — single most-effective antidote to F12 monoculture.
- **Per-role memory tier** — each role has its own ReasoningBank slice ([81-reasoningbank](81-reasoningbank.md), [233-memory-scaling-for-agents](233-memory-scaling-for-agents.md)).
- **Plan-approval gating** — lead approves teammate plans before execution; reduces F8 anchoring.
- **Per-member token budgets with handback** — worker that hits budget hands back with summary.
- **Worktree isolation per agent** — blast-radius limited to one branch.
- **A2A for cross-vendor** — Linux Foundation protocol; not yet needed for in-team comms.
- **MCP for tool exposure** — orthogonal to inter-agent comms; tool protocol, not agent protocol.
- **SKILL.md portable definitions** — Anthropic + OpenAI Codex aligned Dec 2025; reusable specialist roles.

## Failure modes and limitations of the consensus architecture

- **Lead-orchestrator is a single point of failure.** No leadership transfer; lead crash = team crash.
- **Token cost ceiling.** 7× at K=5 caps team size economically.
- **Coordination overhead at K > 6.** Marginal-return per added teammate falls steeply.
- **Filesystem coordination requires single-machine or shared FS.** Distributed teams need [253-tailscale-nats-mesh-for-distributed-agents](253-tailscale-nats-mesh-for-distributed-agents.md) layer.
- **Cross-runtime interop limited without A2A.** Anthropic team can't natively talk to a CrewAI team.
- **Trained orchestrators are expensive.** RL on orchestration traces requires significant compute (Puppeteer, Kimi K2.5).
- **Verifier as bottleneck.** Single terminal verifier becomes the latency floor.
- **Plan-mode multiplier is non-negotiable.** Without it, teams skip the gate and waste tokens.
- **Logging requires MAST-aware structure.** Without typed failure-mode tags, replay is impossible.

## When to use, when not

**Build a multi-agent team architecture** when (a) the workload has natural parallelism (multi-file refactors, multi-reviewer code review, multi-hypothesis investigation, multi-domain research), (b) you can afford 5–7× token cost on plan-mode runs, (c) you have or can train a verifier on a different model family, (d) workflows have clear role boundaries with disjoint file scope. The strongest cases are **software engineering teams** (PM/Arch/Eng/QA), **deep research teams** (Researcher/Critic/Synthesizer), and **review teams** (Security/Performance/Coverage).

**Skip multi-agent and use single-agent + tools** when the workflow is sequential reasoning, when budget cannot absorb 5–7× costs, when role boundaries are unclear, when no good verifier exists for your domain, when MAST audit reveals your failure-mode rate exceeds 50 % without major architectural rework. *Multi-agent is not a capability multiplier on sequential reasoning; it is a parallelism amplifier.*

## Implications for harness engineering (consensus 2026 architecture)

- **Lead-and-spokes with structured channels** — [250-anthropic-agent-teams](250-anthropic-agent-teams.md) is the productized template; replicate in [02-subagent-delegation](02-subagent-delegation.md), [18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md).
- **Filesystem task list with claim-locking** — file-based coordination state; [03-program-graph](../projects/polaris/docs/blocks/03-program-graph.md), [13-daemon-and-scheduler](../projects/polaris/docs/blocks/13-daemon-and-scheduler.md).
- **Targeted messaging > free-chat GroupChat** — [98-diversity-collapse-mas](98-diversity-collapse-mas.md), [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md).
- **Cross-channel verifier mandatory** — [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md), [19-verifier-cross-channel](../projects/polaris/docs/blocks/19-verifier-cross-channel.md), [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md), [97-qwen-prm](97-qwen-prm.md) — different model family per MAST F12.
- **Per-member token budget with handback** — [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md), [86-frugalgpt](86-frugalgpt.md), [p13-cost-discipline](../projects/polaris/docs/research/p13-cost-discipline.md).
- **Worktree isolation per teammate** — [18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md).
- **Subagent / SKILL.md as role template** — [04-skills](04-skills.md), [09-skill-engine](../projects/polaris/docs/blocks/09-skill-engine.md), [171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md), [175-agent-skills-ecosystem-and-security](175-agent-skills-ecosystem-and-security.md).
- **MAST-aware HIR observability** — [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md), every team trace tagged for failure-mode replay.
- **Bright-line gates on team-spawn and task-completion** — [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md), [08-hooks-and-claim-gate](../projects/polaris/docs/blocks/08-hooks-and-claim-gate.md).
- **Daemon-driven team scheduling** — [13-daemon-and-scheduler](../projects/polaris/docs/blocks/13-daemon-and-scheduler.md), [p7-daemon](../projects/polaris/docs/research/p7-daemon.md), [252-routines-pattern-for-self-hosted-agents](252-routines-pattern-for-self-hosted-agents.md).
- **A2A for cross-vendor inter-agent comms** — when teams cross runtime boundaries; OPS implementation later.
- **Per-role memory tier** — [233-memory-scaling-for-agents](233-memory-scaling-for-agents.md), [187-multi-agent-shared-memory-landscape](187-multi-agent-shared-memory-landscape.md), [05-reasoning-bank](../projects/polaris/docs/blocks/05-reasoning-bank.md), [81-reasoningbank](81-reasoningbank.md).
- **MCP for tool exposure** — [07-model-context-protocol](07-model-context-protocol.md), [17-mcp-adapter](../projects/polaris/docs/blocks/17-mcp-adapter.md), orthogonal to inter-agent.
- **Distributed teams across machines** — [253-tailscale-nats-mesh-for-distributed-agents](253-tailscale-nats-mesh-for-distributed-agents.md).
- **MetaGPT-style structured artifacts** — [91-metagpt-deep](91-metagpt-deep.md), [92-chatdev](92-chatdev.md), [204-metagpt-foundationagents-2026-refresh](204-metagpt-foundationagents-2026-refresh.md) — typed handoff payloads.

**One-line takeaway for harness designers.** **The 2026 multi-agent-team consensus is lead-and-spokes hierarchy + structured channels (shared task list + targeted messaging + plan approvals) + cross-channel verifier + isolated worktrees + per-member token budgets — productized in Claude Code Agent Teams ([250](250-anthropic-agent-teams.md)) and theorized in OrgAgent / AgentOrchestra / Hierarchical-MAS-Taxonomy — and free-chat debate is empirically dead, surviving only as a short cross-model-pair gate, while MAST is now the audit lens any new system must publish against.**
