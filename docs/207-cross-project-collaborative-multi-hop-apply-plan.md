# 207 — Cross-Project Apply Plan: Multi-Hop + Collaborative-AI Techniques into Polaris, Lyra, Argus, and the Other In-Tree Projects

> **Disambiguation.** This file is the **cross-project apply plan** for the techniques catalogued in two recent blocks: multi-hop reasoning ([198-multi-hop-qa-datasets-canon](198-multi-hop-qa-datasets-canon.md) → [203-polaris-multi-hop-reasoning-apply-plan](203-polaris-multi-hop-reasoning-apply-plan.md)) and collaborative-AI ([204-metagpt-foundationagents-2026-refresh](204-metagpt-foundationagents-2026-refresh.md) → [206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md)). Where [203](203-polaris-multi-hop-reasoning-apply-plan.md) was Polaris-only and multi-hop-only, this file extends to **all eleven in-tree projects** (Polaris, Lyra, argus, aegis-ops, atlas-research, harmony-voice, helix-bio, cipher-sec, mentat-learn, orion-code, syndicate, quanta-proof, vertex-eval, gnomon, open-fang) and covers **both** technique families.

## One-line definition

A staged adoption matrix that maps the multi-hop techniques (HippoRAG, DSPy multi-hop, Search-R1 RL, BELLE routing, Plan-on-Graph) and the collaborative-AI techniques (ICPEA typed memory, MCP-first marketplace with trust gate, Lobe-style branching + Pages, Voyager-line skill auto-creation, async memory extractor, per-user constitutions) onto each in-tree project's existing module structure — picking the right techniques per project based on its niche (deep research, research synthesis, skill routing, ops, voice, biomed, security, learning, coding, multi-agent platform, math, eval, harness-IR, fang).

## Why this cross-project plan matters

The in-tree projects ([projects/README.md](../projects/README.md)) cover a deliberately broad set of niches — coding, research, ops, voice, biomed, security, personal learning, math, eval, harness-aware evaluation, and more. The multi-hop and collaborative-AI techniques discussed in 198–206 don't apply uniformly to all of them. A blanket "adopt LobeHub everywhere" plan would over-invest in consumer chat for projects (Aegis-Ops, Quanta-Proof) where the user is a system operator running a CLI, not a chat user. A blanket "train Search-R1 everywhere" plan would over-invest in retrieval RL for projects (Harmony-Voice, Vertex-Eval) where retrieval isn't the binding constraint.

The right plan is **per-project**: for each system, pick the 3–6 techniques that match its niche and stage them across Tier 0 (days 1-14), Tier 1 (days 15-45), Tier 2 (days 46-90). The matrix below is the result.

Take this plan seriously and three things change. (1) Each project gets a *concrete, staged* adoption sequence with module paths and tier slots. (2) Cross-project dependencies become visible — argus's MCP marketplace integration unblocks Polaris's federated tool surface; mentat-learn's skill auto-creation unblocks orion-code's skill library; gnomon's harness-IR unblocks shared evaluation across all projects. (3) The collaborative-AI canon becomes operational rather than aspirational — every project has a clear answer to "should I adopt ICPEA / branching / MCP marketplace / Voyager skills, and when."

## Problem this plan addresses

- For each in-tree project, which multi-hop and collaborative-AI techniques are highest-payoff?
- How do those techniques compose with each project's existing distinctive surface (Polaris's gates, argus's trust verdicts, syndicate's multi-agent orchestration)?
- What are the cross-project dependencies — where does adopting a technique in project X unblock or amplify adoption in project Y?
- What sequencing minimises risk and maximises shared infrastructure (e.g., do once in `harness_core`, reuse across projects)?

## §1 — The eleven in-tree projects (refresher)

From [projects/README.md](../projects/README.md):

