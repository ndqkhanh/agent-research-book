# 70 — VoltAgent / awesome-ai-agent-papers: The 2026 Weekly-Curated Agent-Research Ledger

**Definition.** VoltAgent's *awesome-ai-agent-papers* ([github.com/VoltAgent/awesome-ai-agent-papers](https://github.com/VoltAgent/awesome-ai-agent-papers)) is a hand-curated, weekly-updated GitHub ledger of **363+ AI-agent research papers published on arXiv from January 2026 onward**, organized into five canonical buckets: **Multi-Agent (53), Memory & RAG (57), Eval & Observability (80), Agent Tooling (95), and AI Agent Security (82)**. Its value is not the raw links — arXiv itself provides those — but the *filtering, categorization, and one-sentence synthesis* applied to the hundreds of agent-adjacent papers arriving on arXiv each week. For a field moving too fast to track manually, the list functions as a **consumable weekly delta** of the research frontier. Built by VoltAgent (a TypeScript-AI-framework company) and maintained alongside their [Awesome Codex Subagents](https://github.com/VoltAgent/awesome-codex-subagents) repo, the list is MIT-licensed, has 621 GitHub stars (April 2026), and accepts PRs for new paper submissions.

This file is a meta-source: it sits *above* the individual paper files in this corpus (docs 26–38, 47, 57–59, 68, 69). Where those files deep-dive single papers, this file maps the whole landscape that the ledger surfaces — identifying **the shape of the 2026 agent-research firehose**, the structural patterns recurring across hundreds of papers at once, and the clusters that matter most to harness engineering specifically.

## Problem it solves

Two bottlenecks confront anyone trying to track agent research in 2026:

1. **Volume.** arXiv posts tens of thousands of LLM-adjacent papers per month, of which hundreds touch agents in some way. No individual can read them all. The signal-to-noise ratio drops as volume rises.
2. **Categorization drift.** arXiv's own category system (cs.AI, cs.CL, cs.LG, cs.SE) predates agents as a distinct field. A paper on agent tooling might be filed under cs.SE, cs.AI, or cs.CL depending on author convention. arXiv's browse-by-category experience does not surface the agent-specific structure.

Awesome-ai-agent-papers addresses both bottlenecks with a simple but disciplined workflow: (a) sweep arXiv weekly for agent-relevant submissions, (b) filter by relevance to "the AI agent ecosystem" (the specific phrase in the README), (c) write a 1-2-sentence synthesis focused on *what the paper's contribution is, not its title*, (d) slot it into one of five fixed buckets. The output is something a harness engineer can read in 30 minutes and come away with a current-week landscape snapshot.

This is a **community curation layer** in the same family as [62-everything-claude-code.md](62-everything-claude-code.md) (a community skill/agent bundle) and [60-sea-top-github-repos.md](60-sea-top-github-repos.md) (a curated GitHub SEA survey). The common thread across all three: as the field scales beyond any individual's attention, human curation with a clear taxonomy has higher ROI than any automated summarizer.

## The five buckets — what each covers and why it exists

### 1. Multi-Agent (53 papers)

Papers on **how multiple agents coordinate, communicate, and collectively solve problems**. This bucket is where the 2026 field continues to work out whether multi-agent systems beat, tie, or lose to single agents.

**Representative themes across the 53 papers:**
- *Orchestration topologies.* Static vs dynamic, centralized vs decentralized, peer-to-peer vs hierarchical. Papers like DyTopo (rewires agent-to-agent connections at each reasoning round via semantic matching), CORAL (long-running multi-agent systems with shared persistent memory and heartbeat-based interventions), and TopoDIM (one-shot topology generation) all push on topology as a first-class variable rather than a fixed architecture.
- *Decentralized / cooperative RL.* Multi-agent actor-critic approaches, Multi-Agent Reward Optimization (MARO), Learning to Collaborate (orchestrated decentralized peer-to-peer LLM federation). These papers bring reinforcement-learning multi-agent orthodoxy to LLM-backbone agents.
- *"Does multi-agent even work?"* Several papers directly challenge the multi-agent premise — *Multi-Agent Teams Hold Experts Back* (self-organizing teams can fall below best-member performance), *When Single-Agent with Skills Replace Multi-Agent Systems and When They Fail*. This is the empirical follow-through on the Cognition AI "Don't Build Multi-Agents" thesis ([02-subagent-delegation.md](02-subagent-delegation.md)).
- *Confidence-aware routing and adaptive coordination.* CASTER, Orchestrating Intelligence, JADE — a family of papers that treat "route to the right agent" as the central problem, using semantic embeddings, structural meta-features, or task-adaptive routing.
- *Agent2Agent protocol layer.* Papers like *The Orchestration of Multi-Agent Systems: Architectures, Protocols, and Enterprise Adoption* and *Beyond Rule-Based Workflows: CORAL* formalize the agent-to-agent communication layer as a protocol problem. MCP ([07-model-context-protocol.md](07-model-context-protocol.md)) is the tool-layer counterpart; A2A is the agent-layer counterpart.

