# 69 — Agent-World: Scaling Real-World Environment Synthesis for Evolving General Agent Intelligence (arXiv:2604.18292)

**Definition.** *Agent-World* (Dong et al., Renmin University of China × ByteDance Seed; arXiv:[2604.18292](https://arxiv.org/abs/2604.18292), submitted 20 Apr 2026) is a **self-evolving training arena** for general-purpose agent intelligence. It attacks the "training environment drought" bottleneck head-on: agents need realistic, stateful, tool-rich environments to learn from, and hand-crafting those environments does not scale. Agent-World's two contributions are (1) **Agentic Environment-Task Discovery** — an autonomous pipeline that mines topic-aligned databases and executable tool ecosystems from thousands of real-world themes, producing **1,978 environments with 19,822 tools** — and (2) **Continuous Self-Evolving Agent Training** — a closed-loop training regime where multi-environment reinforcement learning alternates with dynamic task synthesis and capability-gap diagnosis, co-evolving the agent policy with its environment ecosystem. Across **23 challenging agent benchmarks**, Agent-World-8B and Agent-World-14B consistently outperform strong proprietary models and environment-scaling baselines, demonstrating clear scaling trends in both environment diversity and self-evolution rounds.

This file extends the training-methodology thread opened in [68-atomic-skills-scaling-coding-agents.md](68-atomic-skills-scaling-coding-agents.md). Where *Atomic Skills* decomposes *what the policy learns* (five basis capabilities); Agent-World scales up *where the policy learns* (1,978 distinct environments, 19,822 tools). Both are paradigm papers published within two weeks of each other in April 2026; together they represent a new consensus that **environment scale and skill decomposition are the two axes along which agentic RL now scales** — replacing the older consensus that "train on one good composite benchmark."

## Problem it solves

The paper names the bottleneck precisely and argues it affects every training lab: **tool-using general agents need stateful, realistic, compositional environments**, but those environments are prohibitively expensive to hand-author. Two existing approaches fall short:

1. **Simulated environments.** LLMs act as textual world models and generate environment feedback. Highly scalable, but vulnerable to hallucinations — the "world model" drifts from real-world dynamics, and agents trained on it develop habits that fail outside the simulator.
2. **Realistic environments.** Real executable tools wrapped around real databases. Strong grounding (τ²-Bench, ClawEval exemplify this direction), but building one of these takes weeks of engineering, so coverage is narrow.

Both approaches leave two bottlenecks unresolved:

- **Scalable realism and complex environment synthesis.** Purely LLM-generated environments mismatch real-world interaction logic; open-source toolchains are limited in scope. Neither supports training on long-horizon, state-intensive tasks.
- **Continuous self-evolving training mechanisms.** Even if you had 1,000 environments, existing work uses them for one-shot training, not continual curriculum. No principled way to let environment quality *drive* what the agent trains on next.

Agent-World's thesis: **the web already contains abundant high-value structured data that can be updated in real time**. Stop generating synthetic databases; start *mining* them. Then **let the same environment ecosystem serve double duty** as a diagnostic arena that guides where the agent needs more training — turning environment scale and training into a closed loop instead of two sequential steps.

This is a direct response to the limitation [47-adaptation-of-agentic-ai-survey.md](47-adaptation-of-agentic-ai-survey.md) documents (environment scarcity as a gating factor on adaptation) and to the environment-stagnation critique in [27-horizon-long-horizon-degradation.md](27-horizon-long-horizon-degradation.md).

## Scope of the environment ecosystem — the numbers

Before diving into method, the raw numbers set expectations:

- **1,978 environments** after quality filtering (from thousands of seed themes).
- **19,822 distinct tools** averaging >10 per environment, with some environments exceeding 40.
- **20 first-tier / 50 second-tier / 2,000+ third-tier taxonomic labels** — a deliberately hierarchical organization to enable stratified sampling during training and evaluation.
- **Multiple database formats** ingested: JSON, CSV, SQL, HTML, TeX, YAML — reflecting real-world file diversity rather than forcing one canonical schema.
- **Tasks ≥ 7 interaction turns minimum; average > 20; some > 40.** Difficulty is controlled explicitly (see §2.3 below).
- **Pass@10 with Doubao-Seed-2.0-pro** shows only a small fraction of tasks are solved in all 10 attempts; most are solved once or not at all. Deliberately *hard* training distribution.

Comparison to the corpus: this is roughly 10-100× the environment diversity of prior published work. LinuxArena ([26-linuxarena-production-agent-safety.md](26-linuxarena-production-agent-safety.md)) has high-fidelity Linux environments but single digits of them. ClawBench ([34-clawbench-live-web-tasks.md](34-clawbench-live-web-tasks.md)) spans 144 live sites but tests, not training. Agent-World is the first published pipeline that puts 1K+ environments *into training*.

## Component 1 — Agentic Environment-Task Discovery

The discovery pipeline is itself an agentic system: its output is the environment ecosystem that downstream training will consume. The authors use an agentic deep-research pipeline to do what would otherwise require thousands of engineering hours.

### Environment theme collection

Seed topics come from three sources:

1. **MCP Servers (~2.8K).** Real-world MCP server specifications from Smithery, each accompanied by structured JSON documenting source data and standardized tool definitions. This is the single biggest source and directly leverages the [07-model-context-protocol.md](07-model-context-protocol.md) ecosystem maturing in 2026 — MCP has become the de-facto interface standard, and its server registry has become a treasure trove of environment specs.
2. **Tool documentation (~0.5K).** Open-source datasets covering real tool-use scenarios are filtered; an LLM reverse-maps extracted tool definitions back to environment topics.
3. **Industrial PRDs (~0.2K).** Product requirement documents for specific industries. These encode domain workflows and system interfaces that do not show up in MCP registries — enterprise-shaped topics that other sources undersample.

The three source types are complementary: MCP is the broadest, tool docs fill technical-domain gaps, PRDs cover industry-specific workflows.

### Agentic database mining

For each topic, a deep-research agent with a standard agentic toolset (search, browser, code compiler, OS tools) conducts iterative retrieval loops. The policy \(π_θ\) drives information gathering; OS tools structure and persist results. The raw mined database is then *complexified* by iterative rounds of the same research agent adding richer content:

\[ D^{(n+1)}(m) = \phi(D^{(n)}(m), m, \mathcal{T}), \quad n = 0, \ldots, N-1 \]

This matters because **a single mining pass typically yields simple, narrow databases**. Multiple rounds of complexification let the database grow to match realistic environment demands — roughly 10-20× the raw starting point. This is itself a use of agentic reasoning *to produce training environments for future agents*, a kind of meta-recursion that characterizes much of the 2026 research frontier.

### Tool interface generation and verification

A separate coding agent \(\psi\) generates executable Python tool implementations from the mined database, along with unit tests. Each candidate tool \(\hat f\) is paired with a test set \(\hat{\mathcal{C}}_{\hat f}\). A tool survives filtering only if:

- it compiles in Python;
- its unit-test pass rate \(\text{Acc}(\hat f; \hat{\mathcal{C}}_{\hat f}) > 0.5\);
- its environment has at least one valid tool and one valid test.

This execution-grounded filter is the critical quality gate — it rejects hallucinated tools, broken implementations, and environments too weak for training. Combined with the taxonomy (next section), it produces a high-signal ecosystem \(\mathcal{E} = \{(D^{(N)}(m), F(m)) \mid m \in \mathcal{M}\}\).

### Hierarchical taxonomy

A three-tier taxonomy organizes the 1,978 environments:

- **Tier 1 (20 labels):** coarse categories — productivity, finance, healthcare, media, developer tools, etc.
- **Tier 2 (50 labels):** mid-grained subcategories via hierarchical clustering of environment themes, refined via GPT-OSS-120B summarization.
- **Tier 3 (2,000+ labels):** fine-grained environment-specific labels.

Human annotators merge second-tier labels into the 20 first-tier types through cross-validation. This matters for two reasons: it enables **stratified sampling** (stratified both for training and evaluation), and it enables the self-evolving arena (§ Component 2) to diagnose capability gaps *at the taxonomic level* ("the agent struggles across tier-1 category 'e-commerce'") rather than at individual-environment level.

## Verifiable task synthesis — two complementary strategies

The mined environments need tasks to train on. Agent-World generates tasks via two complementary synthesis methods.

### Strategy 1 — Graph-Based Task Synthesis (sequential tool dependencies)

For each environment, a **directed weighted tool graph** is constructed with three edge types (LLM-assigned):

- **Strong dependency** (weight 3): Tool B strictly requires Tool A's output (e.g., `create_order → get_order_details`).
- **Weak dependency** (weight 2): B *could* use A's output but has alternatives.
- **Independent** (weight 1): No parameter-level dependency; included to guarantee graph connectivity.

A **random walk** on the graph generates a raw tool sequence; the walk prefers starting nodes that return outputs without strong precursors, then samples successors weighted by edge type. Once a sequence is produced:

- Strong/weak edges pipe actual outputs forward.
- Independent edges sample fresh values from the database.
- An LLM prunes redundancies and verifies logical consistency, producing a refined executable sequence \(\tau^*\).

A language model drafts a task description — *forbidden from mentioning tool names or schema details* (anti-leakage) — then executes \(\tau^*\) in a Python sandbox, observing actual returned fields. The LLM then refines the draft into a realistic final query \(q_{final}\) and emits (i) a JSON ground-truth answer and (ii) structured evaluation rubrics for automated scoring.

### Strategy 2 — Programmatic Task Synthesis (non-linear reasoning)

Graph-based synthesis handles sequential dependencies but not conditionals, loops, or aggregations. For those, programmatic synthesis generates an entire executable Python script \(\pi_{code}\) that uses tools in complex control flow (for-loops, if-else, statistical aggregations) to solve the task. The script is debugged via a ReAct loop until it runs successfully. A separate verification script \(V_{code}(a, a^*)\) is generated and debugged — it contains multi-level assertions that check both the candidate answer and the resulting database state.

### Verification discipline — why this is robust

Both synthesis strategies share the same **quality consistency protocol**: a ReAct agent attempts each generated task 5 times in the sandbox; the task is kept only if the agent achieves ≥ 2 successful runs. This filters out:

- Tasks that are *too easy* (trivial, 5/5 solved) — not useful for training.
- Tasks that are *too hard* (0/5 solved) — likely broken / misspecified.
- Tasks that are *ambiguous* (the agent finds different answers on different runs) — unverifiable.

The 2-out-of-5 criterion is a specific calibration: hard enough to drive learning, but reachable enough to provide training signal. It's a pragmatic answer to a question most agent-RL papers handwave away.

### Difficulty scaling — explicit levers

Both strategies expose explicit difficulty knobs:

- **Graph-based:** increase random-walk max step count (longer tool chains); increase weak/independent-edge sampling probability (less obvious data flow); rewrite task descriptions to obscure tool-name and execution-logic hints.
- **Programmatic:** more unique tools, richer inter-tool logic (conditional branches, cross-database aggregations, sorting, filtering); description rewriting as above.

The key design insight: **difficulty is a controllable dimension, not an emergent property**. The pipeline can produce an easy curriculum early and ramp up complexity as training progresses — enabling the curriculum learning that §Component 2 depends on.

## Component 2 — Continuous Self-Evolving Agent Training

The environment ecosystem is not just a static training source. It's also a **dynamic diagnostic arena** that drives curriculum evolution. Two training modes interleave.

### Multi-environment RL (the inner loop)

Tasks are sampled across the ecosystem, and the policy \(π_θ\) interacts with a closed-loop of three components:

- **Policy \(π_θ\):** emits natural-language reasoning + tool/action decisions.
- **Tool interface/runtime:** executes tools against the environment state, maintains DB connections and caches.
- **Database state \(D^{(N)}(m)\):** read/write substrate for tool calls; verifiable structured backbone.

Formally the interaction follows the POMDP model \((\mathcal{U}, \mathcal{S}, \mathcal{A}, \mathcal{O}, \mathcal{P})\) described in §2 of the paper, with state decomposed into environment state \(s^E\) and dialogue state \(s^H\), and actions partitioned into tool-use vs language-response. This is the formal grounding of what "multi-environment RL" means, and it's worth noting the paper explicitly treats the environment state as **latent** — never directly observed, only inferable through tool output. This forces the agent to maintain an *internal* model of environment state rather than reading it off an oracle, which is closer to real-world conditions than prior training setups.

Tasks within a global batch are **paired with independent, dynamically sampled environments** — multi-environment rollout by construction.

### Structured verifiable rewards

Reward assignment distinguishes the two task types:

- **Graph-based tasks (\(\mathcal{X}_{graph}\)):** rubric-conditioned LLM-as-judge evaluates each criterion \(r_j \in R\); overall reward is the average pass indicator. Concrete example: schema matching rubric + fact-checking rubric.
- **Programmatic tasks (\(\mathcal{X}_{prog}\)):** execute the verification script \(V_{code}\) in the sandbox; binary pass/fail reward.

\[
r(x,y) = \begin{cases} \mathbf{1}\left[\frac{1}{n}\sum_{j=1}^{n}\mathbf{1}[\text{Judge}(x,y,r_j)] = 1\right] & \text{if } x \in \mathcal{X}_{graph} \\ \mathbf{1}[\text{Execute}(V_{code}(y, y^*))] & \text{if } x \in \mathcal{X}_{prog} \end{cases}
\]

This hybrid reward model is interesting. Programmatic tasks get **deterministic, execution-verified rewards** — the gold standard. Graph-based tasks get **LLM-judge-weighted rubrics** — necessarily noisier but cheaper to generate and more flexible about what "success" means across heterogeneous task types. The paper does not directly compare the learning efficiency of the two reward types, but having both is clearly necessary to cover the compositional space.

### GRPO policy update

As with Atomic Skills ([68](68-atomic-skills-scaling-coding-agents.md)), the optimizer is **Group Relative Policy Optimization**:

\[ J_{GRPO}(\theta) = \mathbb{E}\left[\frac{1}{G}\sum_{i=1}^{G} \frac{1}{|y_i|} \sum_{t=1}^{|y_i|} \min\!\left(r_{i,t}(\theta) \hat A_{i,t}, \mathrm{clip}(r_{i,t}(\theta), 1-\epsilon, 1+\epsilon) \hat A_{i,t}\right) - \beta D_{KL}(\pi_\theta \| \pi_{ref})\right] \]

Token-level advantages, group-normalized rewards, clipped importance ratios, KL penalty against a reference policy. Standard — but the reference policy here is the *previous-round* Agent-World policy (§next section). The KL term therefore acts as a stability constraint across evolution rounds, not just within a single training run.

### Self-Evolving Agent Arena (the outer loop)

This is the paper's signature contribution — the environment ecosystem as an *automated curriculum engine*. The loop has four phases:

1. **Arena construction.** Stratified sampling of K=5 environments from each tier-1 category yields an evaluation arena \(\mathcal{E}_{arena}\) that spans the taxonomy at controlled cost.
2. **Dynamic evaluation task synthesis.** At iteration \(r\), for each arena environment, a *fresh* batch of verifiable tasks is synthesized via §Component 1 — preventing overfitting to a static evaluation set and enabling continual diagnosis. This is the critical anti-Goodhart discipline: *the evaluation tasks refresh every round*.
3. **Agentic diagnosis.** Given a trained policy \(π_θ^{(r)}\), an auto-diagnosis agent \(\delta\) (equipped with Python interpreter + search tools) analyzes failure patterns from (i) per-task failure traces, (ii) error distribution statistics by environment and taxonomy category, (iii) environment metadata. It outputs (a) a ranked set of **weak environments** \(\mathcal{W}^{(r)}\), (b) **environment-specific task-generation guidelines** \(\mathcal{G}^{(r)}_{guide}(m)\) that characterize missing capabilities (e.g., "erroneous tool use," "state-update mistakes").
4. **Agent-environment co-evolution.** Conditioned on \((\mathcal{W}^{(r)}, \mathcal{G}^{(r)}_{guide})\), the verifiable task synthesis pipeline re-runs to generate a **targeted training set** \(\mathcal{X}^{(r)}_{target}\). Optionally the environment is expanded via database complexification when weakness is due to insufficient state diversity. Continue RL on the augmented data to produce \(π_θ^{(r+1)}\). Iterate.

The loop formula:

\[ π_θ^{(r)} \xrightarrow{\text{evaluate}} \mathcal{W}^{(r)} \xrightarrow{\text{diagnose+target}} \mathcal{X}^{(r)}_{target} \xrightarrow{\text{continue RL}} π_θ^{(r+1)} \]

Three features distinguish this from prior curriculum methods:

- **The curriculum is agent-authored.** The diagnosis agent \(\delta\) examines failure evidence and emits guidelines. No hand-tuned lesson plan.
- **The environment can expand too.** If the agent struggles *because the environment is too simple*, the environment itself gets complexified. Most curriculum methods hold the environment fixed; here they co-evolve.
- **Grounding through executable verification.** Every task — training, evaluation, diagnosis — runs in a sandbox with deterministic reward. No claim is made that cannot be executed.

## Empirical results — what the numbers actually say

The paper evaluates across **23 agent benchmarks** spanning agentic tool-use, advanced AI assistant, software engineering, deep research, and general reasoning. Key findings:

- **Agent-World-8B and 14B consistently outperform strong proprietary models and environment-scaling baselines.** The paper includes MCP-Mark, BFCL V4, and τ²-Bench results showing Agent-World on top for aggregate scores.
- **Clear scaling relationships** between (a) number of synthesized environments and downstream performance, and (b) number of self-evolution rounds and downstream performance. Both show monotonic but diminishing-returns curves — expected, but empirically demonstrating that *more environments* is a real training-scale axis in 2026.
- **Cross-benchmark generalization** is strong. Training-environment diversity does in fact transfer to evaluation-time diversity, which is the core claim of the "environment scale matters" hypothesis.

A structural point: *the scaling axis is environments, not parameters*. The evaluation includes 8B and 14B Agent-World variants, and the 14B is better — but both beat much larger proprietary baselines. This is consistent with the 2026 thesis that **at the agentic-RL stage, training-data scale (environments × tasks × evolution rounds) dominates parameter scale**. You cannot compensate for narrow environments by scaling parameters, but you can compensate for fewer parameters by scaling environments.

## Connections to the corpus

- **[07-model-context-protocol.md](07-model-context-protocol.md).** Agent-World depends on MCP's success — Smithery provides ~2.8K of the ~3.5K environment seeds. This is the clearest published payoff of MCP's standardization: the registry *is* a training-data source for the next generation of agents. MCP now earns its keep not just in deployment but in environment bootstrapping.
- **[68-atomic-skills-scaling-coding-agents.md](68-atomic-skills-scaling-coding-agents.md).** Complementary. Atomic Skills structures *what the policy learns* (5 basis vectors); Agent-World scales *where it learns* (1,978 environments). A team implementing both would train a single policy with atomic-skill reward functions on Agent-World's environment ecosystem — probably the current best-in-class training recipe.
- **[11-verifier-evaluator-loops.md](11-verifier-evaluator-loops.md) & [21-llm-as-judge-trajectory-eval.md](21-llm-as-judge-trajectory-eval.md).** The rubric-based LLM-judge reward for graph-based tasks is an industrialized version of these patterns, applied at training scale (millions of tasks) not just evaluation scale.
- **[34-clawbench-live-web-tasks.md](34-clawbench-live-web-tasks.md), [38-claw-eval.md](38-claw-eval.md).** Agent-World's 1,978 environments extend the realism thread of these benchmarks but target *training* not evaluation. The papers are complementary: ClawBench/ClawEval tell you if an agent generalizes to real sites; Agent-World trains agents that plausibly will.
- **[27-horizon-long-horizon-degradation.md](27-horizon-long-horizon-degradation.md).** The 20+ turn average task length is squarely in long-horizon territory. Training explicitly on long-horizon tasks (rather than hoping short-horizon training transfers) is a direct response to the degradation evidence. Agent-World does not *solve* long-horizon reasoning but shows what scaling in that direction looks like.
- **[36-autogenesis-self-evolving-agents.md](36-autogenesis-self-evolving-agents.md), [45-hyperagents-self-modification.md](45-hyperagents-self-modification.md).** Agent-World is a self-evolving *training* system; Autogenesis and Hyperagents are self-evolving *deployed* systems. The underlying idea — the agent participates in producing its own next version — is the same; the scope differs.
- **[47-adaptation-of-agentic-ai-survey.md](47-adaptation-of-agentic-ai-survey.md).** The survey's four-paradigm taxonomy (post-training, memory, skills, environment) would label Agent-World as the clearest environment-paradigm exemplar to date. Where the survey mostly speculated about the environment dimension, Agent-World publishes a working instance.
- **[60-sea-top-github-repos.md](60-sea-top-github-repos.md) & [66-meta-harness-landscape.md](66-meta-harness-landscape.md).** Agent-World fits into the self-evolving-agent trend these files track, but is **more prescriptive about the training side**. Where most of the SEA literature focuses on the self-modification loop, Agent-World argues the bottleneck is *environments to run the loop in*.
- **[31-glm-5-agentic-engineering.md](31-glm-5-agentic-engineering.md).** GLM-5 and Agent-World together define the "frontier agentic-RL training stack" as of 2026: long-horizon trajectories (GLM-5) + scalable environment ecosystem (Agent-World) + atomic-skill reward shaping ([68](68-atomic-skills-scaling-coding-agents.md)) + closed evolution loops (Agent-World + [36](36-autogenesis-self-evolving-agents.md)).

## Limitations, failure modes, and open questions

1. **LLM-judge reliability.** Graph-based tasks use rubric-conditioned LLM-as-judge. While rubrics constrain the judge, judge accuracy is never a zero. The paper does not report an independent calibration study of how often the judge's graph-task reward agrees with programmatic verification on tasks amenable to both. This is the single most important missing measurement.
2. **Environment bias toward MCP.** 2.8K of ~3.5K seed themes come from MCP. If MCP servers are biased toward certain domains (developer tooling, productivity, web scraping), so is Agent-World. Industrial PRDs partially compensate but only 200 seeds cover the entire non-tech industrial world.
3. **Diagnosis-agent blind spots.** The auto-diagnosis agent \(\delta\) is itself an LLM. It sees failures and proposes guidelines, but it can only flag patterns visible in the failure traces. Subtle correlated failures (memory leaks across long trajectories, drift in implicit state estimation) may evade it.
4. **Evolution-round cost.** Each round runs the full synthesis-diagnose-train pipeline. The paper does not state total cost, but reasonable estimates put a multi-round Agent-World training in the $1M-$5M range for compute. Reproducibility for academic labs is tight.
5. **Environment staleness.** Mined databases reflect the web at the time of mining. A banking environment trained in January 2026 is stale if banking APIs change in March. The paper does not address refresh cadence or how to detect when an environment has drifted from reality.
6. **Reward-hacking on graph tasks.** The anti-leakage description rewriting ("no tool names or schema details") is a real defense, but an agent that learns to pattern-match on task wording may still exploit distributional cues that survive the rewrite. A separate robustness study would be reassuring.
7. **Scaling ceiling.** The paper shows monotonic scaling with environments and rounds but does not identify where it plateaus. At some point, more environments plateau (the agent has saturated the "general environment interaction" skill); at some point, rounds plateau (the diagnosis agent cannot find more weaknesses). Knowing where these ceilings sit would be directly useful for planning training budgets.

## Takeaways — practical implications for harness engineering

1. **Environment diversity is a first-class training axis.** Not an afterthought. Any training program that uses < 100 environments is leaving capability on the table. The 2026 bar is in the thousands.
2. **MCP as environment substrate.** If you are building training infrastructure, the MCP server registry is your highest-leverage starting point. Scrape it, normalize it, run quality filters, you have half the battle solved.
3. **Hierarchical taxonomy as a strategic asset.** Once environments are organized into tiers, stratified sampling is easy, diagnosis is structured, and evolution loops can target specific gaps. Without a taxonomy, all evaluation failures look like "the agent is bad at something." With one, you get "the agent is weak at tier-1 category 'healthcare'."
4. **Co-evolution, not curriculum.** Curriculum implies a fixed teacher. Co-evolution lets the environment improve as the agent does. It's the right framing for continually-deployed agents where the world changes out from under them anyway.
5. **Dual reward model: execution when possible, LLM-judge when necessary.** Programmatic tasks get executable validators; graph tasks get rubric-conditioned judges. This is the right compromise — execution reward is preferable but not always available.
6. **Diagnosis as an agent.** Using an agentic diagnosis system instead of hand-authored metrics scales better and surfaces patterns humans miss. The Gnomon proposal ([67-recommended-breakthrough-project.md](67-recommended-breakthrough-project.md)) points in the same direction — primitive-level failure attribution at inference time.
7. **Environments are cheaper than parameters, now.** A 14B model + 1,978 environments + 23-benchmark generalization beats much larger proprietary models. If your training budget is finite, spend it on environments before parameters.
8. **The "train on MCP + atomic skills" recipe.** Combining [68 Atomic Skills](68-atomic-skills-scaling-coding-agents.md)'s reward-function decomposition with Agent-World's environment ecosystem is the most concrete recipe for next-generation agent training visible in April 2026. Expect this exact combination to be the default in open research within 6-12 months.

## Production-readiness assessment

- **Green.** The environment-mining pipeline is independently reusable. Any team with compute and an LLM can follow the three-source seeding + database-complexification + tool-verification recipe to produce a domain-specific environment ecosystem.
- **Green.** The dual task-synthesis strategy (graph-based + programmatic) is straightforward to replicate and directly useful even outside Agent-World's RL setting — e.g., as an evaluation generator.
- **Green.** The agentic-diagnosis agent pattern is lift-and-shift into any evaluation harness that produces execution traces.
- **Amber.** Reproducing Agent-World-14B's full training requires ByteDance-scale compute. The paper's project page (https://agent-tars-world.github.io/) promises more, but as of preprint no full model weights are publicly released.
- **Amber.** The LLM-judge calibration story is thin. Teams using rubric-conditioned judges in production should run their own calibration before relying on the rewards.
- **Red.** No code release with preprint v1 (2026-04-20). Watch the project page for updates.

## References

- **Primary.** Dong et al. (Renmin University of China × ByteDance Seed). *Agent-World: Scaling Real-World Environment Synthesis for Evolving General Agent Intelligence*. arXiv:[2604.18292](https://arxiv.org/abs/2604.18292) [cs.AI], v1, 20 Apr 2026. 21.8 MB, ~30+ pages. Correspondence: Guanting Dong (dongguanting@ruc.edu.cn), Zhicheng Dou (dou@ruc.edu.cn). Project page: [agent-tars-world.github.io](https://agent-tars-world.github.io/).
- **Related methodology / baselines.**
  - Shao et al. 2024 — GRPO optimizer (shared with [68](68-atomic-skills-scaling-coding-agents.md)).
  - TOUCAN taxonomy (referenced for hierarchical clustering).
  - Doubao-Seed-2.0-pro — used as the Pass@10 difficulty-calibration model.
  - GPT-OSS-120B — used as supervised summarization for cluster labels.
- **Benchmarks evaluated on.**
  - MCP-Mark, BFCL V4, τ²-Bench (the three "environment scaling" baselines reported in Fig. 1).
  - + 20 others across agentic tool-use, AI assistant, SE, deep research, general reasoning.
- **Ecosystem inputs.**
  - Smithery MCP server registry ([smithery.ai/servers](https://smithery.ai/servers)).

## Cross-references in this corpus

- [07-model-context-protocol.md](07-model-context-protocol.md) — the MCP ecosystem Agent-World mines.
- [11-verifier-evaluator-loops.md](11-verifier-evaluator-loops.md) — verification patterns underlying rewards.
- [21-llm-as-judge-trajectory-eval.md](21-llm-as-judge-trajectory-eval.md) — judge-based evaluation.
- [27-horizon-long-horizon-degradation.md](27-horizon-long-horizon-degradation.md) — long-horizon training response.
- [31-glm-5-agentic-engineering.md](31-glm-5-agentic-engineering.md) — complementary agentic-RL stack.
- [34-clawbench-live-web-tasks.md](34-clawbench-live-web-tasks.md) — evaluation-time realism counterpart.
- [36-autogenesis-self-evolving-agents.md](36-autogenesis-self-evolving-agents.md) — self-evolving paradigm.
- [38-claw-eval.md](38-claw-eval.md) — trajectory-aware evaluation.
- [45-hyperagents-self-modification.md](45-hyperagents-self-modification.md) — self-modification counterpart.
- [47-adaptation-of-agentic-ai-survey.md](47-adaptation-of-agentic-ai-survey.md) — environment-paradigm taxonomy.
- [60-sea-top-github-repos.md](60-sea-top-github-repos.md) — SEA landscape.
- [66-meta-harness-landscape.md](66-meta-harness-landscape.md) — meta-harness synthesis.
- [67-recommended-breakthrough-project.md](67-recommended-breakthrough-project.md) — Gnomon proposal.
- [68-atomic-skills-scaling-coding-agents.md](68-atomic-skills-scaling-coding-agents.md) — complementary training paradigm.