| System | Niche | Key baselines |
|---|---|---|
| [Orion-Code](../projects/orion-code/architecture.md) | Autonomous coding | Claude Code, Devin, Cursor, SWE-agent |
| [Atlas-Research](../projects/atlas-research/architecture.md) | Long-horizon research | OpenAI Deep Research, Perplexity Pro, Gemini Deep Research |
| [Aegis-Ops](../projects/aegis-ops/architecture.md) | Production SRE / ops | Bedrock Agents, runbook automation |
| [Harmony-Voice](../projects/harmony-voice/architecture.md) | Real-time voice + RAG | OpenAI Realtime, Retell, Vapi |
| [Syndicate](../projects/syndicate/architecture.md) | Multi-agent platform | AutoGen, CrewAI, MetaGPT, deepagents |
| [Helix-Bio](../projects/helix-bio/architecture.md) | Biomedical research | GPT-Rosalind, RadAgent |
| [Cipher-Sec](../projects/cipher-sec/architecture.md) | Defensive + offensive sec | LinuxArena, Agents-of-Chaos |
| [Mentat-Learn](../projects/mentat-learn/architecture.md) | Self-improving personal assistant | OpenClaw, Hermes, SemaClaw |
| [Quanta-Proof](../projects/quanta-proof/architecture.md) | Math / formal proof | AlphaProof, DeepSeek-Prover |
| [Vertex-Eval](../projects/vertex-eval/architecture.md) | Third-party eval platform | LangSmith, Langfuse, Claw-Eval |
| [Gnomon](../projects/gnomon/docs/architecture.md) | Harness-aware evaluator + closed evolution loop | LangSmith / Hermes / Voyager |
| [Polaris](../projects/polaris/) | Deep research with discipline | Tongyi DeepResearch, OpenAI Deep Research |
| [Lyra](../projects/lyra/) | Paper-grounded research synthesis | n/a (in-tree research surface) |
| [argus](../projects/argus/) | Skill router agent | Claude Skills, MCP Marketplaces |
| [open-fang](../projects/open-fang/) | n/a (placeholder / pre-spec) | n/a |

## §2 — The cross-project adoption matrix

For each project, the rows show which multi-hop or collaborative-AI techniques are recommended, mapped to a tier (T0 = days 1-14, T1 = days 15-45, T2 = days 46-90). Empty cells mean "not recommended" (off-niche).

| Technique | Polaris | Lyra | argus | Atlas | Aegis | Harmony | Helix | Cipher | Mentat | Orion | Synd | QPr | Vert | Gnom |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **Multi-hop substrate** | | | | | | | | | | | | | | |
| HippoRAG single-shot | T0 | T0 | — | T0 | — | T1 | T0 | T1 | — | — | — | — | — | — |
| LightRAG incremental ingest | T0 | T0 | — | T0 | — | — | T0 | — | — | — | — | — | — | — |
| BELLE-style question router | T0 | T0 | — | T0 | — | T1 | T0 | T1 | T1 | — | T1 | — | — | — |
| Chain-of-Note quality gate | T0 | T0 | — | T0 | T0 | T1 | T0 | T0 | — | — | — | — | — | — |
| Self-Ask / IRCoT externalised chain | T0 | T0 | — | T0 | — | — | T0 | T1 | — | — | — | — | — | — |
| Decomposition cache | T0 | T0 | — | T0 | — | T1 | T0 | — | — | — | — | — | — | — |
| MuSiQue+FRAMES+Seal-0 nightly | T0 | T0 | — | T0 | — | — | T0 | — | — | — | — | — | T0 | T0 |
| DSPy multi-hop program | T1 | T1 | — | T1 | — | T1 | T1 | T2 | — | T1 | — | — | — | — |
| Plan-on-Graph backtracking | T1 | T1 | — | T1 | — | — | T1 | T2 | — | — | — | — | — | — |
| AnchorRAG entity linking | T1 | T1 | — | T1 | — | — | T1 | — | — | — | — | — | — | — |
| Beam Retrieval ≥3 hop | T1 | T1 | — | T1 | — | — | T1 | — | — | — | — | — | — | — |
| Search-o1 Reason-in-Documents | T1 | T1 | — | T1 | — | — | T1 | T2 | — | — | — | — | — | — |
| Search-R1 RL recipe | T2 | T2 | — | T2 | — | — | T2 | T2 | — | — | — | — | — | — |
| **Collaborative-AI substrate** | | | | | | | | | | | | | | |
| ICPEA five-layer memory | T0 | T0 | T1 | T0 | — | T0 | T1 | — | T0 | T1 | T1 | — | — | — |
| Async memory extractor | T0 | T0 | T1 | T0 | — | T1 | T1 | — | T0 | T1 | T1 | — | — | — |
| Per-agent layer access control | T1 | T1 | T1 | T1 | T0 | T1 | T1 | T0 | T1 | T1 | T1 | — | — | T1 |
| MCP-first marketplace + Argus trust gate | T1 | T1 | T0 | T1 | T1 | — | T1 | T0 | T1 | T1 | T1 | — | T1 | T1 |
| Branching conversations | T1 | T1 | T2 | T1 | — | — | T1 | T1 | T1 | T1 | T1 | T1 | — | — |
| Lobe Pages-style co-author | T1 | T1 | — | T1 | T1 | — | T1 | — | — | T1 | T1 | T1 | T1 | — |
| Voyager-line skill auto-creation | T2 | T2 | T0 | T2 | T1 | — | T2 | T1 | T0 | T1 | T2 | — | — | T2 |
| Per-user constitution (ICAI/C3AI) | T2 | T2 | T2 | T2 | T1 | — | T2 | T2 | T1 | T2 | T2 | — | — | — |
| Local-first / IndexedDB mode | T2 | T1 | — | T2 | — | T1 | T1 | T1 | T0 | — | — | — | — | — |
| Voice + screen-share | — | T2 | T2 | T2 | — | T0 | T1 | — | T1 | — | T2 | — | — | — |
| Pure-function agents | T0 | T1 | T1 | T0 | T0 | T1 | T0 | T0 | T0 | T0 | T0 | T0 | T0 | T0 |
| Equal-budget benchmark harness | T0 | T0 | — | T0 | T0 | T1 | T0 | T0 | — | T0 | T0 | T0 | T0 | T0 |
| Test-time-compute curve | T0 | T0 | — | T0 | T0 | T0 | T0 | T0 | — | T0 | T0 | T1 | T0 | T0 |