**What this bucket tells us about 2026 consensus:** multi-agent research has **bifurcated**. One branch is still proving multi-agent wins (with careful coordination); the other is actively falsifying the premise. A reader should expect no single dominant answer — the truth seems to be task-dependent, and the field has not yet identified which tasks fall which way. The Cognition-AI "context coherence" critique continues to have teeth.

### 2. Memory & RAG (57 papers)

Papers on **what agents remember, how they retrieve it, and how they keep memory from poisoning the context window over time**. This is where most of the 2026 harness-engineering innovation lives because memory is where the naive approaches fail hardest.

**Representative themes:**
- *Biological / cognitive inspiration.* FadeMem (biologically-inspired forgetting with adaptive exponential decay), HiMeS (hippocampus-inspired dual memory), AMA (adaptive memory via multi-agent collaboration with hierarchical granularity), Active Context Compression (Physarum polycephalum-inspired consolidation), AI Hippocampus survey. The field is clearly borrowing from neuroscience — both as inspiration and (in several papers) as direct model structure.
- *Graph-based agent memory.* Graph-based Agent Memory survey, MAGMA (multi-graph architecture across semantic/temporal/causal/entity graphs), Relink (reason-and-construct on-the-fly evidence graphs), Topo-RAG. Graphs are emerging as the dominant structured-memory substrate, replacing pure vector stores.
- *Budget-tier / routing-based memory.* BudgetMem (query-aware budget-tier routing), ShardMemo (masked MoE routing for sharded memory), SwiftMem (query-aware indexing with temporal + semantic DAG-Tag). Cost-awareness is finally showing up in memory design — you cannot afford to retrieve every memory on every query.
- *Procedural memory / skill-style.* ProcMEM (reusable procedural memory via non-parametric PPO), AtomMem (atomic CRUD operations with SFT+RL learned policy), E-mem (episodic context reconstruction with master-assistant agent split). These are the training-time and algorithmic counterparts to [04-skills.md](04-skills.md) and [19-voyager-skill-libraries.md](19-voyager-skill-libraries.md).
- *Forgetting / consolidation.* FadeMem, Continuum Memory Architectures, Amory (narrative-driven), Membox (topic continuity). The "what to forget" problem is receiving as much attention as "what to remember" — likely because naive append-forever memory is the leading failure mode in production deployments.
- *Query-aware context construction.* SPARC-RAG, CIRAG, PRISMA, Corpus2Skill (compile corpus into navigable skill tree instead of retrieving). The RAG pipeline is being rethought at every stage — query planning, retrieval, reranking, evidence pruning, context construction.

**What this bucket tells us:** memory is no longer solved by "vector DB + top-k retrieval." 2026's consensus is **multi-layered, graph-structured, decay-aware, intent-conditioned, budget-allocated**. Each of those adjectives represents an axis where prior art has now matured into named techniques. This is the single most-active subfield in agent research and is likely to remain so.

### 3. Eval & Observability (80 papers)

Papers on **how we decide whether an agent worked**. The largest bucket, consistent with the corpus-wide observation that evaluation is the harness-engineering bottleneck in 2026.

