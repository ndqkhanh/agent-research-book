# 191 — From Skills to Talent: OneManCompany — Organising Heterogeneous Agents as a Real-World Company

**Paper.** Zhengxu Yu, Yu Fu, Zhiyuan He, Lee Ka Yiu, Weilin Luo, Yuxuan Huang, Meng Fang, Jun Wang — *From Skills to Talent: Organising Heterogeneous Agents as a Real-World Company* — arXiv:2604.22446v1 — Huawei Noah's Ark Lab / University of Liverpool / University College London — April 24, 2026 (33 pages, 13 figures). Code: https://github.com/1mancompany/OneManCompany (Apache-2.0, ~165 stars, Python 83.9% / JS 11.2%). Talent Market: https://one-man-company.com/market. HuggingFace daily papers: 116 upvotes (week of 2026-W18).

**One-line definition.** OneManCompany (OMC) is the first multi-agent framework that promotes individual agents from "skill bundles" to **portable Talent identities** decoupled from execution Containers, organised under a real-company structure (CEO + COO + HR + EA founding team, dynamically-formed departments), governed by an HR lifecycle (review → PIP → offboarding), and orchestrated through an **E²R Tree Search** (Explore–Execute–Review) with seven formal correctness guarantees — posting **84.67% on PRDBench vs 69.19% best baseline (+15.48pp)** at $6.91/task.

## Why this paper matters

Multi-agent frameworks have spent two years adding *more* — more roles (ChatDev, MetaGPT), more graph topologies (LangGraph, AutoGen), more coordination patterns (CrewAI, Magentic-One). What none of them addressed is the structural problem that makes real organizations work: **the unit of staffing is not a skill, it's a person**. A skill is a callable function; a person is a persistent identity who carries skills, accumulates a reputation, can be reviewed, promoted, performance-improved, and ultimately fired. ChatDev's "CTO" is not an entity that survives projects; it's a prompt template instantiated and discarded. MetaGPT's "engineer" is the same. AutoGen's agents are graph nodes. None of them have lifecycle, none accumulate cross-project reputation, and none can be replaced by an equivalent agent on a different runtime without code changes.

OMC's reframe — **skills → talent** — is structurally analogous to the move from procedure-call to object-with-state in software engineering. A *skill* is a tool/script/prompt. A *Talent* is a portable identity package — role definition, system prompts, working principles, tool configurations, skill scripts, domain-knowledge files, benchmark-validated capability signatures — that can run on any compatible **Container** (LangGraph, Claude Code, OpenClaw, OpenRouter, self-hosted) without modification. Talent and Container are decoupled by **six typed interfaces**, so the same Talent runs across runtimes and the same Container hosts different Talents. This is the first time multi-agent literature has formalized substitutability of identity and substrate.

The organizational model is a real-company structure rather than an abstract topology. A CEO (the only human stakeholder) submits directives. The Executive Assistant decomposes them into project phases. The COO chooses execution strategy and runs retrospectives, distilling SOPs. The HR Manager queries a **Talent Market** for hires, runs periodic reviews, and manages PIP / offboarding. Departments form dynamically as needed (Researcher, Writer, Game Dev, Art Designer, Paper Scientist, AV Producer, AI Engineer in case studies). Communication runs through an organisational event bus and per-employee task queues. Every node in the resulting task tree carries a state machine (PENDING → PROCESSING → COMPLETED → ACCEPTED → FINISHED, with FAILED branch). The whole system runs an **E²R Tree Search** that explores decompositions, executes leaves, reviews results, and re-explores when reviews fail.

Three things land hardest. First, the Talent–Container split unlocks **cross-runtime mixing without code changes**: empirically, OMC mixes GPT-5.2 + Claude Sonnet 4 (content gen), Claude + Gemini 2.5 (game dev), Gemini 3.1 Pro alone (audiobook), Claude Sonnet 4.6 + self-hosted (research survey). Second, the **HR lifecycle** (3-project rolling reviews → PIP after 3 failed reviews → offboarding after 1 PIP failure) gives multi-agent systems their first lifecycle governance — agents that fail their projects don't keep running silently; they enter coaching, then deprovisioning. Third, the **seven formal guarantees** (DAG invariant, mutual exclusion, schedule idempotency, review termination, cascade completeness, dependency completeness, recovery correctness) give bounded-time, bounded-cost, deadlock-free termination on a *dynamically expanding* task tree — a property no prior multi-agent framework provably has.