The "—" cells aren't omissions — they're explicit "not recommended for this niche." Cipher-Sec doesn't get user-facing branching UX; Quanta-Proof doesn't get HippoRAG (formal proofs aren't multi-hop QA); Aegis-Ops doesn't get ICPEA personal memory (the user is a system, not a person).

## §3 — Per-project briefs

Below: 3–6 sentence briefs per project explaining the rationale for the recommended techniques and how they fit the project's distinctive niche.

### Polaris — deep-research with discipline

Already covered exhaustively in [203-polaris-multi-hop-reasoning-apply-plan](203-polaris-multi-hop-reasoning-apply-plan.md). The collaborative-AI delta on top: **ICPEA personal-memory layer** (T0) gives Polaris a per-user lens (researcher's preferred citation style, T1-source domains, language) that survives across projects without polluting the project graph. **MCP-first marketplace with Argus trust gate** (T1) wires external tools as first-class T-tiered sources. **Branching as research-tree UX** (T1) surfaces the daemon's exploration to the user. **Lobe Pages co-authored writeup** (T1) replaces flat-markdown output with a Notion-style timeline. **Voice + screen-share** is T2 (low priority for a discipline-first deep-research agent). **Voyager-line skill auto-creation** (T2) applies argus and the existing `polaris-skills/auto_creator`.

### Lyra — paper-grounded research synthesis

Lyra's existing strength is the curated paper deep-dive corpus (files 77–99 plus the broader knowledge base). The right additions are: **HippoRAG over the deep-dive corpus** (T0) — query-time multi-hop retrieval across a 200+-doc collection wins big. **BELLE-style routing** (T0) — different deep-dive types (theory vs application vs survey) get different reading methods. **ICPEA + Lobe Pages** (T0/T1) — turn each deep-dive from a static artifact into a co-authored teammate that knows the user's research history. **Voice + TTS via `@lobehub/tts`** (T2) — every deep-dive becomes a podcast at zero authoring cost. **MuSiQue+FRAMES+Seal-0 nightly** (T0) measures whether the deep-dive corpus is being read multi-hop-faithfully.

### argus — skill-router agent

argus's role is **discovery / routing / curation / refinement / governance** for skills (see [180-argus-skill-router-agent-design](180-argus-skill-router-agent-design.md), [194-197]). The right additions invert the usual hierarchy: argus is the *provider* of trust-gating for everyone else, not the consumer. **MCP-first marketplace integration** (T0) — argus already indexes 22k+ Claude Code skills and 25k+ MCP servers; LobeHub's curated 169,739-skill index + permission-negotiation-at-install gives argus a higher-trust starting catalog. **Voyager-line skill auto-creation** (T0) is core to argus's mission. **ICPEA Preference + Experience layers** (T1) make argus's router *user-aware* (this user prefers code-emitting skills over UI skills). **Branching as alternative-skill-loadouts UX** (T2) when argus is uncertain between two activations. argus does not need its own multi-hop substrate — its role is to route to skills that *have* multi-hop substrates.

### Atlas-Research — long-horizon research agent

Atlas's niche overlaps Polaris's, but Atlas targets **broader / less-disciplined** deep research. The plan mirrors Polaris's but with relaxed gating. **HippoRAG + LightRAG + BELLE router + Self-Ask/IRCoT + Chain-of-Note + decomposition cache** all T0. **DSPy multi-hop + Plan-on-Graph + AnchorRAG + Beam Retrieval + Reason-in-Documents** all T1. **Search-R1 RL** at T2. Plus the ICPEA + Pages + branching collaborative surface to feel like a teammate. The differentiator vs Polaris is that Atlas does *not* enforce twenty bright-line gates — it ships a smaller gate surface and prioritises throughput.

### Aegis-Ops — production SRE / ops

The user here is *a system operator running a CLI*, not a chat user. Personal memory, branching UX, voice, and Lobe Pages are off-niche. The right additions are: **Per-agent layer access control** (T0) on the runbook context (different agents see different runbooks), **MCP-first marketplace** (T1) for ops tools (k8s, AWS, GCP MCP servers), **Voyager-line skill auto-creation** (T1) for runbook-as-skill promotion when an incident response succeeds, **per-user constitution** (T1) — the *operator*'s constitution (be conservative, alert before mutating, prefer dry-run) is a serialisable preference object. **Pure-function agents** (T0) is non-negotiable for replayable incident response. **Equal-budget + TTC curve** (T0) for observability.

### Harmony-Voice — real-time voice + RAG

The voice surface ([137-voice-agents](137-voice-agents.md), `@lobehub/tts` reference) is *the* niche. **Voice + screen-share** at T0. **HippoRAG / BELLE / Chain-of-Note / decomposition cache** at T1 for the RAG side once latency budgets allow. **ICPEA personal memory** (T0) — voice agents lose the chat-history visual cue, so persistent memory matters more, not less. **Pure-function agents** (T1). The full multi-hop substrate (Plan-on-Graph, AnchorRAG, Search-R1) is off-niche — voice agents don't have time for deep reasoning. The right shape is fast retrieval + light reasoning + persistent memory.

### Helix-Bio — biomedical research

Biomed deep research overlaps Atlas + Polaris but adds **dual-use safety as first-class**, **structured KGs (UniProt, PDB, AlphaFold)**, and **regulatory provenance**. The full multi-hop substrate at T0. **ICPEA personal memory** (T1) — the researcher's preferred organism, assay, and chemistry style. **MCP-first marketplace** (T1) for biomed tools. **Voice + screen-share** (T1) for "show the agent this gel image / cryo-EM map." **Voyager-line skill auto-creation** (T2) for protocol-as-skill promotion. The argus trust gate is *especially* important here (papers in biomed get retracted; cf. [172-polaris-2026-deep-research-roadmap](172-polaris-2026-deep-research-roadmap.md) §3 Gap 5 + JMIR retracted-citation findings).

### Cipher-Sec — defensive + offensive security

Security agents have a different threat model: the *agent itself* is potentially adversarial, and the user is a security professional whose budget for false positives is low. **HippoRAG / BELLE** at T1 for CVE / threat-intel multi-hop. **Per-agent layer access control** (T0) on the security context — different agents see different scoped credentials. **MCP-first marketplace with argus trust gate** (T0) — supply-chain attacks via MCP servers are real. **Voyager-line skill auto-creation** (T1) for offensive playbooks (with HITL gating). **Pure-function agents** (T0). **Per-user constitution** (T2) — the operator's offensive scope authorisation is a serialisable contract. No personal-memory pollution allowed; ICPEA is *off* for cipher-sec.

### Mentat-Learn — self-improving personal assistant

Mentat's whole point is the personalisation axis. **ICPEA personal memory** (T0) is the centre of gravity. **Async extractor** (T0). **Voyager-line skill auto-creation** (T0) — every successful task spawns a candidate skill, gated by the user's edit pattern. **Local-first / IndexedDB mode** (T0) — the user's daily assistant is the highest-leverage privacy target. **Per-user constitution** (T1). **Branching conversations + voice + screen-share** (T1) for "what-if" exploration. **MCP marketplace** (T1) for tool discovery. The multi-hop substrate is mostly off-niche — mentat-learn is short-horizon assistant tasks, not deep research.

### Orion-Code — autonomous coding

Coding agents have their own substrate (filesystem, sandbox, test runner). The right additions are: **DSPy multi-hop program** (T1) for code-search across large codebases. **ICPEA + Lobe Pages** (T1) — the user's coding style, preferred libraries, and project conventions are personal-memory layers; Pages becomes the architecture-doc surface. **MCP-first marketplace + argus trust gate** (T1) for coding tools (linters, formatters, security scanners). **Voyager-line skill auto-creation** (T1) for project-specific patterns (the cliché example: "the user always wants tests written in pytest"). **Branching as alternative-implementation UX** (T1). **Pure-function agents** (T0). **Per-user constitution** (T2).

### Syndicate — multi-agent product platform

Syndicate's niche is *the multi-agent platform itself*. The right additions are mostly the multi-agent collaboration substrate from [206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md) §2. **GroupChat / role-catalog / supervisor / graph-as-node / orchestrator-ledger / handoffs / code-as-action** all available as composable patterns. **ICPEA + per-agent layer access control** (T1) for shared memory across agents in a session. **MCP marketplace** (T1) for the agent's tool surface. **Lobe Pages + branching** (T1) as the user-facing collaboration UX. **Per-user constitution** (T2). The multi-hop substrate is at T1 — syndicate's agents will need it for multi-hop subtasks but it's not the centre of gravity. **Equal-budget benchmarks** (T0) is critical because syndicate is the project most exposed to the [202-multi-agent-multi-hop-reckoning-2026](202-multi-agent-multi-hop-reckoning-2026.md) critique.

### Quanta-Proof — math / formal proof

Math agents have a *very* different substrate: Lean 4 / Coq / Isabelle / Z3 / sympy / proof certificates. Most of the canon doesn't apply. **Lobe Pages** (T1) as the proof-document surface. **Pure-function agents** (T0). **Branching** (T1) as the proof-strategy-exploration UX. **TTC curve** (T1) — the AlphaProof-class compute regime is extreme. **Equal-budget benchmark** (T0). The multi-hop substrate is *off* — formal-proof multi-hop is graph-walk over proof state, fundamentally different from QA multi-hop.

### Vertex-Eval — third-party eval platform

Vertex-Eval is the project that *hosts* benchmarks for everyone else. **MuSiQue+FRAMES+Seal-0 nightly** (T0) — vertex-eval should ship these as first-class benchmarks. **Equal-budget harness + TTC curve + active-parameter accounting** (T0) — these are the cross-cutting evaluation discipline tools the [202-multi-agent-multi-hop-reckoning-2026](202-multi-agent-multi-hop-reckoning-2026.md) critique demands. **Lobe Pages co-authored** (T1) for the eval report surface. **MCP-first marketplace** (T1) for benchmark tools. **Pure-function agents** (T0). Personal memory and branching are off-niche.

### Gnomon — harness-aware evaluator + closed evolution loop

Gnomon's HIR (Harness IR) sits at the meta-level — it's the substrate that lets *all the other projects* share a primitive-attributed failure-rate signal. **MuSiQue+FRAMES+Seal-0 nightly** (T0) over the HIR. **Per-agent layer access control** (T1) on the HIR's trace bundles. **MCP marketplace** (T1) for harness adapters. **Voyager-line skill auto-creation** (T2) for HIR-pattern promotion. **Pure-function agents + equal-budget + TTC curve** (T0) cross-cuts.

### open-fang — placeholder

Out of scope for this plan; revisit when the project ships an architecture document.

## §4 — Cross-project shared infrastructure (where the leverage is)

Several techniques should ship as **shared library code** in a place all projects can import, not as per-project copies.

### Shared in `harness_core/`

Each project vendors `harness_core/` ([projects/README.md](../projects/README.md) §"Vendored library"). The following modules should be added once and reused:

- **`harness_core.memory.icpea`** — typed five-layer memory schema, async extractor, per-agent layer access control. Adopted by Polaris, Lyra, Atlas, Helix, Mentat, Orion, Syndicate.
- **`harness_core.multi_hop.hipporag`** — HippoRAG-style PPR-weighted KG retrieval. Adopted by Polaris, Lyra, Atlas, Helix.
- **`harness_core.multi_hop.dspy_program`** — DSPy multi-hop program template. Adopted by Polaris, Lyra, Atlas, Helix, Orion, Harmony.
- **`harness_core.routing.bell_router`** — BELLE-style question-type router (without the debate). Adopted by Polaris, Lyra, Atlas, Helix.
- **`harness_core.gates.chain_of_note`** — Chain-of-Note quality gate. Adopted by all multi-hop-using projects.
- **`harness_core.orchestration.pure_function`** — Yenugula's side-effect-free agent base class. Adopted by **all** projects.
- **`harness_core.evals.equal_budget`** — equal-budget benchmark harness. Adopted by all benchmarking projects.
- **`harness_core.evals.ttc_curve`** — test-time-compute curve plotter. Adopted by all benchmarking projects.
- **`harness_core.evals.active_params`** — active-parameter cost accounting. Adopted by all MoE-aware benchmarking.
- **`harness_core.skills.auto_creator`** — Voyager-line skill auto-creation primitives. Adopted by argus, Mentat, Orion, Aegis, Cipher, Polaris, Helix.
- **`harness_core.marketplace.mcp_install`** — one-click MCP install + permission-at-install. Adopted by every project that wants third-party tools.
- **`harness_core.constitution`** — per-user constitution (ICAI/C3AI) data model + injection. Adopted by Mentat, Aegis, Orion, Polaris, Lyra, Atlas, Helix, Syndicate.
- **`harness_core.ux.branching`** — branching trajectory tree primitive. Adopted by Polaris, Lyra, Atlas, Helix, Cipher, Mentat, Orion, Syndicate, Quanta-Proof.
- **`harness_core.ux.pages`** — Lobe Pages-style co-authored document. Adopted by all "research output" projects.

### Shared via argus

argus is the natural home for the **MCP marketplace + trust gate** for every other project. The pattern:

- Every project that wants MCP tools imports `argus.mcp_marketplace` (or whatever the shipped name becomes).
- argus owns the trust verdicts; the calling project gets a typed `TrustedMCPServer` that fails closed if argus's verdict is below threshold.
- argus owns skill auto-creation; calling projects emit successful trajectories to argus's curator endpoint and pull promoted skills back.

### Shared via gnomon

gnomon's HIR is the natural home for the **cross-harness evaluation discipline**:

- Every project that wants benchmarking emits HIR traces via `gnomon.hir_emit`.
- gnomon's HIR engine runs MuSiQue+FRAMES+Seal-0 + equal-budget + TTC curves over the trace bundles.
- The primitive-attributed failure-rate signal (cf. [67-recommended-breakthrough-project](67-recommended-breakthrough-project.md)) becomes the shared SEA reward across all projects.

## §5 — Cross-project dependency graph

Some adoption decisions are blocked by others. The dependency graph (with → reading "X unblocks Y"):

- **harness_core.icpea** → Polaris, Lyra, Atlas, Helix, Mentat, Orion, Syndicate ICPEA adoption.
- **harness_core.hipporag** → Polaris, Lyra, Atlas, Helix multi-hop substrate.
- **harness_core.pure_function** → all projects' replayability.
- **argus.mcp_marketplace + trust_gate** → every project's MCP integration.
- **argus.auto_creator** → every project's skill self-evolution.
- **gnomon.hir_emit** → cross-harness evaluation across all projects.
- **vertex-eval's MuSiQue+FRAMES+Seal-0 + equal-budget + TTC curve** → reusable benchmarks every project pulls from.

The implication: **build the shared infrastructure first**, then per-project adoption is fast.

## §6 — Suggested 90-day cross-project roadmap

A coordinated three-month plan that respects the dependency graph.

### Days 1-14 — Tier 0 across all projects

**Shared infra (week 1).** Build `harness_core.{icpea, hipporag, pure_function, equal_budget, ttc_curve, active_params}`. Smoke-test in one consumer (Polaris).

**Project-level adoption (week 2).** Each project pulls the relevant T0 techniques:

- Polaris — full Tier 0 from [203](203-polaris-multi-hop-reasoning-apply-plan.md) plus ICPEA.
- Lyra — HippoRAG over deep-dive corpus + ICPEA.
- argus — MCP marketplace integration + skill auto-creator.
- Atlas — multi-hop full T0 + ICPEA.
- Aegis — pure-function agents + per-agent layer access on runbook context.
- Harmony — voice + ICPEA (T0).
- Helix — multi-hop full T0 + Chain-of-Note + biomed-specific gates.
- Cipher — pure-function + per-agent layer access + MCP trust gate.
- Mentat — ICPEA + skill auto-creator + IndexedDB mode.
- Orion — pure-function + equal-budget + TTC curve.
- Syndicate — pure-function + equal-budget + TTC curve.
- Vertex-Eval — MuSiQue+FRAMES+Seal-0 nightly + equal-budget + active-params.
- Gnomon — HIR emit + nightly benchmark integration.

### Days 15-45 — Tier 1 across all projects

**Shared infra (week 3-4).** Build `harness_core.{dspy_program, bell_router, chain_of_note, branching, pages, constitution, mcp_install}`. argus's marketplace + trust gate exposed as a stable API.

**Project-level adoption (week 5-6).** Per-project Tier 1 lifts.

### Days 46-90 — Tier 2 across all projects

**Shared infra (week 7-8).** Build `harness_core.skills.auto_creator` (consuming argus). Search-R1 RL training scaffold in `harness_core.rl.search_r1`.

**Project-level adoption (week 9-13).** Per-project Tier 2 lifts including RL training where applicable.

## §7 — Risks and mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Shared infra changes break every project simultaneously | Med | High | SemVer the shared libraries; per-project pinning; nightly cross-project smoke tests in vertex-eval |
| ICPEA memory cross-contaminates between projects | Low-Med | High | Per-project namespace + per-agent layer access control + per-user scope keys |
| MCP marketplace supply-chain attack | Med | High | argus trust gate before install; sandboxed MCP execution; CVE feed integration |
| Search-R1 training reward-hacks each project's gates | Med | High | Per-project reward shaping that mirrors the project's bright-line gates |
| Skill auto-creator over-promotes noise | Med | Med | Surrogate-verifier + held-out eval gate ([169-coevoskills-co-evolutionary-verification](169-coevoskills-co-evolutionary-verification.md)) |
| Branching state inconsistency | Low-Med | Med | Snapshot agent memory state at branch points; document fork semantics |
| Equal-budget enforcement is hard to enforce across heterogeneous architectures | High | Med | Token-counting at the harness boundary; document Gemini-style accounting caveats per [202-multi-agent-multi-hop-reckoning-2026](202-multi-agent-multi-hop-reckoning-2026.md) §3 |
| LobeHub Community License affects derivative-shell distribution | Low (we're not redistributing LobeHub) | Med | Lift patterns, not code; track license updates |
| Per-user constitution drift over time | Med | Low | Versioned constitutions + drift detection at extraction time |
| Cross-project dependency loops (argus depends on gnomon, gnomon depends on argus) | Low | High | Define minimal stable APIs; break circular dependencies via traits / interface modules |

## §8 — Concrete next-steps checklist

A pragmatic next-2-weeks list for whoever owns this plan:

1. **Day 1.** Open a `harness_core/` workstream branch. Land `harness_core.orchestration.pure_function` first — it's the prerequisite for everything else.
2. **Day 2-3.** `harness_core.evals.{equal_budget, ttc_curve, active_params}`. Vertex-Eval pulls these.
3. **Day 4-5.** `harness_core.memory.icpea` skeleton + async extractor. Smoke-test in Mentat.
4. **Day 6-8.** `harness_core.multi_hop.hipporag` over a small KG. Smoke-test in Lyra.
5. **Day 9-10.** argus exposes `mcp_marketplace + trust_gate` as a stable API. Polaris and Aegis subscribe.
6. **Day 11-12.** gnomon exposes `hir_emit` API. Vertex-Eval and Polaris subscribe.
7. **Day 13.** First cross-project smoke: Polaris uses ICPEA + HippoRAG + argus trust gate + gnomon HIR emit.
8. **Day 14.** Tier-0 retro across all projects. Sign-off for Tier 1.

## §9 — Cross-references

- [198-multi-hop-qa-datasets-canon](198-multi-hop-qa-datasets-canon.md), [199-multi-hop-reasoning-techniques-arc](199-multi-hop-reasoning-techniques-arc.md), [200-graph-grounded-multi-hop-retrieval](200-graph-grounded-multi-hop-retrieval.md), [201-compositionality-gap-canon](201-compositionality-gap-canon.md), [202-multi-agent-multi-hop-reckoning-2026](202-multi-agent-multi-hop-reckoning-2026.md), [203-polaris-multi-hop-reasoning-apply-plan](203-polaris-multi-hop-reasoning-apply-plan.md) — multi-hop block.
- [204-metagpt-foundationagents-2026-refresh](204-metagpt-foundationagents-2026-refresh.md), [205-lobehub-collaborative-teammate-platform](205-lobehub-collaborative-teammate-platform.md), [206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md) — collaborative-AI block.
- [172-polaris-2026-deep-research-roadmap](172-polaris-2026-deep-research-roadmap.md) — Polaris v2.x phasing.
- [180-argus-skill-router-agent-design](180-argus-skill-router-agent-design.md), [194-argus-omega-enhanced-design](194-argus-omega-enhanced-design.md), [195-argus-omega-vol-2-trajectory-temporal-horizon](195-argus-omega-vol-2-trajectory-temporal-horizon.md), [196-argus-vs-field-skill-loading-comparison](196-argus-vs-field-skill-loading-comparison.md), [197-argus-omega-vol-3-recursive-skills-curator](197-argus-omega-vol-3-recursive-skills-curator.md) — argus.
- [167-autoskill-experience-driven-lifelong-learning](167-autoskill-experience-driven-lifelong-learning.md), [168-evoskill-coding-agent-skill-discovery](168-evoskill-coding-agent-skill-discovery.md), [169-coevoskills-co-evolutionary-verification](169-coevoskills-co-evolutionary-verification.md), [170-skillrl-recursive-skill-augmented-rl](170-skillrl-recursive-skill-augmented-rl.md), [171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md) — skill self-evolution canon.
- [184-strongest-memory-techniques-synthesis-may-2026](184-strongest-memory-techniques-synthesis-may-2026.md), [185-memory-integration-playbook](185-memory-integration-playbook.md), [187-multi-agent-shared-memory-landscape](187-multi-agent-shared-memory-landscape.md), [188-witness-provenance-memory-techniques-synthesis](188-witness-provenance-memory-techniques-synthesis.md) — memory canon.
- [189-recursive-multi-agent-systems](189-recursive-multi-agent-systems.md), [191-onemancompany-skills-to-talent](191-onemancompany-skills-to-talent.md), [193-recursive-world-organizations-synthesis](193-recursive-world-organizations-synthesis.md) — recursive-agent / talent / organisations.

**The one-line takeaway for harness designers:** Stop building per-project bespoke memory + multi-hop + marketplace + skill substrates — build them once in `harness_core/` (with argus as the marketplace owner and gnomon as the eval owner), and every in-tree project gets the teammate-that-grows-with-you surface for one engineering investment.