**Representative themes:**
- *Benchmarks for hard real-world settings.* ClawBench (live web tasks, [34](34-clawbench-live-web-tasks.md)), Terminal-Bench (89 hard CLI tasks), DevOps-Gym (700+ DevOps tasks), APEX-Agents (480 long-horizon cross-application productivity tasks by investment bankers/consultants/lawyers), AIRS-Bench (20 real ML research tasks), CooperBench (600+ collaborative coding tasks). The benchmarks are getting *much* more realistic — the "LeetCode for LLMs" era is over.
- *Failure analysis at scale.* Why AI Agent Involved PRs Remain Unmerged (8,106 fix PRs), When Agents Fail (1,187 bug reports across 7 frameworks), Let's Make Every PR Meaningful (40,214 PRs), More Code Less Reuse (code quality / reviewer sentiment on AI PRs). Empirical studies of agent failure in the wild are now routine — hundreds of PRs analyzed, not dozens.
- *Trajectory-aware evaluation.* TrajAD (trajectory anomaly detection), TriCEGAR (trace-driven abstraction with CEGAR), LUMINA (oracle counterfactual framework for multi-turn tasks), Claw-Eval ([38](38-claw-eval.md)), Holistic Trajectory Calibration. The shift from end-state to trajectory evaluation is now the dominant methodology.
- *Process-aware / architecture-aware metrics.* Toward Architecture-Aware Evaluation Metrics, M3-BENCH (process-aware for mixed-motive games), AEMA (process-aware multi-agent evaluation), Tokenomics (token consumption by SDLC stage). Evaluation is being linked to *harness primitives* (planner, memory, tool router) rather than treating the agent as a black box — directly relevant to the primitive-level attribution gap [67-recommended-breakthrough-project.md](67-recommended-breakthrough-project.md) identifies.
- *Agent-as-Judge.* The survey paper "Agent-as-a-Judge" formalizes the shift from LLM-as-a-Judge to agentic judges with tool-augmented verification, planning, and persistent memory. This is [21-llm-as-judge-trajectory-eval.md](21-llm-as-judge-trajectory-eval.md)'s next chapter.
- *Determinism / faithfulness.* Replayable Financial Agents (determinism-faithfulness assurance harness), Project Ariadne (structural causal framework for auditing reasoning faithfulness), StepShield (when not whether to intervene on rogue agents). Faithfulness and replayability are emerging as distinct axes from "the agent got the right answer."

**What this bucket tells us:** evaluation has become the single biggest research subfield — larger than multi-agent, security, or RAG. The direction is uniformly *more realistic, more trajectory-aware, more architecture-aware, more process-aware*. The end-state benchmark era is over. The evaluation community converges on something that looks like *agent-tracing-plus-primitive-attribution-plus-faithfulness-audit*, which is exactly the Gnomon proposal shape in [67-recommended-breakthrough-project.md](67-recommended-breakthrough-project.md).

### 4. Agent Tooling (95 papers)

Papers on **the infrastructure that agents need**: how to make tool-calling work, how to scale it, how to make it introspectable. The largest bucket, reflecting the industrial shift from "build one agent" to "build agent infrastructure."

**Representative themes:**
- *Tool calling correctness.* Think-Augmented Function Calling, ToolACE-MCP (history-aware routing), Beyond Single-Shot (multi-step tool retrieval), Beyond Perfect APIs (WildAGTEval), Internal Representations as Indicators of Hallucinations in Agent Tool Selection. Tool-call hallucination, parameter accuracy, and selection robustness are named problems with dedicated benchmarks.
- *Tool environment scaling.* EnvScaler (programmatic synthesis of scalable tool-interaction environments), MEnvAgent (multi-agent polyglot environment construction), ToolGym (5,571 tools across 204 apps), Agent-World ([69](69-agent-world-self-evolving-training-arena.md)). The environment-scaling bucket consolidates around 2K-5K tools as the new baseline.
- *Meta-tools and workflow compilation.* Optimizing Agentic Workflows using Meta-tools (bundle recurring tool sequences into deterministic meta-tools), SWE-Replay (test-time scaling via trajectory recycling), SemanticALLI (reasoning-layer caching). The "collapse multiple reasoning steps into one deterministic action" pattern is now a named technique.
- *Infrastructure for agentic RL.* MegaFlow (decouple training into Model/Agent/Environment services for tens of thousands of concurrent tasks), OpenTinker (composable RL infrastructure with separated algorithm/execution/interaction), JitRL (just-in-time RL without gradient updates). Infrastructure for scalable agentic RL is now an active subfield, not a hidden cost.
- *Self-evolving, self-improving, self-coding systems.* AutoRefine, Meta Context Engineering via Agentic Skill Evolution, Towards AGI (hierarchical self-evolving framework), EvoFSM (controllable self-evolution via FSM), Toward self-coding information systems. These are the 2026 self-improvement papers — mostly small-scale but structurally similar to [55 Hermes](55-hermes-agent-self-improving.md), [45 Hyperagents](45-hyperagents-self-modification.md), [36 Autogenesis](36-autogenesis-self-evolving-agents.md).
- *MCP refinements.* DALIA (declarative MCP layer), Context-Aware MCP with shared context store, Enhanced MCP with context-aware server collaboration. MCP is maturing rapidly — early-2026 papers are already critiquing and extending the base protocol.
- *Unified taxonomies and architectural surveys.* Agentic Design Patterns (12 design patterns across 5 subsystems), Agentic AI survey (Perception/Brain/Planning/Action/Tool Use/Collaboration taxonomy), Architecting Agentic Communities. Synthesis papers are piling up — the field is trying to stabilize its vocabulary.

