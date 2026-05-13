# 76 — Ten-Link Synthesis: What the April-2026 Harness-Engineering Landscape Tells Us

**Context.** This synthesis ties together the ten links deep-dived in docs [55](55-hermes-agent-self-improving.md), [62](62-everything-claude-code.md), [68](68-atomic-skills-scaling-coding-agents.md), [69](69-agent-world-self-evolving-training-arena.md), [70](70-voltagent-awesome-ai-agent-papers.md), [71](71-karpathy-skills-single-file-guardrails.md), [72](72-claude-mem-persistent-memory-compression.md), [73](73-multica-managed-agents-platform.md), [74](74-kronos-foundation-model-financial-markets.md), and [75](75-gstack-garry-tan-claude-code-setup.md). Considered together, these ten artifacts — two arXiv papers, five GitHub community projects, one research-paper ledger, one hosted agent, one domain foundation model — capture the **structural state of harness engineering as of April 2026**. The synthesis identifies **ten cross-cutting themes**, three **emergent stack patterns**, two **unresolved open questions**, and a **recommended reading order** for integrating the material.

## The ten artifacts at a glance

| # | Artifact | Type | Scale | Corpus file |
|---|---|---|---|---|
| 1 | Nous Research Hermes Agent | Hosted self-improving agent | 1 hosted product | [55](55-hermes-agent-self-improving.md) |
| 2 | affaan-m/everything-claude-code | Community Claude-Code bundle | Dozens of skills/agents/commands | [62](62-everything-claude-code.md) |
| 3 | arXiv 2604.05013 — Atomic Skills | Research paper | 5 atomic skills, joint RL | [68](68-atomic-skills-scaling-coding-agents.md) |
| 4 | arXiv 2604.18292 — Agent-World | Research paper | 1,978 environments, 19,822 tools | [69](69-agent-world-self-evolving-training-arena.md) |
| 5 | VoltAgent/awesome-ai-agent-papers | Research-paper ledger | 363+ papers, 5 buckets | [70](70-voltagent-awesome-ai-agent-papers.md) |
| 6 | forrestchang/andrej-karpathy-skills | Minimum-ambition CLAUDE.md | ~60 lines, 71,398 stars | [71](71-karpathy-skills-single-file-guardrails.md) |
| 7 | thedotmack/claude-mem | Persistent-memory plugin | 5 hooks + worker + web viewer, 65,040 stars | [72](72-claude-mem-persistent-memory-compression.md) |
| 8 | multica-ai/multica | Managed-agents platform | 8 agent CLIs, 18,350 stars | [73](73-multica-managed-agents-platform.md) |
| 9 | shiyu-coder/Kronos | Domain foundation model | 4.1M-499M params, 20,018 stars | [74](74-kronos-foundation-model-financial-markets.md) |
| 10 | garrytan/gstack | Maximum-ambition Claude-Code stack | 23 skills + 8 tools + browser | [75](75-gstack-garry-tan-claude-code-setup.md) |