Take the paper seriously and three downstream things change. (1) **Multi-agent topology debate becomes a smaller question** — once Talent/Container is decoupled, the topology debate is just an HR routing question over a substitutable workforce. (2) **The benchmark question shifts** — from "which framework solves task X?" to "which Talent profile (skill set, principles, knowledge) solves task X best, and on which Container?" (3) **Self-improving multi-agent harnesses gain a governance shape** — review/PIP/offboarding is not a metaphor; it's a concrete recipe for which agents survive, get promoted, or get retired in a continuously-running system.

## Problem it solves

- **Agents have no persistent identity.** ChatDev's "CTO," MetaGPT's "engineer," AutoGen's role-agents are stateless prompt templates re-instantiated per run. They carry no reputation, accumulate no improvements, and cannot be ranked, reviewed, or retired.
- **Frameworks lock you into one runtime.** ChatDev/MetaGPT pipelines run on a single SDK; AutoGen graphs run on AutoGen; CrewAI roles run on CrewAI. Mixing GPT + Claude + self-hosted in the same project requires plumbing per integration.
- **No organizational evolution.** Frameworks ship with fixed roles and fixed pipelines. There is no notion of an agent improving across projects, being promoted to a larger department, or being replaced by a higher-rated agent from a marketplace.
- **No lifecycle governance.** Underperforming agents keep running; over-budget projects keep spending; deadlocked task trees keep waiting. There is no formal termination guarantee, no review gate, no offboarding mechanism.
- **No formal correctness on dynamically-expanding plans.** Agentic frameworks decompose tasks at runtime, then sub-decompose, but cannot prove their decomposition will terminate, won't deadlock, and won't loop. OMC proves seven properties on its E²R Tree Search.

## Core idea in one paragraph

A multi-agent system should be modeled as a **real company**: a founding team of CEO (human), COO (strategy), HR (hiring/review/firing), EA (decomposition); dynamic departments staffed from a marketplace of **Talents** (portable identity packages — system prompt, skills, tools, knowledge, capability signature, reputation) running on any compatible **Container** (LangGraph, Claude Code, OpenClaw, OpenRouter, self-hosted) via six typed interfaces. Tasks are decomposed into a tree where every node has a finite-state machine and an AND-resolution semantics; execution is **E²R Tree Search** (Explore decomposition + assignment, Execute leaves, Review results, re-Explore on rejection). HR runs every 3 projects, scoring completion quality, pass rate, and collaboration. Three failed reviews → Performance Improvement Plan (coaching, closer supervision, adjusted assignments). One more failure → automated offboarding and gap-flag for re-recruitment. Top performers' refined Talents are published back to the Talent Market — a knowledge flywheel. Seven formal guarantees on the search (DAG invariant via DFS cycle detection, mutual exclusion on employees, schedule idempotency, review termination via bounded retries, cascade completeness, dependency completeness, recovery correctness) yield bounded-time, bounded-cost termination plus deadlock-freedom. Empirically, this beats minimal-agent, ChatDev-style, AutoGen-style, and Claude-Code-style baselines on PRDBench by 15.48pp at $6.91/task.

## The Talent–Container split

The Talent is a portable artifact. Schema (paper §3, App. A.1):
- **Identity**: name, role, version.
- **System prompt**: persistent operating frame.
- **Working principles**: behavioral guardrails (e.g., "verify before commit," "ask CEO before scope expansion").
- **Tool configurations**: which tools/MCP servers, with credentials abstracted.
- **Skill scripts**: deterministic recipes (Python, shell) the Talent reaches for.
- **Domain knowledge files**: Markdown / PDF / structured notes the Talent loads on activation.
- **Capability signature**: benchmark-validated profile (PRDBench score, domain pass rates, peer-review ratings).

The Container is a runtime that satisfies the six typed interfaces:
1. **Lifecycle** — instantiate, hibernate, deactivate hooks (with pre/post validation, guardrails).
2. **Execution** — receive a task, return a result.
3. **Memory** — read/write workspace, episodic state, persistent skill cache.
4. **Tool** — invoke configured tools, return results.
5. **Communication** — publish/subscribe on the organisational event bus, point-to-point review messages.
6. **Reflection / Review** — accept feedback, update working principles or capability signature.