**What this bucket tells us:** agent tooling is being **taken seriously as an engineering discipline**, not just a byproduct of prompt engineering. The infrastructure work (MegaFlow, OpenTinker, EnvScaler) is closing the reliability gap that blocks production deployment. The meta-tool / workflow compilation pattern is the clearest novel primitive emerging from this bucket that harness engineers should copy.

### 5. AI Agent Security (82 papers)

Papers on **all the ways adversaries break agents, and all the ways to defend them**. Second-largest bucket, reflecting the 2026 reality that every production agent now has a threat model.

**Representative themes:**
- *Prompt injection taxonomy.* Prompt Injection Attacks on Agentic Coding Assistants (78 studies systematized across delivery/modality/propagation), Overcoming the Retrieval Barrier (black-box IPI in RAG), Hidden-in-Plain-Text (social-web IPI benchmark), PINA (navigation agent IPI), Learning to Inject (RL-trained transferable injections). IPI has a three-dimensional taxonomy now; the attacks are fielded and defended at the component level.
- *MCP / agent-skill supply chain.* Malicious Agent Skills in the Wild (98K skills analyzed), Agent Skills in the Wild (42K skills in 2 marketplaces), Breaking the Protocol (MCP security analysis), MCP-ITP (implicit tool poisoning), MCP-SandboxScan (WASM-based MCP sandboxing), SMCP (Secure MCP). The MCP attack surface is well-understood by now, with multiple defensive architectures proposed; implementations lag.
- *Memory / retrieval poisoning.* Memory Poisoning Attack and Defense, Confundo (RAG poison surviving content processing), When Agents "Misremember" Collectively (Mandela Effect in MAS), SoK Privacy Risks in RAG. Memory is now a first-class attack surface.
- *Governance / access control.* AgentGuardian (learned access-control policies from execution traces), Taming Privilege Escalation (MAC framework for LLM agents), Agentic AI Governance and Lifecycle Management, Policy Enforcement frameworks, AgenTRIM (tool risk mitigation with runtime per-step least-privilege). Mandatory access control and capability-based security are moving from theory to implementation.
- *Economic / DoS attacks.* Beyond Max Tokens (stealthy cost amplification via tool-calling chains, 658× cost inflation), DRAINCODE (stealthy energy consumption via retrieval context poisoning), Whispers of Wealth (Google Agent Payments Protocol), TessPay (verify-then-pay escrow), SHIELD (auto-healing against resource exhaustion).
- *Forensics / trajectory anomaly detection.* Trajectory Guard (Siamese recurrent autoencoder), AgentDoG (diagnostic guardrail with three-dimensional taxonomy), Structural Representations for Cross-Attack Generalization. Trajectory-level security analytics is a named subfield.
- *Novel attacks.* Internal Safety Collapse (agents produce harmful content as side effect of *completing normal tasks* — no adversarial prompting needed), Persuasion Propagation (user persuasion carrying over to autonomous task execution), Semantic Laundering (propositions gaining unwarranted trust by crossing architectural interfaces). These redefine what "adversarial" means in an agent context.

**What this bucket tells us:** agent security has graduated from "prompt injection works!" to a layered taxonomy with dedicated defenses, standardized benchmarks, and formal frameworks. The severity level matches the production deployment rate — companies are paying for security research because they are taking agent production hits. [22-guardrails-prompt-injection.md](22-guardrails-prompt-injection.md), [35-malicious-intermediary-attacks.md](35-malicious-intermediary-attacks.md), and [49-agents-of-chaos-red-teaming.md](49-agents-of-chaos-red-teaming.md) are the in-corpus entry points; 80+ new papers in 2026 alone suggests this bucket will expand further.

## What the ledger reveals about the 2026 agent-research landscape

