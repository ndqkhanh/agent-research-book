# 225 — Scaling Laws for the Agent Era: a synthesis (pretrain × TTC × trajectory × multi-agent)

**Synthesis of:** [216-pretraining-scaling-laws-foundation](216-pretraining-scaling-laws-foundation.md), [217-test-time-compute-scaling](217-test-time-compute-scaling.md), [237-agent-trajectory-scaling](237-agent-trajectory-scaling.md), [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md), [224-multi-agent-parallel-scaling](224-multi-agent-parallel-scaling.md). Cross-references: [01-agent-loop-architecture](01-agent-loop-architecture.md), [02-subagent-delegation](02-subagent-delegation.md), [04-skills](04-skills.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [15-tree-of-thoughts-lats](15-tree-of-thoughts-lats.md), [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md), [68-atomic-skills-scaling-coding-agents](68-atomic-skills-scaling-coding-agents.md), [79-skill-rag](79-skill-rag.md), [80-knowrl](80-knowrl.md), [81-reasoningbank](81-reasoningbank.md), [84-swe-search-mcts](84-swe-search-mcts.md), [86-frugalgpt](86-frugalgpt.md), [87-routellm](87-routellm.md), [88-confidence-driven-router](88-confidence-driven-router.md), [89-voyager-deep](89-voyager-deep.md), [90-reflexion-deep](90-reflexion-deep.md), [91-metagpt-deep](91-metagpt-deep.md), [92-chatdev](92-chatdev.md), [94-eagle3-spec-decoding](94-eagle3-spec-decoding.md), [95-osworld](95-osworld.md), [96-gdpval](96-gdpval.md), [97-qwen-prm](97-qwen-prm.md), [98-diversity-collapse-mas](98-diversity-collapse-mas.md), [99-papers-deep-dive-synthesis](99-papers-deep-dive-synthesis.md), [142-trajectory-simulation-agents](142-trajectory-simulation-agents.md), [156-heavyskill-rlvr](156-heavyskill-rlvr.md), [189-recursive-multi-agent-systems](189-recursive-multi-agent-systems.md), [195-argus-omega-vol-2-trajectory-temporal-horizon](195-argus-omega-vol-2-trajectory-temporal-horizon.md), [199-search-r1-multi-hop](199-search-r1-multi-hop.md), [202-multi-agent-multi-hop-reckoning-2026](202-multi-agent-multi-hop-reckoning-2026.md), [206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md), [211-cross-project-power-up-plan-with-tradeoffs](211-cross-project-power-up-plan-with-tradeoffs.md).

**One-line definition.** A unified four-axis frame for budgeting and architecting agentic systems — **pretraining-compute (Kaplan/Chinchilla, [216](216-pretraining-scaling-laws-foundation.md)) × test-time-compute (Snell/o1, [217](217-test-time-compute-scaling.md)) × trajectory-budget (SWE-agent/GAIA, [237](237-agent-trajectory-scaling.md)) × multi-agent-parallelism (MoA/debate, [224](224-multi-agent-parallel-scaling.md))** — with the **verifier ([223](223-verifier-and-best-of-n-scaling.md)) as the cross-cutting infrastructure** that determines how much capability each axis converts into usable answers.

## Why this synthesis matters (every harness decision is a choice between these four axes, and most are mis-allocated)

The deep-dive corpus 77–216 covers ~140 distinct techniques for building agent harnesses. Almost every one — agent loops, skills, hooks, verifiers, debate, reflection, ReAct, ReWOO, ReasoningBank, Voyager, MetaGPT, ChatDev, MoA, OSWorld, GAIA, MCTS, beam-search, lookahead, BoN, ORM, PRM, multi-hop, MEMTIER, fresh-context, sub-agents, worktrees — is, *under the hood*, a way to spend on one or more of four axes:

1. **Pretraining compute (axis A1).** What you bought when you picked the base model. Kaplan-Chinchilla curve. Sets the capability floor and unit economics. ([216](216-pretraining-scaling-laws-foundation.md))
2. **Test-time compute per call (axis A2).** Sequential revisions, parallel sampling, beam search, lookahead. Difficulty-conditional optimum. ([217](217-test-time-compute-scaling.md))
3. **Trajectory budget per task (axis A3).** Steps, tool calls, context length, ACI quality, memory. ([237](237-agent-trajectory-scaling.md))
4. **Multi-agent parallelism (axis A4).** K agents, aggregator, debate, role-specialization, recursion. ([224](224-multi-agent-parallel-scaling.md))

The **verifier ([223](223-verifier-and-best-of-n-scaling.md))** is not itself a scaling axis but the cross-cutting infrastructure that determines how much capability each axis *realizes*. A1's pretraining ceiling is invisible without verification; A2's BoN/beam search needs a PRM; A3's verifier-checkpoints catch coherence collapse; A4's aggregator is a verifier in disguise.

Most agent harnesses in production today over-invest in A1 (chasing the latest flagship) and A3 (raw step budget), under-invest in A2 (no difficulty router, no PRM-guided search), and either over-invest in A4 (multi-agent for the sake of headcount) or under-invest (no aggregator, no diversity audit). The *unit economics* of marginal investment heavily favour A2 + A4 + verifier infrastructure for most current production tasks; this synthesis is a decision matrix for that allocation.

Take this synthesis seriously and three things change in how you architect harnesses. **First**, you stop debating "which is the best model" and start debating "which axis has the steepest marginal-return at *our* operating point." **Second**, you build a measurement infrastructure ([16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md)) that *captures the curves on each axis* — without curves you cannot allocate. **Third**, you understand that the agent era's defining innovation is not "agents" but *the fact that capability now scales on multiple orthogonal axes simultaneously*, and that the harness's job is the cross-axis allocator.

## The four axes formalized

### A1 — Pretraining compute

- **Coordinate:** `(N, D, C)` where `N` = parameters, `D` = training tokens, `C ≈ 6ND` = total training FLOPs.
- **Curve:** `L(N, D) ≈ E + A·N^-α + B·D^-β` with `α ≈ 0.3`, `β ≈ 0.3` ([216](216-pretraining-scaling-laws-foundation.md), Hoffmann §3).
- **Compute-optimal allocation:** `D / N ≈ 20` (Chinchilla); production overshoot to D/N = 100–1000 for inference economics.
- **Marginal cost:** $$$$ — large, slow, infrastructure-bound.
- **Marginal return at 2026 frontier:** `α ≈ 0.3` exponent → halving the loss requires ~10× compute. Returns are real but slow.
- **Practitioner lever:** model selection. Pick the highest D/N you can afford for your budget.

### A2 — Test-time compute per call

- **Coordinate:** `(N_parallel, T_sequential, search-method)` where N is parallel samples, T is sequential revisions, and search-method ∈ {majority-vote, BoN, beam, lookahead}.
- **Curve:** `acc(B; difficulty)` is difficulty-conditional ([217](217-test-time-compute-scaling.md), Snell §4). Easy → high T, low N; hard → low T, high N. Substitution rate vs A1: ~1:1 at medium difficulty.
- **Marginal cost:** $$ per call — proportional to N · T · cost_per_token.
- **Marginal return at 2026 frontier:** steeper than A1 in the easy-medium difficulty band; flatter at hardest. PRM quality determines the realized exponent.
- **Practitioner lever:** difficulty router + PRM + search algorithm. Each prompt allocated `(N*, T*, method)` from its difficulty bucket.

### A3 — Trajectory budget per task

- **Coordinate:** `(max_steps, max_tokens, ACI, memory)`.
- **Curve:** `acc(M, H, ACI, T) ≈ acc_∞ · (1 − exp(−T / T_plateau))` ([237](237-agent-trajectory-scaling.md)). Three parameters: height (model + per-call TTC), location (ACI quality), slope (memory + verifier checkpoints).
- **Marginal cost:** $$$ per task — proportional to expected trajectory length × per-step cost.
- **Marginal return at 2026 frontier:** ACI is the steepest single lever; memory the second; raw step-count the third (saturates at plateau).
- **Practitioner lever:** custom ACI per domain shell, memory blocks, plateau-aware step cap.

### A4 — Multi-agent parallelism

- **Coordinate:** `(K, diversity, aggregator, structure)` where K is agent count, diversity is variance over priors, aggregator is the synthesis mechanism, structure ∈ {flat, debate, MoA, role-specialized, recursive}.
- **Curve:** `acc(K) ≈ acc_∞ · (1 − exp(−K · div / K_sat))` — saturates at K_sat ≈ 6–40 depending on diversity and task ([224](224-multi-agent-parallel-scaling.md)).
- **Marginal cost:** $$ per request — K parallel calls + aggregator call + coordination overhead.
- **Marginal return at 2026 frontier:** highest when diverse open-weights agents are available; collapses with diversity loss.
- **Practitioner lever:** ensemble selection, aggregator model choice, role specialization, debate protocol.

### V — Verifier infrastructure (cross-cutting)

- **Coordinate:** `(PRM-quality, ProcessBench-F1, prm@N, cross-channel-pair)`.
- **Curve:** verifier separation determines pass@N realization for A2 and A4; verifier checkpoints determine A3 slope.
- **Marginal cost:** $$$$ — training data, training compute, ongoing eval.
- **Marginal return at 2026 frontier:** very high — multiplier on every other axis.
- **Practitioner lever:** [97-qwen-prm](97-qwen-prm.md) recipe, cross-model adversarial pair ([02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md)), domain-specific PRMs, ProcessBench-tracked.

## How existing techniques map to the four axes

| Technique | Primary axis | Secondary axis | Notes |
|---|---|---|---|
| Picking GPT-4o vs Llama-3-70B vs DeepSeek-V3 | **A1** | — | First decision, sets floor |
| Reflexion ([90](90-reflexion-deep.md)) | **A2** | — | Sequential test-time compute |
| BoN with PRM ([223](223-verifier-and-best-of-n-scaling.md)) | **A2** | V | Parallel TTC + verifier |
| Tree-of-Thoughts / LATS ([15](15-tree-of-thoughts-lats.md), [p18](../projects/polaris/docs/research/p18-tree-search-thinking.md)) | **A2** | V | Parallel + sequential TTC |
| SWE-agent ACI ([237](237-agent-trajectory-scaling.md)) | **A3** | — | ACI design dominates |
| Domain shell ([11-domain-shell](../projects/polaris/docs/blocks/11-domain-shell.md)) | **A3** | A4 | ACI + role specialization |
| Skills ([04-skills](04-skills.md), [09-skill-engine](../projects/polaris/docs/blocks/09-skill-engine.md)) | A3 | — | Productivity per step |
| ReasoningBank ([81-reasoningbank](81-reasoningbank.md)) | **A3** | — | Memory raises slope |
| MEMTIER trilogy ([151-153]) | **A3** | — | Hierarchical memory |
| Plan mode ([03-plan-mode](03-plan-mode.md)) | **A2** | A3 | Sequential TTC; plans amortize trajectory |
| Subagent delegation ([02-subagent-delegation](02-subagent-delegation.md)) | **A3** | A4 | Trajectory split + parallelism |
| Worktree isolation ([18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md)) | **A3** | A4 | Same |
| Multi-agent debate ([224](224-multi-agent-parallel-scaling.md)) | **A4** | V | K agents + critique |
| MoA ([224](224-multi-agent-parallel-scaling.md)) | **A4** | V | K + aggregator |
| MetaGPT / ChatDev ([91](91-metagpt-deep.md), [92](92-chatdev.md)) | **A4** | A3 | Role-specialized + structured trajectory |
| RAG ([79-skill-rag](79-skill-rag.md), [128-135 RAG depth]) | **A3** | A2 | Retrieved context = trajectory state |
| Multi-hop reasoning ([199](199-search-r1-multi-hop.md), [202](202-multi-agent-multi-hop-reckoning-2026.md)) | **A2** | A3, A4 | Sequential TTC + multi-step trajectory |
| Verifier-evaluator loops ([11](11-verifier-evaluator-loops.md)) | **V** | A2, A3 | Verifier checkpoints |
| Cross-channel verifier ([19-verifier-cross-channel](../projects/polaris/docs/blocks/19-verifier-cross-channel.md)) | **V** | A4 | Adversarial diversity |
| HeavySkill ([156](156-heavyskill-rlvr.md), [p10](../projects/polaris/docs/research/p10-heavyskill-rlvr.md)) | **A2** | A4 | Parallel-then-deliberate inner skill |
| RLVR / RL-on-reasoning-traces ([80-knowrl](80-knowrl.md)) | **A1+A2** | V | Pushes A2 into the model |
| Recursive multi-agent ([189](189-recursive-multi-agent-systems.md)) | **A4** | A3 | Recursive depth on K |
| Trajectory simulation ([142](142-trajectory-simulation-agents.md), [195](195-argus-omega-vol-2-trajectory-temporal-horizon.md)) | A3 | V | Verify trajectories before deployment |
| Cost router ([86](86-frugalgpt.md), [87](87-routellm.md), [88](88-confidence-driven-router.md), [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md)) | **A1+A2** | A3 | Per-prompt model + budget |
| Speculative decoding ([94](94-eagle3-spec-decoding.md)) | meta-A2 | — | Reduces FLOPs/token, shifts curves |
| Daemon-mode agents ([13-daemon-and-scheduler](../projects/polaris/docs/blocks/13-daemon-and-scheduler.md), [p7-daemon](../projects/polaris/docs/research/p7-daemon.md)) | A3 wall-clock | — | Decouples wall-clock from FLOPs |
| Voyager ([89](89-voyager-deep.md)) | A3 | A2 | Skill memory + open-ended exploration |

## The decision matrix: where to spend the next dollar

The marginal-return ranking depends on the operating point. Three regimes cover most production scenarios.

### Regime 1: research / exploration / hard-tail tasks

You want maximum capability and can afford latency.

- **A1 (pretraining):** pick the best frontier model you can afford. Marginal: medium.
- **A2 (TTC):** large-K BoN + best-PRM + lookahead search. Marginal: **high**.
- **A3 (trajectory):** generous step budget; rich ACI; full memory stack. Marginal: high.
- **A4 (multi-agent):** MoA-style ensemble of diverse models; debate rounds; cross-model verifier. Marginal: **high**.
- **V (verifier):** invest hard. Domain-specific PRM, ProcessBench-tracked. Marginal: **highest**.

Recommended order of investment: V → A2 → A4 → A3 → A1.

### Regime 2: production agents / latency-tolerant / cost-sensitive

You want best capability per dollar at moderate latency.

- **A1:** Chinchilla-overshot open-weights model (e.g. Llama-3-70B at D/N=215, Qwen-2.5-72B). Marginal: low — the model is already good.
- **A2:** difficulty-aware router; BoN-8 with PRM on medium-hard prompts; single-shot on easy. Marginal: **high**.
- **A3:** custom ACI per task; memory blocks; plateau-aware step cap. Marginal: **high** for ACI, medium for memory.
- **A4:** MoA-Lite (3 agents + aggregator) for hardest tasks; flat single-agent for typical. Marginal: medium.
- **V:** trained PRM for the dominant task family; cross-channel pair for safety. Marginal: high.

Recommended order: A3 (ACI) → A2 (router + PRM) → V → A4 (selectively) → A1 (last).

### Regime 3: latency-bound / interactive / chat

You want sub-second response.

- **A1:** smaller, faster model. Distilled or speculative-decoded ([94-eagle3-spec-decoding](94-eagle3-spec-decoding.md)). Marginal: high.
- **A2:** single-shot with optional 1 sequential revision; no parallel BoN. Marginal: low.
- **A3:** short trajectory; tight ACI; cached memory. Marginal: medium.
- **A4:** none — too much latency. Marginal: zero.
- **V:** lightweight self-consistency or single-pass critique. Marginal: low.

Recommended order: A1 (smaller + spec-decoding) → A3 (cached memory + tight ACI) → A2 (revision) → others as backstop.

## Cross-axis interactions

The axes are not independent; several interactions matter.

- **A1 × A2:** the better the base model, the steeper the per-call TTC curve (verifier and revision quality both depend on base model). At the very hardest difficulty bucket, A1 dominates A2.
- **A1 × A3:** the trajectory plateau height equals base-model ceiling at infinite trajectory; pretraining priors directly cap A3.
- **A2 × A3:** TTC at each agent step compounds with trajectory length; an agent that thinks longer per step needs fewer steps. Reflexion is exactly this.
- **A2 × A4:** the MoA-2 layer aggregator is a per-call TTC step. K agents at A4 substitute for some of A2's parallel sampling.
- **A3 × A4:** subagent delegation splits one A3 trajectory into K parallel A4 trajectories; coordination overhead is the cost.
- **V × {A2, A3, A4}:** verifier quality bounds realization on all three. A weak PRM caps A2's BoN; a weak verifier-checkpoint caps A3's slope; a weak aggregator caps A4's saturation.
- **A2 × A4 × V together = MCTS / LATS / SWE-Search:** tree search is the formal unification — the verifier provides value estimates, the search expansion provides parallelism, the rollouts provide depth. ([84-swe-search-mcts](84-swe-search-mcts.md), [p18-tree-search-thinking](../projects/polaris/docs/research/p18-tree-search-thinking.md), [15-tree-of-thoughts-lats](15-tree-of-thoughts-lats.md)).

## Empirical: where the agent-era frontier moved 2024 → 2026

| Axis | 2024 frontier | 2026 frontier | Mechanism |
|---|---|---|---|
| A1 | GPT-4 (1.8T params est., undertrained) | Llama-4 / Claude 3.7 / DeepSeek-V3 (overshot D/N) | Chinchilla-aware retraining |
| A2 | Reflexion + BoN-8 + ORM | o1 / R1 / extended-thinking with RL-learned allocation | RL on reasoning traces |
| A3 | SWE-agent custom ACI, ~50 steps, basic memory | OSWorld + MEMTIER + worktrees + 200+ steps | Memory + ACI + subagent maturity |
| A4 | Single-agent dominant; debate experimental | MoA, role-specialized teams, recursive multi-agent | Diversity audit + aggregator quality |
| V | ORMs common; PRMs experimental | PRMs standard; cross-channel verifier mandatory | Qwen-PRM recipe; ProcessBench |

The METR time-horizon doubled-every-7-months trend is, at the system level, the integral of progress on all four axes plus V. A single-axis story cannot explain it.

## Failure modes of single-axis thinking

- **"Just buy a bigger model" fallacy.** Maxes A1, ignores A2/A3/A4. Misses ~5–10× capability on most tasks at higher cost.
- **"More steps will solve it" fallacy.** Maxes A3, ignores plateau. Wastes budget past T_plateau.
- **"Add more agents" fallacy.** Maxes A4, ignores diversity. Diversity collapse erases gains; coordination overhead dominates.
- **"Just sample more" fallacy.** Maxes A2 N, ignores PRM quality. pass@N rises but realized accuracy stalls.
- **"Skip the verifier" fallacy.** Treats V as optional. All other axes under-realize.
- **Single-axis micro-optimization.** Tuning ACI for SWE-bench while leaving PRM untouched is a 5 % gain when the verifier upgrade is a 15 % gain.

## When the four-axis frame breaks

- **Tasks without verifiable correctness.** Open-ended creative work, conversation, advisory. V is unavailable; A2 and A4 lose much of their leverage.
- **Multi-modal tasks.** Vision-language, speech, video — the axes still apply but constants and exponents shift; per-modality re-fitting needed.
- **Adversarial environments.** Red-team / security / agentic-attack settings — the axes describe capability, not robustness; orthogonal threat-modeling required.
- **Long-running daemons.** Wall-clock economics decouple from FLOPs; A3 measured in days not steps.
- **Privacy / local-first deployments.** A1 capped to small local models; A4 capped to local ensembles; alters the recommended-order ranking.

## Implications for harness engineering (the synthesis)

- **Build the measurement substrate first.** Without HIR observability ([16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md), [p13-cost-discipline](../projects/polaris/docs/research/p13-cost-discipline.md)) you cannot fit any curve and cannot allocate.
- **Difficulty router is the central allocator.** [88-confidence-driven-router](88-confidence-driven-router.md), [87-routellm](87-routellm.md), [86-frugalgpt](86-frugalgpt.md) — its quality determines whether per-prompt allocation is correct.
- **Verifier infrastructure is the multiplier.** [97-qwen-prm](97-qwen-prm.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [19-verifier-cross-channel](../projects/polaris/docs/blocks/19-verifier-cross-channel.md), [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md) — invest before A2 / A3 / A4.
- **Custom ACI per domain shell is the highest-ROI A3 lever.** [11-domain-shell](../projects/polaris/docs/blocks/11-domain-shell.md), [17-mcp-adapter](../projects/polaris/docs/blocks/17-mcp-adapter.md), [SWE-agent](237-agent-trajectory-scaling.md).
- **Memory raises trajectory slope.** [05-reasoning-bank](../projects/polaris/docs/blocks/05-reasoning-bank.md), [81-reasoningbank](81-reasoningbank.md), [151-153 MEMTIER] — without it, A3 saturates early.
- **Subagents are A3 × A4 hybrid.** [02-subagent-delegation](02-subagent-delegation.md), [18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md), [189-recursive-multi-agent-systems](189-recursive-multi-agent-systems.md) — split trajectory into K parallel sub-trajectories.
- **Tree-search is A2+A4+V unified.** [15-tree-of-thoughts-lats](15-tree-of-thoughts-lats.md), [84-swe-search-mcts](84-swe-search-mcts.md), [p18-tree-search-thinking](../projects/polaris/docs/research/p18-tree-search-thinking.md), [156-heavyskill-rlvr](156-heavyskill-rlvr.md).
- **Multi-agent demands diversity audit.** [98-diversity-collapse-mas](98-diversity-collapse-mas.md), [224](224-multi-agent-parallel-scaling.md) — measure before scaling K.
- **Role-specialized organizations for multi-step workflows.** [91-metagpt-deep](91-metagpt-deep.md), [92-chatdev](92-chatdev.md), [205-lobehub-collaborative-teammate-platform](205-lobehub-collaborative-teammate-platform.md), [206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md).
- **RL on reasoning traces internalizes A2.** [80-knowrl](80-knowrl.md), [156-heavyskill-rlvr](156-heavyskill-rlvr.md) — push the difficulty-aware allocation into the model itself.
- **Plan mode + verifier-checkpoints raise A3 slope.** [03-plan-mode](03-plan-mode.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md).
- **Cost router operates across axes.** [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md), [86](86-frugalgpt.md), [87](87-routellm.md) — per-prompt (model, N, T, K) tuple from observed curves.
- **Bright-line gates respect plateaus.** [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md) — terminate at plateau, escalate at violation.
- **Daemon mode shifts wall-clock economics.** [13-daemon-and-scheduler](../projects/polaris/docs/blocks/13-daemon-and-scheduler.md), [p7-daemon](../projects/polaris/docs/research/p7-daemon.md) — A3 measured in long horizons; A1 stays cheap.
- **Trajectory simulation reduces deployment risk.** [142-trajectory-simulation-agents](142-trajectory-simulation-agents.md), [195-argus-omega-vol-2-trajectory-temporal-horizon](195-argus-omega-vol-2-trajectory-temporal-horizon.md) — verify trajectories offline before paying for real-environment compute.
- **Cross-project plans must allocate across axes.** [203-polaris-multi-hop-reasoning-apply-plan](203-polaris-multi-hop-reasoning-apply-plan.md), [207-cross-project-collaborative-multi-hop-apply-plan](207-cross-project-collaborative-multi-hop-apply-plan.md), [208-lyra-multi-hop-collaborative-apply-plan](208-lyra-multi-hop-collaborative-apply-plan.md), [211-cross-project-power-up-plan-with-tradeoffs](211-cross-project-power-up-plan-with-tradeoffs.md) — every per-project plan should be auditable against the four axes.

## The strategic claim

The agent era is defined not by "agents" as an architecture but by the **multiplication of orthogonal scaling axes**. Pretraining was the only axis pre-2024; in 2026 there are four (plus verifier infrastructure as the multiplier). A harness that allocates well across all five outperforms one that maxes any single axis, often by an order of magnitude in capability per dollar. The single most predictive question for any harness's trajectory: **does the team have curves for each axis, and do they revise the allocation as the curves shift?**

Frontier model labs (OpenAI, Anthropic, DeepMind, DeepSeek) are productizing this implicitly — o1/o3 fold A2 into A1 via RL, "thinking models" fold A2+V into the model, agentic-finetune folds A3 into the model, and reasoning-team builders fold A4 into the model. Each productization shifts the harness's marginal-return curve: when A2 is internal, the harness invests less in BoN and more in difficulty routing; when A3 is internal, the harness invests less in step caps and more in tool quality.

The harness's job becomes meta-allocator: which axes are *internal to the model this quarter* vs *external to be supplied by the harness*. This is a moving target and is the fundamental engineering discipline of the agent era.

**One-line takeaway for harness designers.** **The agent era's defining feature is that capability now scales on four orthogonal axes — pretraining, test-time-compute, trajectory, multi-agent — multiplied by a verifier; the harness's job is to *measure* curves on each axis, *allocate* per-prompt across them, and *re-allocate* as the model itself absorbs axes via RL — single-axis thinking is the dominant failure mode of 2024–2026 harness engineering, and a disciplined cross-axis allocator is the largest unrealized capability lever in production today.**