Combined stars across the GitHub projects alone: **~258,500**. Combined across ecosystem references (OpenClaw at 247K, awesome-codex-subagents, Multica's cloud adopters): **~500K+ developers** are visibly engaged in this ecosystem. This is not a niche.

## Ten cross-cutting themes

### Theme 1 — Skills are the unit of agent capability in 2026

Six of the ten artifacts explicitly center on skills:

| Artifact | Skills variant |
|---|---|
| Atomic Skills paper (68) | 5 *atomic skills*: code localization, code editing, unit-test generation, issue reproduction, code review |
| karpathy-skills (71) | Single-file behavioral contract (principles, not skills per se — but framed as "karpathy-skills") |
| claude-mem (72) | `mem-search` skill — natural-language memory retrieval |
| Multica (73) | Team-shared workspace-scoped skill library |
| everything-claude-code (62) | Community Claude Skills bundle |
| gstack (75) | 23 specialist skills (CEO, Eng Manager, Staff Engineer, etc.) |

The remaining four treat skills indirectly:
- Hermes (55) — self-improvement is a meta-skill acquisition mechanism.
- Agent-World (69) — environments train skill generalization.
- VoltAgent ledger (70) — many papers cover skill libraries, skill evolution, skill calibration.
- Kronos (74) — domain capability as a "skill" (forecasting) exposed via tool API.

**The unification:** *the atomic unit of agent capability in 2026 is a skill* — a named, scoped, composable capability with a contract (input, output, pre/postconditions). Whether skills are MD files (Claude Skills), model-trained atomic policies (Atomic Skills paper), SQL-indexed team artifacts (Multica), or foundation-model endpoints (Kronos), the abstraction is the same. The research (Atomic Skills, Voyager-style libraries) and practice (Claude Skills, gstack specialists) converge on this substrate.

### Theme 2 — Persistent memory at multiple scales

Seven of the ten touch persistent memory:

| Scale | Artifact | Storage |
|---|---|---|
| Per-session | karpathy-skills (71) | CLAUDE.md (static) |
| Cross-session / per-developer | claude-mem (72) | SQLite + Chroma + Bun worker |
| Hosted / per-user | Hermes Agent (55) | Proprietary vector + graph |
| Team-shared | Multica (73) | Postgres + pgvector |
| Training-time / environment | Agent-World (69) | Self-evolving environment state |
| Foundation-model weights | Kronos (74) | Transformer params |
| Lightweight per-project | gstack (75) | `/learn` skill + git-backed checkpoints |

**The pattern:** memory is the substrate that differentiates one-shot agents from durable agents. As agents move from chatbot → session-spanning → team-shared → self-evolving, memory scales up along the same axis. *Memory is not a feature of agent harnesses — it is the defining constraint.*

The specific implementations converge on:
- **Hybrid retrieval** (semantic + keyword/FTS5). Claude-mem, Multica, and (presumably) Hermes all use this.
- **Progressive disclosure** of retrieval results. Claude-mem's 3-layer pattern is the template.
- **Citation-based auditability**. Claude-mem's observation IDs + web viewer is the template.
- **Decay / forgetting mechanisms**. Claude-mem's Endless Mode (biomimetic), Multica's taste decay (5%/week), Kronos's context windowing all address forgetting.
- **Token economics at scale**. Claude-mem's 10× savings claim is representative of the per-memory-op economics the field is now optimizing.

### Theme 3 — Multi-IDE / multi-host distribution is table stakes

Five of ten artifacts ship to multiple agent surfaces:

| Artifact | Host count | Hosts supported |
|---|---|---|
| karpathy-skills (71) | 3 | CLAUDE.md + Claude Code plugin + Cursor rule |
| claude-mem (72) | 4 | Claude Code, Gemini CLI, OpenCode, OpenClaw |
| Multica (73) | 8 | Claude Code, Codex, OpenClaw, OpenCode, Hermes, Gemini, Pi, Cursor Agent |
| gstack (75) | 10 | Claude Code, Codex, OpenCode, Cursor, Factory Droid, Slate, Kiro, Hermes, GBrain, OpenClaw |
| VoltAgent ledger (70) | ∞ | Research-wide (not a product) |

The remaining five are either single-host (Hermes hosted, Kronos model, arXiv papers) or the distribution model is orthogonal (everything-claude-code is Claude Code-only).

**The pattern:** **if your tooling targets only one agent CLI in 2026, you are leaving ~50% of adoption on the table.** Developers run 2-5 agent CLIs concurrently. The marginal cost of supporting another agent is a config file; the marginal benefit is another 5-20% of the market.

The dominant implementation approach for multi-host:
- One codebase, one install script.
- Auto-detection of which agents are installed on the user's `PATH`.
- Adapter layer that maps the common contract to each CLI's native format.
- Config files (Multica's `runtimes`, gstack's `--host` flag) rather than separate code paths.

### Theme 4 — Research-to-practice gradient is tight and bidirectional

Arrange the ten artifacts along a research-practice axis:

```
RESEARCH ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ PRACTICE

VoltAgent      Atomic         Agent-World    Hermes     Kronos    Multica    claude-mem    gstack     karpathy-   ECC
 ledger         Skills                                                                                  skills
  (70)           (68)            (69)          (55)       (74)      (73)        (72)         (75)        (71)       (62)
```

The distance between research and practice is unusually small. Examples:

- **Atomic Skills → gstack.** The paper's "5 atomic skills" vs gstack's 23 specialist skills — same abstraction at different granularity.
- **Agent-World → Multica.** The paper's environment-scaling substrate becomes Multica's Runtimes abstraction.
- **Hermes' self-improvement → Multica's skill compounding.** Same mechanism (accumulate learnings over time), different scale (per-user vs team-wide).
- **VoltAgent ledger's Memory & RAG bucket → claude-mem's production architecture.** Ledger surfaces the research thesis; claude-mem productizes it.
- **Kronos → specialist tools for coding agents.** Financial FM is a blueprint for domain-specialist tools called by general agents.

**The implication:** harness engineers in 2026 should read research papers and community projects *together* — the gap between them is shrinking, and the leverage of connecting them is growing.

### Theme 5 — Vendor neutrality as a strategic bet

Multica (73), gstack (75), VoltAgent ledger (70), karpathy-skills (71), claude-mem (72) all commit to vendor neutrality. Hermes (55) is the one hosted counter-example — it is vendor-specific (Nous Research's own agent).

**The bet:** agent CLIs (Claude Code, Codex, Cursor Agent, OpenCode, Gemini CLI, etc.) are **commoditizing**. The value shifts from any single CLI to the **orchestration / skills / memory / observability layer above all of them**. Whoever builds that layer as vendor-neutral captures the infrastructure position.

The counter-bet (Hermes-style, Claude-Code-first gstack originally, etc.) is that **one CLI wins** and the best investments are vertical integrations. This bet is not obviously wrong — Claude Code has a strong lead — but the number of vendor-neutral bets suggests the market expects the CLI layer to stay fragmented.

**For practitioners:** if you are building *on top of* agent CLIs, bet on neutrality unless you have specific reasons to do otherwise. The adapter cost is worth it.

### Theme 6 — Self-evolution is cross-cutting, not a single sub-field

Self-improvement / self-evolution appears in multiple forms across the ten:

| Artifact | Self-evolution mechanism |
|---|---|
| Atomic Skills (68) | Joint RL across skills generalizes; skills compound improvements |
| Agent-World (69) | Environment and agent co-evolve (closed loop) |
| Hermes (55) | Session-level self-improvement on user feedback |
| Multica (73) | Team-level skill compounding |
| claude-mem (72) | Endless Mode (biomimetic memory consolidation) |
| gstack (75) | `/learn` skill compounds cross-session |
| Kronos (74) | Fine-tune on your own data to adapt |
| VoltAgent ledger (70) | Many papers on self-evolving agents across buckets |

**The unification:** self-evolution is not a sub-specialty; it is the *dominant meta-theme* of 2026 agent work. Every layer of the stack — training-time (Agent-World), inference-time (Hermes), retrieval-time (claude-mem), team-time (Multica), deploy-time (gstack) — is adopting self-evolution patterns.

The implementations vary:
- *Gradient-based* self-evolution (Atomic Skills joint RL, Agent-World continuous training).
- *Non-gradient memory consolidation* (claude-mem, Hermes, gstack `/learn`).
- *Team-level accumulation* (Multica skills, everything-claude-code community curation).
- *Data-driven fine-tuning* (Kronos on user data).

**Implication:** if your agent product does not include a self-evolution mechanism, you will be outpaced by products that do. The question is no longer *whether* to include self-evolution but *which layer* to include it at.

### Theme 7 — Token economics and progressive disclosure are central

Three of the ten artifacts explicitly surface token economics:

- **claude-mem (72):** ~10× token savings via 3-layer progressive-disclosure retrieval.
- **Atomic Skills (68):** 18.7% average performance gain by composing 5 primitive skills — cost-efficient compared to training monolithic policies.
- **gstack (75):** Smart model routing — Sonnet for fast actions, Opus for analysis. `gstack-model-benchmark` for per-workload cost comparison.

The remaining seven imply token economics without foregrounding it:
- Multica's per-agent token cost becomes a team-level concern.
- Hermes presumably has token-efficient retrieval internally.
- Agent-World's massive 19,822-tool environment requires token-efficient invocation.
- Kronos's 24.7M-param small model is cost-efficient by design.

**The pattern:** the field has internalized that **token cost is the dominant bottleneck at scale**. Every architecture that makes memory / retrieval / reasoning / tool-use cheaper per-token wins long-term. Progressive disclosure is the single most-replicated cost-reduction pattern.

This is the 2026 analog of what *data efficiency* was to the 2020 LLM era — the optimization axis that differentiates winners from losers.

### Theme 8 — Community curation and meta-curation as first-class infrastructure

Three of the ten are curation / meta-curation artifacts:

- **VoltAgent/awesome-ai-agent-papers (70):** 363+ papers, 5 buckets. Research-paper curation.
- **everything-claude-code (62):** Dozens of skills/agents/commands. Community-artifact curation.
- **gstack (75):** 23 specialists from Garry Tan. Personal-artifact curation.

The remaining seven *are* the curated artifacts — they get curated into the above.

**The pattern:** as the number of agent-related artifacts exceeds any individual's reading bandwidth, **curation becomes infrastructure, not a side project**. The curators are *as important as* the artifact authors. VoltAgent's list, everything-claude-code's bundle, and gstack's specialist list all function as **reliability filters** — users trust the curator's selection and spend their attention downstream.

For any 2026+ harness-engineering project, the question of "who will curate us" is as important as "who will use us." Projects get adopted through curator visibility.

### Theme 9 — The multi-surface delivery pattern

Documented in [71](71-karpathy-skills-single-file-guardrails.md), demonstrated across the corpus:

- **karpathy-skills:** CLAUDE.md + Claude Code plugin + Cursor rule.
- **claude-mem:** `npx` installer for Claude Code / Gemini CLI / OpenCode / OpenClaw.
- **Multica:** Brew / install.sh / PowerShell / Docker-compose.
- **gstack:** `./setup --host <agent>` for 10 agents.
- **Kronos:** pip package + HuggingFace Hub + Qlib fine-tuning pipeline.

**The pattern:** one codebase, multiple entry points, user picks the appropriate one. The cost is a dispatch layer in the installer. The benefit is N× adoption reach.

For harness engineers, this is now a default requirement. A 2026 open-source tool that ships only via one path (e.g., "pip install" only, or "git clone" only) will underperform tools that ship 3-5 entry points.

### Theme 10 — Operational realism and production disclaimers

Several artifacts surface the research-production gap explicitly:

- **Kronos:** *"This pipeline is intended as a demonstration. Not a production-ready quantitative trading system."* Clear scoping.
- **Multica:** Explicit comparison with Paperclip, explicit mention of lightweight vs heavy governance tradeoffs.
- **karpathy-skills:** *"These guidelines bias toward caution over speed. For trivial tasks, use judgment."*
- **gstack:** Full LOC-controversy methodology doc with caveats. AI-generated-comments disclosure.
- **claude-mem:** Explicit notes about beta features and stability tradeoffs.

**The pattern:** the field is maturing into *honesty about its limits*. Overclaiming ("this replaces engineers!") is being replaced by scoped honesty ("this is a demo, this has tradeoffs, here are the caveats"). Readers and buyers trust projects that disclose limits more than projects that oversell capabilities.

For harness engineers, this is a practice to adopt: **publish the caveats, the methodology, the reproduction scripts, the failure modes**. Trust is built through honesty about limits, not through marketing bravado.

## Three emergent stack patterns

Synthesizing the themes above into the **operational stack patterns** the ten artifacts collectively reveal:

### Stack Pattern A — The Solo-Developer Super-Productive Stack

Target: one developer shipping like a team of 20.

```
Layer                             Artifact(s)              Scale
──────────────────────────────────────────────────────────────────────────
Agent CLI                         Claude Code              1 primary +
                                                             fallbacks
Behavioral contract               karpathy-skills (71)     60 lines
Specialist workflow               gstack (75)              23 skills
Memory                            claude-mem (72)          Local daemon
Browser / web                     gstack browser           Custom Chromium
Research awareness                VoltAgent ledger (70)    Weekly skim
Specialist tools (by domain)      Kronos (74) or analog    1 tool per
                                                             domain need
Self-evolution                    gstack /learn +          Lightweight
                                    claude-mem Endless
Parallel sprint orchestration     Conductor                10-15 concurrent
```

**Adoption path:**
1. Install Claude Code.
2. Add karpathy-skills CLAUDE.md (30 seconds).
3. Install gstack (30 seconds) + run `/office-hours` / `/review` / `/qa`.
4. Install claude-mem for cross-session memory.
5. Weekly: skim VoltAgent ledger, pick 3-5 papers to read.
6. When a domain task comes up, wrap specialist FMs (Kronos for finance, SAM for vision, etc.) as MCP tools.
7. Scale to 10-15 parallel sprints via Conductor.

**Result:** Garry Tan's 810× productivity claim becomes reproducible (modulo individual skill).

### Stack Pattern B — The Team / Organization Stack

Target: 5-50 human + 10-100 agent team shipping together.

```
Layer                             Artifact(s)
────────────────────────────────────────────────────────────
Work tracking                     Multica (73) — agents as teammates
Agent CLIs                        8+ supported by Multica
Per-developer stack               Solo-Developer stack (above)
Team-shared skills                Multica skill library (pgvector)
Team-shared memory                Either claude-mem per-dev or
                                    Multica skills
Research curation                 VoltAgent ledger for team
Governance                        Multica runtime isolation
                                    + gstack /careful/freeze/guard
Self-hosting option               Multica docker-compose
```

**Adoption path:**
1. Pick team's primary agent CLI(s) (typically Claude Code + 1-2 others).
2. Deploy Multica (cloud or self-hosted).
3. Onboard team members with gstack + karpathy-skills on their machines.
4. Agree on issue-assignment conventions for agents vs humans.
5. Shared skill library grows organically from agent sessions.
6. Weekly team review of VoltAgent ledger for emerging patterns.

**Result:** agent-assisted team at productivity levels competitive with much larger traditional teams.

### Stack Pattern C — The Research / Specialist-Stack

Target: research org or specialist team building new agent capabilities.

```
Layer                             Artifact(s)
────────────────────────────────────────────────────────────
Primitive skills training         Atomic Skills pattern (68)
Environment scaling               Agent-World pattern (69)
Domain foundation model           Kronos-style (74) + fine-tune
Self-improvement                  Hermes-style (55) feedback loop
Tool/MCP exposure                 Wrap specialists as MCP tools
Research awareness                VoltAgent ledger (70)
Hosted product (optional)         Hermes Agent's hosting model
Community artifacts               karpathy-skills, gstack, claude-mem
                                    as precedent for community adoption
```

**Adoption path:**
1. Identify the specialist capability (e.g., financial forecasting, biomedical).
2. Build or adopt a domain FM (Kronos-style).
3. Train atomic skills relevant to the domain (Atomic Skills paper's methodology).
4. Scale environments for training (Agent-World approach).
5. Wrap as MCP tools.
6. Build self-improvement feedback loop (Hermes pattern).
7. Publish community-adoption artifacts (bundle, docs, example integrations).
8. Distribute via HuggingFace Hub + GitHub + multi-IDE integrations.

**Result:** specialist-agent capability that becomes callable infrastructure for the broader agent ecosystem.

## Two unresolved open questions

### Q1 — When does "one developer + ten agents" beat "traditional team"?

gstack (75) claims 810× productivity on logical-lines-of-code for a specific individual (Garry Tan) on specific projects. But this is *one data point*. The field needs:

- **Reproduction across developers.** Can a median developer achieve 50×, 100×, 500×? Or is the ceiling determined by factors beyond tooling (taste, domain expertise, managerial skill)?
- **Reproduction across domains.** Does the productivity gain apply equally to infra / backend / frontend / ML / data-engineering / security? Or does it concentrate in specific domains?
- **Reproduction across scales.** Is the gain larger or smaller on 10-person projects vs 100-person projects vs 1000-person projects?
- **Long-term reliability.** Does 810× productivity on months-to-year timescales hold, or do the AI-authored codebases accumulate debt that degrades productivity over time?

None of the ten artifacts answers these questions. They collectively demonstrate *feasibility*; they do not yet demonstrate *generalizability*. Harness engineers in 2026 should expect a wave of studies attempting exactly these reproductions — positive and negative — over the next 12-24 months.

### Q2 — What is the right granularity for skills — 5 atomic, 23 specialist, 100 community, or more?

The five-to-hundred range spans the skill-granularity debate:

- **5 atomic** (Atomic Skills paper, 68): primitive, composable, jointly-RL-trained, each skill generalizable.
- **23 specialist** (gstack, 75): professional roles with methodology, gated in a sprint process.
- **30-100 community** (everything-claude-code, 62): per-use-case skills curated from the community.
- **1000+ team library** (Multica, 73): organically accumulated across a team's work.
- **100,000+ marketplaces** (VoltAgent ledger references "42,447 agent skills" / "98,000 skills" in the security bucket): unvetted community catalogs.

Each scale has different tradeoffs:
- **5 atomic**: maximum generalization, minimum configuration burden, but may underfit complex workflows.
- **23 specialist**: clear roles, strong workflow enforcement, but high cognitive load for users.
- **30-100 community**: discovery-friendly, customizable, but quality varies.
- **1000+ team**: captures organizational knowledge, but retrieval quality becomes critical.
- **100,000+ marketplace**: comprehensive coverage, but adversarial risk (malicious skills, as the VoltAgent Security bucket documents).

**There is no consensus on the right granularity.** Different users, teams, and domains probably benefit from different answers. The future stack likely combines:
- **Few atomic** skills for generalizable primitives.
- **Dozens of specialist** skills for workflow enforcement.
- **Hundreds of team-library** skills for organizational knowledge.
- **Millions of marketplace** skills for optional discovery.

The *composition* of these layers — how to decide when to use atomic vs specialist vs team vs marketplace — is an open design question. This is arguably the most important harness-engineering design debate of 2026.

## Recommended reading order — from ten links to a mental model

For a new harness engineer approaching this corpus, the efficient reading order is:

**Foundation (start here):**
1. [71-karpathy-skills-single-file-guardrails.md](71-karpathy-skills-single-file-guardrails.md) — The four principles that underlie everything else. Understand the failure modes first.
2. [46-components-of-coding-agent.md](46-components-of-coding-agent.md) — Raschka's six-component model that organizes the rest of the landscape.
3. [66-meta-harness-landscape.md](66-meta-harness-landscape.md) — The meta-harness concept that connects primitive layers.

**Core patterns:**
4. [68-atomic-skills-scaling-coding-agents.md](68-atomic-skills-scaling-coding-agents.md) — Atomic skills as the research-grade abstraction.
5. [72-claude-mem-persistent-memory-compression.md](72-claude-mem-persistent-memory-compression.md) — Persistent memory with progressive disclosure.
6. [75-gstack-garry-tan-claude-code-setup.md](75-gstack-garry-tan-claude-code-setup.md) — Specialist skills organized as a sprint.

**Scaling up:**
7. [69-agent-world-self-evolving-training-arena.md](69-agent-world-self-evolving-training-arena.md) — Environment scaling and self-evolution.
8. [73-multica-managed-agents-platform.md](73-multica-managed-agents-platform.md) — Team-scale orchestration.

**Infrastructure and context:**
9. [70-voltagent-awesome-ai-agent-papers.md](70-voltagent-awesome-ai-agent-papers.md) — Research-landscape ledger.
10. [55-hermes-agent-self-improving.md](55-hermes-agent-self-improving.md) — Hosted self-improving-agent reference.
11. [62-everything-claude-code.md](62-everything-claude-code.md) — Community artifact ecosystem.

**Specialist tools:**
12. [74-kronos-foundation-model-financial-markets.md](74-kronos-foundation-model-financial-markets.md) — Domain FM pattern transferable to other domains.

**Synthesis and reflection:**
13. This file (76) — Cross-cutting themes and stack patterns.
14. [67-recommended-breakthrough-project.md](67-recommended-breakthrough-project.md) — Open research direction: Gnomon / primitive-level attribution.

## Closing thesis — what April 2026 looks like from this vantage

Harness engineering in April 2026 has three simultaneous characteristics:

1. **Rapid maturation.** What was experimental 12 months ago (multi-agent systems, persistent memory, cross-vendor coordination) is now production infrastructure with tens of thousands of stars and millions of users.

2. **Cross-cutting themes are stable.** Skills, memory, multi-host distribution, self-evolution, progressive disclosure, curation. These are the axes the field is organizing around. New work will be measured against these axes.

3. **Unresolved debates define the frontier.** How general vs specific should skills be? When does solo + agents beat teams? Which layer is best for self-evolution? These are the open questions; they will define 2026-2027 research and product development.

The ten artifacts collectively represent **approximately 70-80% of the harness-engineering ground worth understanding as of April 2026**. A harness engineer who understands the ten, the corpus files indexing them, and the six-component mental model from [46-components-of-coding-agent.md](46-components-of-coding-agent.md) has — at minimum — a working mental model of the 2026 state of the art. The Gnomon proposal ([67-recommended-breakthrough-project.md](67-recommended-breakthrough-project.md)) sketches what the next meaningful contribution to the field would look like, given this vantage.

The remaining 20-30% is in niches not represented in this ten-link selection:
- **Enterprise governance and compliance.** Large-org agent deployment has more constraints than these ten artifacts suggest.
- **Non-text modalities.** Vision, audio, robotics agents are separate trajectories that occasionally intersect.
- **Cross-agent protocols (A2A).** VoltAgent ledger references A2A protocols but a canonical A2A implementation has not emerged.
- **Full cost accounting across the stack.** Token economics is covered but end-to-end cost (LLM + infra + human review time + debt accumulation) is not yet systematized.

These are the gaps where the next ten-link synthesis in 2027 or 2028 will live.

## References and cross-links

See the references section in each individual file ([55](55-hermes-agent-self-improving.md), [62](62-everything-claude-code.md), [68](68-atomic-skills-scaling-coding-agents.md)-[75](75-gstack-garry-tan-claude-code-setup.md)) for primary-artifact URLs, author information, and associated documentation.

## Cross-references to other synthesis docs in this corpus

- [46-components-of-coding-agent.md](46-components-of-coding-agent.md) — six-component mental model underlying this synthesis.
- [47-adaptation-of-agentic-ai-survey.md](47-adaptation-of-agentic-ai-survey.md) — adaptation-paradigm synthesis.
- [56-sea-landscape-2026.md](56-sea-landscape-2026.md) — SEA-specific synthesis.
- [60-sea-top-github-repos.md](60-sea-top-github-repos.md) — SEA community synthesis.
- [65-karpathy-new-programming.md](65-karpathy-new-programming.md) — Karpathy's broader thesis that motivates this landscape.
- [66-meta-harness-landscape.md](66-meta-harness-landscape.md) — meta-harness synthesis this builds on.
- [67-recommended-breakthrough-project.md](67-recommended-breakthrough-project.md) — the next-step research direction.