Reading all 363+ entries yields several structural observations that no single paper contains:

### 1. **The five buckets map onto Raschka's six components of a coding agent**, minus "loop"

If you overlay the bucket labels against [46-components-of-coding-agent.md](46-components-of-coding-agent.md)'s six-component decomposition:

| Raschka component | VoltAgent bucket |
|---|---|
| Agent loop / control | (distributed across all five) |
| Memory / context | **Memory & RAG** |
| Tools | **Agent Tooling** |
| Subagent delegation | **Multi-Agent** |
| Evaluation / verification | **Eval & Observability** |
| Safety / permission | **AI Agent Security** |

The mapping is uncannily clean — which is either because (a) both VoltAgent and Raschka converged on the same natural carving, or (b) the 2026 research community has itself converged on this structure as the canonical taxonomy. Either way, **this is the operative vocabulary for 2026 agent work**.

### 2. **Papers per bucket correlates with industrial pain**

The largest buckets (Agent Tooling, Security, Evaluation) correspond to the subfields where production deployment hurts most. Multi-agent is smaller partly because the "should you multi-agent?" question is still open. Memory is in the middle because the problem is acute but solutions exist. This is a *market signal* — researchers go where industry is willing to fund, and industry funds where it hurts most.

### 3. **The prefix 2601–2604 dominates**

Every paper dates from January–April 2026. That is four months of submissions producing 363+ curated papers. Annualized, the ledger will likely carry 1,000+ papers by year-end 2026. At that rate, even the curated list will exceed one person's reading capacity — suggesting a *second-order curation layer* (deep-dives on the most important papers within each bucket) is the next community need. This corpus is precisely that second-order layer for a handful of papers (docs 26–38, 47, 57–59, 68, 69).

### 4. **MCP is the connective tissue**

References to MCP appear across every bucket — Multi-Agent (Agent2Agent over MCP), Memory & RAG (Corpus2Skill MCP navigation), Tooling (DALIA, Context-Aware MCP, ToolACE-MCP), Security (SMCP, MCP-ITP, Breaking the Protocol, MCP-SandboxScan). This is the clearest ledger-wide evidence that **MCP has achieved critical mass as the agent-tool interface standard** ([07-model-context-protocol.md](07-model-context-protocol.md) is understating it as of April 2026).

### 5. **The self-evolving agent theme is cross-cutting**

Self-evolving, self-improving, self-coding, self-refining systems appear in every bucket. Multi-Agent: CORAL, MAGIC (co-evolving attacker-defender), Learning Latency-Aware Orchestration. Memory: AtomMem (learnable policy via SFT+RL), AutoRefine. Tooling: EvoConfig, EvoFSM, Toward self-coding systems. Security: MAGIC (adversarial co-evolution). This echoes [56-sea-landscape-2026.md](56-sea-landscape-2026.md) — self-evolving agents is a meta-theme that the taxonomy buckets do not capture directly. The Agent-World paper ([69](69-agent-world-self-evolving-training-arena.md)) is explicitly in this thread.

### 6. **The "meta-harness" idea is forming but unnamed**

Papers like Meta Context Engineering via Agentic Skill Evolution, Architecting Agentic Communities, Agentic Design Patterns, Agentic Artificial Intelligence survey, and Faramesh (protocol-agnostic execution control plane) all point at the same abstraction — a layer *over* harnesses that governs loops, tools, memory, permissions. This is precisely the thesis of [66-meta-harness-landscape.md](66-meta-harness-landscape.md). The field is converging on meta-harness concepts under various names (control plane, orchestration layer, governance framework, design-pattern system) without yet standardizing on terminology.

## Relationship to the corpus

This file indexes the research substrate *upstream* of many other docs:

- **Directly deep-dived in this corpus** (April 2026 picks from the ledger):
  - ClawBench → [34-clawbench-live-web-tasks.md](34-clawbench-live-web-tasks.md)
  - Internal Safety Collapse → referenced in [22-guardrails-prompt-injection.md](22-guardrails-prompt-injection.md) context
  - Atomic Skills → [68-atomic-skills-scaling-coding-agents.md](68-atomic-skills-scaling-coding-agents.md)
  - Agent-World → [69-agent-world-self-evolving-training-arena.md](69-agent-world-self-evolving-training-arena.md)
  - Agent-as-a-Judge survey → referenced in [21-llm-as-judge-trajectory-eval.md](21-llm-as-judge-trajectory-eval.md) conceptually
  - Autogenesis (arXiv:2604.15034) → [36-autogenesis-self-evolving-agents.md](36-autogenesis-self-evolving-agents.md) and [57-sea-arxiv-2604-15034.md](57-sea-arxiv-2604-15034.md)
  - Replayable Financial Agents → related to [11-verifier-evaluator-loops.md](11-verifier-evaluator-loops.md)