Concretely, a Talent file lives in `talents/<name>.yaml`; a Container is a code module implementing the six interfaces (LangGraph adapter ≈ 200 LOC, Claude Code adapter ≈ 350 LOC). The same Talent runs on either; the same Container hosts any Talent.

This is the first time multi-agent literature has formalized **substitutability**: identity and substrate are independent dimensions. The contrast to prior frameworks is sharp:

| Framework | Substitutable identity? | Substitutable runtime? | Cross-project reputation? | Lifecycle governance? |
|---|---|---|---|---|
| ChatDev | ✗ (prompt template) | ✗ (CAMEL only) | ✗ | ✗ |
| MetaGPT | ✗ (role class) | ✗ (MetaGPT runtime) | ✗ | ✗ |
| AutoGen | ✗ (agent graph node) | partial | ✗ | ✗ |
| LangGraph | ✗ (state graph) | partial | ✗ | ✗ |
| CrewAI | ✗ (role class) | ✗ (CrewAI runtime) | ✗ | ✗ |
| **OMC** | **✓ Talent file** | **✓ Container interfaces** | **✓ capability signature** | **✓ HR lifecycle** |

## The founding team

Four roles, three of which are agents. Their mandates are codified in the paper (§4):

- **CEO (human).** The only human in the loop. Submits directives, approves recruitment, can inject mid-search overrides ("change scope," "stop"), accepts or rejects final deliverables. The CEO's stop / approve signal is what gates termination.
- **EA (Executive Assistant).** Decomposes CEO directives into project phases with budgets, deadlines, success criteria.
- **COO.** Chooses decomposition strategy per phase (sequential, parallel, recursive); runs project retrospectives; distills standard operating procedures into reusable artifacts.
- **HR Manager.** Queries the Talent Market for hires; runs periodic reviews (every 3 projects); manages PIP and offboarding; publishes top-performer Talents back to the marketplace.

Departments form dynamically as the task tree expands. The case studies in the paper instantiate departments for Research, Writing, Game Development, Art Design, Paper Science, AV Production, and AI Engineering — none predefined; all spawned by HR matching marketplace Talents to decomposition needs.

## E²R Tree Search — formal model

The system maintains a search tree 𝒯 = (V, E_tree, E_dep), where E_tree are decomposition edges and E_dep are dependency edges across siblings. Each node v carries:
- description d_v
- assigned employee e_v
- status φ_v ∈ {PENDING, PROCESSING, COMPLETED, ACCEPTED, FINISHED, FAILED}
- result r_v
- accumulated cost c_v
- workspace 𝒲
- review log ℛ

A **policy** π(𝒯) at iteration emits an action sequence:
  σ = (α_d(v, {v'₁, …, v'ₙ}), α_a(v'₁, e₁), …, α_a(v'ₙ, eₙ))
That is, decompose v into children {v'₁..v'ₙ} (α_d), then assign each child to an employee (α_a).

**Execution** of a leaf: (r_v, c_v) = f_ev(d_v) — the employee runs the task description and returns a result + cost.

**AND-semantics resolution.** A non-leaf v is resolved iff all non-skipped children are resolved. A leaf is resolved iff φ_v ∈ {ACCEPTED, FINISHED}. This guarantees that the parent's "complete" claim reflects all child completions.

**Ready condition.** ready(v) ⟺ φ_v = PENDING ∧ ∀u ∈ deps(v): φ_u ∈ {ACCEPTED, FINISHED}. A node only enters PROCESSING when its dependency set is satisfied.

**Circuit breakers.**
- Review-bound: n_rev(v) ≥ k_rev = 3 → escalate to supervisor or to the CEO.
- Time-bound: t_exec(v) > T_max = 3600 s → fail the node, trigger recovery.
- Cost-bound: Σ_v c_v > B (project budget) → pause and request CEO approval to extend.

The state machine (App. A.4) has explicit FAILED transitions that trigger bounded retries up to k_retry, then escalation.

### Seven formal guarantees

The paper proves (App. A.5–A.11) seven properties of the E²R Tree Search:

1. **DAG invariant.** Every node insertion runs DFS cycle detection; the tree-plus-dependencies graph remains a DAG throughout.
2. **Mutual exclusion.** Each employee processes at most one task simultaneously (single-task assertion in Container's Lifecycle interface).
3. **Schedule idempotency.** Re-issuing the same schedule (e.g., after a crash recovery) produces no duplicated work.
4. **Review termination.** Bounded review retries (k_rev = 3) plus escalation prevent infinite review loops.
5. **Cascade completeness.** When a node is rejected and re-enters Explore, all dependent children are correctly invalidated and rescheduled.
6. **Dependency completeness.** Failed dependencies either cascade-cancel descendants or block them with explicit failure flag, never silently leave them ready.
7. **Recovery correctness.** A crashed worker resuming from persisted workspace produces results consistent with un-crashed execution.

Together these yield: bounded-time termination (t ≤ Σ T_max + k_retry · per-node retry), bounded-cost termination (Σ c_v ≤ B before pause), and deadlock-freedom under dynamic decomposition. No prior multi-agent framework provably has these.

## The HR lifecycle

The HR Manager runs every 3 completed projects on every employee. Score factors:
- **Completion quality** — supervisor-accepted result rate.
- **Pass rate** — fraction of nodes that reached FINISHED without re-review.
- **Collaboration** — peer feedback from co-assigned siblings.

Three rolling reviews:
- **Pass** — capability signature updated; eligible for promotion (refined Talent published to marketplace).
- **3 consecutive fails** → **Performance Improvement Plan (PIP)**: HR adjusts the Talent (system-prompt edits, skill additions, tool configuration changes), assigns coaching from a higher-rated peer, and tightens supervision (lower k_rev for this employee's tasks).
- **PIP failure (1 more failed review under PIP)** → **Offboarding**: deprovision Talent, log gap, HR re-queries Talent Market to fill.

Top performers' refined Talents are *published back to the Talent Market* with their improved capability signature. This creates a knowledge flywheel: the marketplace accumulates not just initial Talents but field-tested, evolved Talents from prior projects.

## The Talent Market

Three sourcing channels:

1. **Community-contributed peer-reviewed packages.** Talents published openly with capability signatures; community ratings drive discoverability.
2. **AI-recommended assembly.** When HR queries the marketplace and no good match exists, an AI recommender scrapes web for relevant skills, tools, and domain knowledge, assembles a candidate Talent, and ships it for review. This addresses long-tail domains the marketplace doesn't cover.
3. **Internal promotion.** Top performers' refined Talents are republished — the company's own evolved workforce becomes a public resource.

The marketplace is currently hosted at https://one-man-company.com/market.

## Mechanism (step by step)

**1. CEO submits directive.** "Build me a turn-based street-fight game with two characters and a tutorial."

**2. EA decomposes** into project phases: design, asset production, game logic, integration, playtest, polish.

**3. COO selects strategy:** parallel for design + asset production; sequential for game logic → integration; iterative for playtest → polish.

**4. HR queries Talent Market** for each phase. Game-design Talent already exists (Claude Sonnet 4 Container); art-asset Talent exists (Gemini 2.5 Container); game-logic Talent (Claude Sonnet 4.6 Container, Code Interpreter tool); integration Talent (Claude Code Container).

**5. CEO approves** the proposed roster.

**6. E²R execution.** Each phase becomes a sub-tree; tasks are decomposed further as they encounter complexity; leaves execute; supervisors review; rejected tasks re-enter Explore with updated context; cascade-failed dependencies are reassigned.

**7. Review gates fire.** Game logic agent fails 3 reviews on physics. PIP triggers: HR adds a Code Interpreter tool, gives access to a vetted physics-sim skill from the marketplace, assigns a tutoring session with a Math-Reasoning Talent, lowers k_rev for the next 3 tasks.

**8. Project completes.** COO retrospective distills "physics-sim integration" into a reusable SOP added to the project workspace's knowledge artifacts.

**9. Talent updates.** Game logic agent's capability signature now includes physics-sim pass; refined Talent (Claude Sonnet 4.6 + physics-sim skill + integration SOP) republished to marketplace.

This loop runs uninterrupted across multi-day projects in the case studies.

## Empirical results

### PRDBench headline (paper Table 2)

PRDBench (arXiv:2510.24358) is a 50-task project-level Python benchmark across 20+ domains, each task with a structured PRD plus comprehensive evaluation criteria scored agent-as-judge across ~1,200 scoring points. Run in DEV mode (one-shot, no iterative human feedback).

| System | PRDBench score |
|---|---:|
| GPT-5.2 (single agent) | 62.49% |
| Claude-4.5 (minimal agent) | 69.19% |
| CodeX | 62.09% |
| Claude Code | 56.65% |
| **OMC (Claude Code Sonnet 4.6 + Gemini 3.1 Flash Lite)** | **84.67%** |

OMC beats the strongest baseline (Claude-4.5 minimal agent) by **+15.48 percentage points**.

### Cost

Across 50 PRDBench tasks, OMC spent **$345.59**, ≈ **$6.91 / task**. The paper does not report cost for baselines (a methodological gap; OMC's cost-per-task is the only number).

### Case studies

- **GitHub AI weekly trend report.** $4.48, < 10 minutes. Talents: GPT-4o (research) + Claude Sonnet 4 (writing). Output: structured Markdown with sources.
- **Street-fight game.** Multi-day with human-in-the-loop refinement. Talents: Claude Sonnet 4.6 (game logic) + Gemini 2.5 (art) + Claude Sonnet 4 (design).
- **Audiobook with 16 scenes + voice-over + music + 2 videos.** $1.57. Talent: Gemini 3.1 Pro alone, with the AV Producer and Art Designer departments instantiated.
- **Automated research survey on world models.** $16.26, < 1 hour. Talents: Claude Sonnet 4.6 (research lead) + self-hosted agent (literature scraping). Output: 17 documents, 70-node mind map, 3 novel ideas surfaced.

These case studies are qualitative — they demonstrate the heterogeneity range and the cross-domain generalization, but only PRDBench is the quantitative benchmark.

## Variants and ablations

- **Founding team configurations.** CEO is always a human; COO/HR/EA are agents. Smaller deployments collapse COO + HR into a single agent; the paper notes this works for ≤ 3 employees but loses the review/PIP discipline.
- **Talent Market sourcing.** Community-only vs AI-recommended-assembly: AI-assembly is required for long-tail domains (audiobook AV production was AI-assembled); community-only suffices for software domains.
- **Container backends.** Claude Code (best for code-heavy tasks), LangGraph (best for graph-explicit pipelines), OpenClaw (experimental), OpenRouter (best for cost-mixed deployments), self-hosted (best for data-sensitive deployments). Heterogeneous mixes outperformed homogeneous in 4 / 4 case studies.
- **Self-evolution.** The talent-republishing knowledge flywheel is implemented but not quantitatively ablated; the paper acknowledges this as future work.
- **Circuit-breaker tuning.** k_rev = 3, k_retry implementation-defined, T_max = 3600 s, B is project-specific. Defaults work; per-domain tuning is a future-work item.

## Failure modes and limitations

- **Single-benchmark quantitative evaluation.** Only PRDBench (software-only). Non-coding generality is shown only via qualitative case studies. Open-ended tasks (writing, design, research) have no comparable benchmark.
- **Cost comparison missing for baselines.** OMC's $6.91/task is reported; baselines' costs are not. The +15.48pp gain may or may not survive cost-normalized comparison.
- **No quantitative ablation of self-evolution.** The talent-republishing flywheel is implemented but not measured. How much does cross-project evolution add over fresh-Talents-per-project?
- **Convergence depends on the human CEO's stop/approve judgments.** The seven formal guarantees yield bounded-time/cost termination, but task-quality termination (the deliverable is "good enough") depends on the human signal. Fully autonomous CEO is untested.
- **Coordination failure modes.** Deadlock-detector and DFS cycle prevention catch structural failures; semantic deadlock (two agents waiting on incompatible semantic preconditions) is not covered formally.
- **Role drift.** Mitigated by Lifecycle interface hooks (pre/post validation, guardrails) and persistent Talent artifacts that survive sessions, but no quantitative measurement of drift across long projects.
- **Repository traction is early.** 165 GitHub stars vs 116 HF upvotes suggests the community is reading the paper but not yet running the framework at scale. Production-readiness signals are limited.
- **Talent Market is an early-stage marketplace.** No published quality-control mechanism for community-contributed Talents; rating systems and abuse detection are unstated.
- **Cost overhead is real.** $6.91/task is acceptable for project-scale tasks (the case-study scope) but heavy for short-form interactions.

## When to use, when not

**Use** for project-scale, multi-day, multi-domain tasks where a single LLM or fixed pipeline plateaus (PRD-to-code, research surveys, multimedia production); when heterogeneous backends are unavoidable (data-sensitive on-prem + best-quality cloud); when self-improving/evolving workforce is a goal; when you need lifecycle governance (review, PIP, offboarding) for production-running agents; when the formal correctness guarantees (bounded-time, deadlock-freedom) matter (regulated domains, safety-relevant pipelines); when the Talent Market's long-tail coverage helps (specialized domains beyond your team's expertise).

**Don't use** for short-form, single-shot interactions (chat, single tool call); for cost-sensitive deployments below ~$1/task; when the team prefers a single-runtime stack and Talent–Container heterogeneity adds operational burden; for quantitative-only benchmarks where PRDBench-style structured evaluation isn't the goal; when human-CEO availability is constrained (the framework currently depends on human approval at recruitment + final delivery); for replacement of single-agent harnesses on small projects where the org overhead exceeds the value.

## Implications for harness engineering

- **Substitutability becomes a first-class harness primitive.** [02-subagent-delegation](02-subagent-delegation.md) and [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md) describe orchestrator-worker and coordination patterns. OMC's Talent–Container split is the next abstraction layer: identity is portable across runtimes. A future harness should treat agent identity as a versioned artifact (like a container image), the runtime as a separate concern (like a container orchestrator), and routing as a marketplace query.
- **Skills get a wrapper: Talent.** [04-skills](04-skills.md) defines model-invocable skill packages with progressive disclosure. OMC's Talent is a *bundle* of skills + system-prompt + working-principles + capability-signature. The skill-discovery literature ([177-skills-discovery-curator](177-skills-discovery-curator-strongest-2026-techniques.md), [178-online-skill-discovery-and-curation-on-the-go](178-online-skill-discovery-and-curation-on-the-go.md)) operates at the skill level; OMC operates at the talent level. Both are needed: skills are the units of capability, talents are the units of staffing.
- **Lifecycle governance for self-improving agents.** [165-ralph-autonomous-loop](165-ralph-autonomous-loop.md), [167-autoskill-experience-driven-lifelong-learning](167-autoskill-experience-driven-lifelong-learning.md), and [169-coevoskills-co-evolutionary-verification](169-coevoskills-co-evolutionary-verification.md) describe self-improving loops without explicit governance. OMC's HR lifecycle (review/PIP/offboarding) provides the governance shape that [190-agentic-world-modeling-taxonomy](190-agentic-world-modeling-taxonomy.md)'s L3 boundary conditions specify in the abstract.
- **Formal-correctness guarantees for agentic plans.** [03-plan-mode](03-plan-mode.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), and [123-robustness-fault-tolerance](123-robustness-fault-tolerance.md) discuss planning and verification but stop short of bounded-termination proofs. OMC's seven E²R guarantees (DAG, mutual exclusion, idempotency, review-termination, cascade-completeness, dependency-completeness, recovery-correctness) are the first published spec for a multi-agent planner that *provably terminates* on a dynamically-expanding plan.
- **The CEO interaction pattern is the durable HITL design.** [23-human-in-the-loop](23-human-in-the-loop.md) describes high-stakes-action approval. OMC's CEO interaction is more constrained: approve recruitment, approve cost extensions, accept/reject final delivery. This is a tighter HITL surface than per-action approval and should generalize to other multi-agent harnesses.
- **Eywa's federation, RecursiveMAS's bridges, OMC's organization.** [103-eywa-heterogeneous-fm-collaboration](103-eywa-heterogeneous-fm-collaboration.md) federates non-LLM FMs through MCP. [189-recursive-multi-agent-systems](189-recursive-multi-agent-systems.md) bridges heterogeneous LLMs through latent links. OMC organizes heterogeneous agents through Talent–Container interfaces and HR governance. The three together sketch a stack: federation API for cross-modality, latent bridge for cross-LLM, talent abstraction for cross-runtime + cross-project. A production harness building on all three would have substitutability at three levels: model, agent, organization.
- **Vendor lock-in is now an organizational concern, not a model concern.** [147-vendor-lock-in](147-vendor-lock-in.md) treats LLM-vendor lock-in as a model-routing question. OMC's Talent–Container split makes the vendor decision a per-Container concern; switching from LangGraph to Claude Code is a Container swap, not a Talent rewrite. This aligns with [126-frameworks-comparison](126-frameworks-comparison.md)'s framing of frameworks as substitutable substrates.
- **Skill flywheel via marketplace.** [167-autoskill-experience-driven-lifelong-learning](167-autoskill-experience-driven-lifelong-learning.md) describes per-agent skill accumulation. OMC's Talent Market makes the flywheel *cross-organizational* — top performers become public Talents, accelerating the field's collective workforce. This is the multi-agent analog of HuggingFace's model hub: a marketplace of evolved staffing artifacts.
- **The COO retrospective is harness-as-meta-agent.** [157-may-2026-synthesis-memory-and-skills](157-may-2026-synthesis-memory-and-skills.md) frames memory and skills as the two trainable substrates. OMC adds a third: SOPs distilled by retrospectives. The COO is a harness-level meta-agent whose output is a workspace artifact (SOP) that all subsequent project agents consult. This is closer to [185-memory-integration-playbook](185-memory-integration-playbook.md)'s organizational memory than to per-agent memory.
- **PRDBench as the project-level benchmark to watch.** OMC's quantitative win is on PRDBench; the benchmark's structured-PRD-with-1,200-scoring-points design is itself a contribution. Multi-agent harness evaluation should add PRDBench-style project-level benchmarks alongside trajectory-level ([21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md)) and live-task ([34-clawbench-live-web-tasks](34-clawbench-live-web-tasks.md)) tests.
- **Heterogeneity that matters is *task-conditioned*, not vendor-conditioned.** [98-diversity-collapse-mas](98-diversity-collapse-mas.md) showed mixing GPT + Claude doesn't help. Eywa showed mixing LLMs + non-LLM FMs does. RecursiveMAS showed mixing tuning-objective specialists does. OMC adds: mixing *role-tuned* Talents on appropriate Containers does. The pattern across all four: heterogeneity is useful when it tracks structural task differences (modality, tuning, role) rather than vendor differences.

The one-line takeaway for harness designers: **the next abstraction above "agent" is "Talent" — a portable, reviewable, replaceable identity package — and the next abstraction above "framework" is "Container" — a substitutable runtime — and the formal HR lifecycle is what makes a self-improving multi-agent system actually safe to run.**

## References and cross-links

- PRDBench paper: arXiv:2510.24358 (Oct 2025).
- Adjacent (different) hierarchical-MAS work: *Talk Structurally, Act Hierarchically* (arXiv:2502.11098).
- Prior MAS frameworks: ChatDev ([92-chatdev](92-chatdev.md)), MetaGPT ([20-metagpt-role-based-workflows](20-metagpt-role-based-workflows.md), [91-metagpt-deep](91-metagpt-deep.md)), AutoGen, LangGraph, CrewAI ([164-crewai-multi-agent-framework](164-crewai-multi-agent-framework.md)).
- Internal cross-links: [02-subagent-delegation](02-subagent-delegation.md), [04-skills](04-skills.md), [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md), [126-frameworks-comparison](126-frameworks-comparison.md), [167-autoskill-experience-driven-lifelong-learning](167-autoskill-experience-driven-lifelong-learning.md), [177-skills-discovery-curator](177-skills-discovery-curator-strongest-2026-techniques.md), [185-memory-integration-playbook](185-memory-integration-playbook.md), [189-recursive-multi-agent-systems](189-recursive-multi-agent-systems.md), [190-agentic-world-modeling-taxonomy](190-agentic-world-modeling-taxonomy.md), [193-recursive-world-organizations-synthesis](193-recursive-world-organizations-synthesis.md).
- Project artifacts: paper https://arxiv.org/abs/2604.22446, code https://github.com/1mancompany/OneManCompany, marketplace https://one-man-company.com/market, HF page https://huggingface.co/papers/2604.22446.