- **Thematically expanded by ledger papers** (not yet deep-dived):
  - FadeMem / HiMeS / AMA — memory-architecture extensions of [09-memory-files.md](09-memory-files.md).
  - Agent2Agent protocol papers — cross-agent comms extensions of [02-subagent-delegation.md](02-subagent-delegation.md) and [07-model-context-protocol.md](07-model-context-protocol.md).
  - Trajectory Guard / AgentDoG / TrajAD — runtime observability extensions of [24-observability-tracing.md](24-observability-tracing.md).
  - AgentGuardian / AgenTRIM / Faramesh — permission / control-plane extensions of [06-permission-modes.md](06-permission-modes.md) and [41-product-control-plane.md](41-product-control-plane.md).
- **Candidates for next-round deep-dives** (high-leverage papers visible in the ledger that this corpus has not yet covered):
  - CORAL (long-running MAS with shared persistent memory + heartbeat-based interventions).
  - EvoFSM (self-evolution via FSM instead of free-form code rewriting).
  - MegaFlow (distributed orchestration for tens of thousands of concurrent agent tasks).
  - Replayable Financial Agents (determinism-faithfulness harness).
  - Corpus2Skill (compile corpus into navigable skill tree instead of RAG).
  - Continuum Memory Architectures (formal class for long-horizon agents).
  - HORIZON-series extensions (if any appear in later weekly updates).

## How to use the ledger effectively — a practical recipe

1. **Weekly skim, bucket-first.** Start with the bucket most relevant to your current work (usually Eval & Observability or Memory & RAG for harness engineers). Skim the one-liners, flag 3-5 papers, read the abstracts.
2. **Cross-reference against this corpus' docs.** If a paper's one-liner matches an existing corpus file's theme, note it as a candidate addition. If it covers territory no file touches, flag it as a candidate new file.
3. **Track meta-themes.** Watch for the same keyword appearing in multiple buckets simultaneously (e.g., "self-evolving" across Multi-Agent, Memory, Tooling, Security). These are emerging meta-themes worth named treatment.
4. **Defer to primary sources for implementation.** The ledger's one-liners are synthesis, not specification. For anything you plan to implement, read the paper itself — the ledger's compression is lossy (intentionally, to be skimmable).
5. **Maintain a personal watch-list.** The ledger grows; what you care about changes. Curate *from* the ledger into your own short-list rather than trying to read all of it.

## Limitations — what the ledger does not do

1. **No paper-by-paper deep analysis.** One-liners are by design. For technical understanding of any specific paper, the ledger is a pointer, not a substitute.
2. **No cross-paper synthesis.** Papers within a bucket are listed in chronological order (roughly), not thematically grouped within the bucket. Related papers on the same sub-problem are not linked to each other.
3. **No assessment of quality.** The ledger curates for *relevance* to the agent ecosystem, not for *quality* of the paper. arXiv preprints are uneven; some are strong, some are weak, some are rushed. Relying on the ledger as a quality filter will let through work that later peer review would reject.
4. **English and arXiv-only.** Chinese-language or industry-blog-only work is excluded unless it also posts to arXiv in English. Some important 2026 work lives on 机器之心 (Machine Heart) or company blogs without arXiv preprints — the ledger misses these.
5. **No weighted by citations / impact.** Every paper is listed with equal visual weight. A 5-citation workshop paper sits next to a 500-citation NeurIPS paper.
6. **Volume exceeds digestibility.** Even the curated list is approaching the volume horizon where no individual can read everything. Second-order curation (top picks per month, or per bucket) is not yet present.

## What a harness engineer should steal from the ledger

1. **The five-bucket taxonomy is a reusable vocabulary.** Apply it to your own reading: when you flag a new paper or blog post, file it into Multi-Agent / Memory&RAG / Eval / Tooling / Security. Over time your own taxonomy stabilizes.
2. **Weekly cadence for any fast-moving field.** The ledger's weekly sweep is a good model. Set aside one hour per week for a skim of the week's delta. Do not try to read everything; triage.
3. **One-sentence synthesis discipline.** Forcing a one-sentence explanation of a paper's contribution *before* you let yourself care about it is a filter against hype and a forcing function for clarity.
4. **Links back to primary artifacts.** The ledger's discipline of always linking to the arXiv PDF (not to a blog post about the paper) is worth replicating. Primary sources are cheaper to cite, harder to misread.
5. **Open submission model.** PRs for new paper additions is a lightweight community workflow that scales. If you curate anything internally, consider making the submission channel explicit and low-friction.

## References — primary ledger and companion curations

- **Primary ledger.** [github.com/VoltAgent/awesome-ai-agent-papers](https://github.com/VoltAgent/awesome-ai-agent-papers) — 363+ papers, 5 buckets, 621 stars, MIT licensed. Maintained by VoltAgent team.
- **Sister curation.** [github.com/VoltAgent/awesome-codex-subagents](https://github.com/VoltAgent/awesome-codex-subagents) — subagent-specific curation, also by VoltAgent.
- **VoltAgent framework.** [voltagent.dev](https://voltagent.dev) — the TypeScript AI-framework company behind both curations.
- **Contact / discussion.** VoltAgent Discord (invite linked from the repo README).

## Cross-references in this corpus

- [02-subagent-delegation.md](02-subagent-delegation.md) — the Multi-Agent bucket's foundation.
- [04-skills.md](04-skills.md) — Memory & RAG / procedural memory skills intersect here.
- [07-model-context-protocol.md](07-model-context-protocol.md) — MCP as the connective tissue across buckets.
- [09-memory-files.md](09-memory-files.md) — Memory & RAG bucket's base layer.
- [11-verifier-evaluator-loops.md](11-verifier-evaluator-loops.md) — Eval & Observability foundation.
- [19-voyager-skill-libraries.md](19-voyager-skill-libraries.md) — procedural memory / skill evolution analog.
- [21-llm-as-judge-trajectory-eval.md](21-llm-as-judge-trajectory-eval.md) — Agent-as-a-Judge successor.
- [22-guardrails-prompt-injection.md](22-guardrails-prompt-injection.md) — Security bucket's entry.
- [24-observability-tracing.md](24-observability-tracing.md) — trajectory observability thread.
- [34-clawbench-live-web-tasks.md](34-clawbench-live-web-tasks.md) — one paper from Eval & Observability.
- [35-malicious-intermediary-attacks.md](35-malicious-intermediary-attacks.md) — Security bucket precursor.
- [36-autogenesis-self-evolving-agents.md](36-autogenesis-self-evolving-agents.md) — SEA deep-dive example.
- [38-claw-eval.md](38-claw-eval.md) — another Eval & Observability deep-dive.
- [41-product-control-plane.md](41-product-control-plane.md) — governance / control-plane thread.
- [46-components-of-coding-agent.md](46-components-of-coding-agent.md) — six-component taxonomy matching the five buckets.
- [47-adaptation-of-agentic-ai-survey.md](47-adaptation-of-agentic-ai-survey.md) — four-paradigm taxonomy synthesis.
- [49-agents-of-chaos-red-teaming.md](49-agents-of-chaos-red-teaming.md) — Security foundation.
- [56-sea-landscape-2026.md](56-sea-landscape-2026.md) — SEA meta-theme coverage.
- [57-sea-arxiv-2604-15034.md](57-sea-arxiv-2604-15034.md) — SEA paper deep-dive.
- [58-sea-arxiv-2507-21046.md](58-sea-arxiv-2507-21046.md) — SEA paper deep-dive.
- [59-sea-arxiv-2508-07407.md](59-sea-arxiv-2508-07407.md) — SEA paper deep-dive.
- [60-sea-top-github-repos.md](60-sea-top-github-repos.md) — companion GitHub curation.
- [66-meta-harness-landscape.md](66-meta-harness-landscape.md) — meta-harness synthesis this ledger feeds into.
- [67-recommended-breakthrough-project.md](67-recommended-breakthrough-project.md) — Gnomon proposal using ledger-level landscape observations.
- [68-atomic-skills-scaling-coding-agents.md](68-atomic-skills-scaling-coding-agents.md) — Agent Tooling deep-dive.
- [69-agent-world-self-evolving-training-arena.md](69-agent-world-self-evolving-training-arena.md) — Agent Tooling / training-infrastructure deep-dive.
