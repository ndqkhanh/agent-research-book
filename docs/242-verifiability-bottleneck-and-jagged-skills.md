# 242 — Verifiability as the Bottleneck: why jagged skills track verifier availability, not raw capability

**Anchors.** Andrej Karpathy — IICYMI talk, May 2026, chapter *Verifiability and Jagged Skills* (09:41) — `https://www.youtube.com/watch?v=96jN2OCOfLs`. DeepSeek-AI — *DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via Reinforcement Learning* — arXiv:2501.12948 — Jan 2025. OpenAI — *o1 system card and learning-to-reason* — Sep 2024. Hunter Lightman et al. — *Let's Verify Step by Step* — arXiv:2305.20050 — OpenAI 2023. Karl Cobbe et al. — *Training Verifiers to Solve Math Word Problems* — arXiv:2110.14168 — OpenAI 2021. Charlie Snell et al. — *Scaling LLM Test-Time Compute Optimally Can Be More Effective than Scaling Model Parameters* — arXiv:2408.03314 — 2024. Brown et al. — *Large Language Monkeys: Scaling Inference Compute with Repeated Sampling* — arXiv:2407.21787 — 2024. AI2 *Tülu 3* — RLVR pipeline, 2024–2025. Companion: [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [80-knowrl](80-knowrl.md), [97-qwen-prm](97-qwen-prm.md), [97](97-qwen-prm.md), [115](115-evaluating-llm-systems.md), [217](217-test-time-compute-scaling.md), [223](223-verifier-and-best-of-n-scaling.md), [232](232-rl-on-reasoning-traces-scaling.md). High-star repos with verifier-driven workflows: princeton-nlp/**SWE-agent**, All-Hands-AI/**OpenHands**, Aider-AI/**aider** (test-suite gating), Pythagora-io/**gpt-pilot** (debugger loop), **bolt.new** (preview-driven verification), DeepSeek/**DeepSeek-R1** (RLVR public release).

**One-line definition.** **Verifiability** — the per-task availability of a *cheap, reliable, value-aligned* check on agent output — is the *load-bearing axis* on the agent era's progress curve: where verifiers exist, RL ([232](232-rl-on-reasoning-traces-scaling.md)) and Best-of-N ([223](223-verifier-and-best-of-n-scaling.md)) compound and tasks fall to automation; where they do not, the jagged-skill ceiling holds, and no amount of base-model improvement crosses it without first building the verifier.

## Why verifiability matters (the binding constraint hiding behind the capability headlines)

The capability story of 2024–2026 — o1, R1, Tülu-3, GPT-5, Claude-3.7, Gemini-2.5 — is, at the mechanism level, a **verifier story**. RL on verifiable rewards (RLVR, [232](232-rl-on-reasoning-traces-scaling.md)) is what folds test-time compute ([217](217-test-time-compute-scaling.md)) into the base model; verifier scaling ([223](223-verifier-and-best-of-n-scaling.md)) is what makes Best-of-N a budgeted lever rather than a free lunch. Every domain where 2024 → 2026 saw step-change agent improvement — competition math, code competition, formal proofs, SQL, web automation in narrow flows — is a domain with a *cheap verifier*. Every domain where progress was incremental — long-form writing, novel research, ambiguous business judgment, ethical adjudication, creative cohesion — is a domain where *verifier construction is hard*. The capability headlines are a verifier-availability map in disguise.

The framing matters because it gives founders, harness engineers, and roadmap planners a *single load-bearing question* to ask before any other: **"What's the verifier?"** For a task to be RL-eligible, you need a verifier. For an agent loop to close, you need a verifier. For Best-of-N to compound, you need a verifier. For autonomy to be safe to scale, you need a verifier. If the answer is "we'll figure it out", the task is not yet ready for the techniques that look so impressive in the verifier-rich domains; it is sitting in a trough on Karpathy's jagged curve, waiting for either a verifier or a verifier-substitute.

The framing also matters because **verifiers themselves are engineering artifacts** with cost, latency, alignment risk, and gameability. A verifier is not free; it can be wrong; it can be reward-hacked. The discipline of verifier engineering — choosing what to verify, how cheaply, how robustly to gaming, with what oversight — is the unglamorous infrastructure that the agent era runs on. Most teams treat eval as a checklist; *verifier engineering* treats it as a load-bearing system with its own SRE.

Take this seriously and three things change. **First**, you re-rank your roadmap by **verifier density** not by "how good is the LLM at this" — the two diverge sharply. **Second**, you treat **verifier construction as the primary investment** for any task family you intend to RL or scale autonomously; everything else is gated by it. **Third**, you accept that **the jagged frontier moves with the verifier frontier**, not with the model frontier — and you plan for occupational impact accordingly ([96](96-gdpval.md), [149](149-sector-use-case-catalog.md)).

## Problem it solves (the field's miscoordination of effort)

1. **"LLM is good at X" is uncorrelated with "agent loop closes on X".** Many teams chase tasks where the LLM seems capable in chat but cannot be wrapped in a closing loop because the verifier is missing.
2. **Test-time compute scaling lacks a coordinator.** Best-of-N is wasted compute without a verifier to pick from N; reasoning chains are wasted compute without a verifier to terminate them. Verifier is the missing axis.
3. **RL on reasoning traces requires verifiable rewards.** RLVR ([232](232-rl-on-reasoning-traces-scaling.md)) is the productization mechanism; without verifiers it does not run.
4. **Long-horizon autonomy is unsafe without verifier coverage.** The compounding error problem ([27](27-horizon-long-horizon-degradation.md)) is unsolvable from inside the agent; verifier checkpoints are how the harness keeps the trajectory bounded.
5. **LLM-as-judge is necessary but not sufficient.** Judges are themselves jagged ghosts ([97](97-qwen-prm.md)); they need calibration, agreement protocols, and ground-truth anchoring ([21](21-llm-as-judge-trajectory-eval.md), [115](115-evaluating-llm-systems.md)).
6. **Founders pick wrong wedges.** Verifier-rich wedges close the loop and compound; verifier-poor wedges produce demos that do not productionize.

## Core idea in one paragraph

The agent-era performance curve, holding base model fixed, is a function of four scaling axes — pretraining ([216](216-pretraining-scaling-laws-foundation.md)), test-time compute ([217](217-test-time-compute-scaling.md)), trajectory budget ([237](237-agent-trajectory-scaling.md)), multi-agent parallelism ([224](224-multi-agent-parallel-scaling.md)) — *crossed* by verifier infrastructure ([223](223-verifier-and-best-of-n-scaling.md), [225](225-agent-era-scaling-synthesis.md)); and **the verifier crossing dominates whether the other axes pay off**. RL on verifiable rewards ([232](232-rl-on-reasoning-traces-scaling.md)) folds TTC into the base model; Best-of-N with verifier picks the best sample; agent loops with verifier checkpoints bound long-horizon error; multi-agent ensembles with verifier aggregation avoid diversity collapse ([98](98-diversity-collapse-mas.md)). Where the verifier is cheap (compile + tests, theorem checker, simulator outcome, schema check, exact-match SQL row, image-3D-reconstructibility ([192](192-world-r1-3d-constraints-t2v.md)), ELO from self-play), the curve compounds and tasks fall to automation. Where the verifier is expensive (taste, ethics, ambiguous user intent, novel scientific judgment, long-form coherence), the *jagged ceiling* holds and the agent is competence-bounded by verifier scarcity. **Per-task automation surface ≈ verifier coverage × RL eligibility × deployment integration**, and the binding factor in 2026 is the first.

## Mechanism (how verifiability gates the agent loop)

### (a) The four roles of a verifier in an agent loop

A verifier shows up in four distinct places, often confused:

1. **Reward signal for RL** — at training time, RLVR uses the verifier to shape weights ([232](232-rl-on-reasoning-traces-scaling.md), [80-knowrl](80-knowrl.md)). High verifier quality → tighter reward signal → better convergence.
2. **Best-of-N selector** — at inference time, the verifier picks among N samples ([223](223-verifier-and-best-of-n-scaling.md)). Even a noisy verifier gives meaningful lift over majority voting if it correlates with true correctness.
3. **Loop terminator / step gate** — within an agent trajectory, the verifier decides when to stop or when to retry ([11](11-verifier-evaluator-loops.md), [18](18-chain-of-verification-self-refine.md)). Without it, the agent either over-runs (cost) or under-runs (quality).
4. **Eval / regression check** — at CI time, the verifier is the regression test for prompt/skill/tool changes ([21](21-llm-as-judge-trajectory-eval.md), [115](115-evaluating-llm-systems.md), [38](38-claw-eval.md)).

Each role has different cost/latency/correctness tolerances. A reward-signal verifier can be slow and rough; a loop-terminator verifier must be fast; a Best-of-N selector must be calibrated.

### (b) Verifier types, with examples

| Verifier type | Coverage | Cost | Examples |
|---|---|---|---|
| **Hard checker** (exact, deterministic) | High where applicable | Low | Compile + unit tests; theorem prover (Lean, Coq); SAT/SMT; schema validation ([112](112-constrained-decoding.md)); SQL exact-match |
| **Simulator outcome** | High in simulator-rich domains | Medium | Game self-play (chess, Go, StarCraft); robotics sim; code-execution sandbox ([102](102-clawgym-scalable-claw-agents.md)); 3D-reconstructibility ([192](192-world-r1-3d-constraints-t2v.md)) |
| **Process Reward Model (PRM)** | Per-step in math/reasoning | Medium | Lightman 2023 step-judging; Qwen2.5-Math PRM ([97](97-qwen-prm.md)) |
| **LLM-as-judge** | Broad but jagged | Medium-high | Trajectory eval ([21](21-llm-as-judge-trajectory-eval.md)), faithfulness check ([135](135-trustworthy-generation.md)) — needs calibration |
| **Retrieval-grounded check** | Where ground-truth corpus exists | Medium | Citation faithfulness, RAG groundedness ([25](25-agentic-rag.md), [135](135-trustworthy-generation.md)) |
| **Human grader** | Universal but expensive | Very high | RLHF ratings, preference labels |
| **Self-consistency** | Where multiple samples should agree | Low | Majority vote, agreement-based confidence ([88](88-confidence-driven-router.md)) |
| **External world feedback** | Production deployment | Variable | A/B test outcome, user click-through, returned-or-not, conversion |

The cost-per-verification *and* the correlation-with-true-value together determine whether RL or Best-of-N pays off.

### (c) The verifier hierarchy: hard > simulator > PRM > LLM-as-judge > human

Empirically, verifier types have strikingly different reward-hacking susceptibilities and scaling properties:

- **Hard checkers** are cheap, deterministic, and robust to gaming *if* the spec is correct. Failure mode: spec gaming (the agent satisfies the test but breaks the intent). Mitigation: stronger specs, mutation testing.
- **Simulators** scale broadly when domain has good simulators; reward hack is failure of sim-to-real gap.
- **PRMs** are useful per-step but their reward shape is hand-engineered or LLM-judged; Qwen2.5-Math PRM ([97](97-qwen-prm.md)) shows that LLM-as-judge step labels can *actively harm* PRM quality — the verifier needs ground-truth data, not other-LLM-generated labels.
- **LLM-as-judge** is the Swiss-army verifier but has the highest gameability surface; agreement protocols, calibration, and ground-truth anchoring are required ([21](21-llm-as-judge-trajectory-eval.md)).
- **Human grading** is the gold standard but does not scale; the entire RL-from-AI-feedback line is the attempt to substitute LLM-as-judge for it.

The harness rule: **prefer the highest-fidelity verifier that meets your latency and cost budget**; if forced to use LLM-as-judge, anchor it to a hard-checker subset for calibration.

### (d) Verifier-RL compounding: the actual mechanism behind the capability headlines

DeepSeek-R1 (Jan 2025) and OpenAI o1 (Sep 2024) both train via *RL on verifiable reasoning rewards* — math and code where the answer is checkable by hard checkers. The mechanism: sample trajectories, score with verifier, RL on the trajectories with positive reward, repeat. This compounds because:

- The verifier's coverage is wide enough that many trajectories produce signal.
- The signal is noisy but unbiased.
- Trajectories that pass the verifier *and* reach the answer in fewer tokens are preferred (length penalty), so the model learns concise reasoning.
- Once base capability has compounded, the verifier itself can be improved (better PRM, harder math sets), continuing the loop.

The DeepSeek-R1 paper's most-quoted finding — *RL alone on a strong base model produces o1-style reasoning behavior, with a stable training curve and emergent long-chain-of-thought* — is the existence proof that this compounding works at scale. The catch: it works only in domains with the verifier. In domains without, RL drifts or reward-hacks ([97](97-qwen-prm.md)).

### (e) Best-of-N + verifier: the inference-time analog

The 2024–2025 *Large Language Monkeys* result ([223](223-verifier-and-best-of-n-scaling.md)) shows that for many benchmarks, Pass@N grows steadily with N. But Pass@N is a *budget if you can pick*; without a verifier you can use only majority vote (which collapses on hard tasks where the right answer is rare). Verifier-driven Best-of-N closes the gap between Pass@N and Pass@1, and the lift is large where the verifier is good and zero where it is not.

### (f) Loop-terminator verifier: the long-horizon mechanism

In long-horizon trajectories ([237](237-agent-trajectory-scaling.md)), the verifier serves as a *checkpoint*: the agent runs N steps, the verifier scores progress, the harness either continues, retries the sub-trajectory, escalates, or terminates. This is the *step-gate* role and it is the load-bearing mechanism that prevents the compounding-error collapse on horizons longer than the model's natural coherence length ([27](27-horizon-long-horizon-degradation.md)). Without verifier checkpoints, agents drift; with them, they converge.

### (g) Verifier engineering as a discipline

A verifier is not a one-shot artifact. It needs:

- **Coverage measurement** — what fraction of incorrect outputs does it catch? ([21](21-llm-as-judge-trajectory-eval.md), [115](115-evaluating-llm-systems.md))
- **False-positive measurement** — what fraction of correct outputs does it reject?
- **Calibration** — verifier confidence vs true correctness probability.
- **Cost-per-verification** — dollars and milliseconds.
- **Gameability audit** — adversarial probing for reward-hack surfaces.
- **Drift monitoring** — does the verifier degrade over time as the agent's distribution shifts?
- **Ground-truth anchoring** — is the verifier tied to hard ground truth somewhere, or is it a chain of LLM-as-judges all the way down?
- **Versioning** — pinned, signed, and CI-tested.

Treat verifier work as core SRE, not as one-off ML.

### (h) The verifier-scarcity map

Different domains sit at different points on the verifier curve. Karpathy's "founder advice" claim is that founder-quality opportunities live where the verifier is **cheap** *and* the **deployment surface** is **constrained** — those are the tasks where the agent loop closes durably.

| Domain | Verifier availability | Notes |
|---|---|---|
| Competition math | High (hard checkers) | RLVR works; o1/R1 evidence |
| Competitive programming | High (compile + tests) | SWE-agent, Aider, OpenHands, gpt-pilot |
| Theorem proving | High (Lean / Coq) | AlphaProof, AlphaGeometry trajectory |
| SQL | High (exact-row) | Text-to-SQL agents ([138](138-text-to-sql-agents.md)) |
| Pattern-bound web automation | Medium (DOM checks, screenshots, simulators) | OSWorld, ClawBench, WebArena, computer-use |
| Document understanding (closed-form) | Medium | Schema + faithfulness checks ([135](135-trustworthy-generation.md)) |
| Open scientific research | Low (peer review surrogate) | AutoResearchBench ([101](101-autoresearchbench.md)); jagged 9% vs 80% across tasks |
| Long-form creative writing | Low | LLM-as-judge with calibration; weak signal |
| Customer service | Low-medium | A/B + return-rate as proxy verifier |
| Legal advice | Low (citation faithfulness only as substrate) | Verifiability ≠ correctness in this domain |
| Strategic decisions | Very low | No verifier; jagged ceiling holds |
| Ethical adjudication | Very low | Verifier construction is the underlying ethical question |

The insight: the entire post-2024 *capability story* is concentrated in the high-verifier rows; the low-verifier rows have not moved much, regardless of model improvements.

### (i) Karpathy's founder rule, formalized

The talk's founder advice translates into:

```
score(task) = verifier_density(task) × deployment_constraint(task) × user_value(task)
              − verifier_construction_cost(task)
```

Pick wedges that maximize this score, not raw "LLM-good-at" score. Many shiny wedges score badly because the verifier is expensive or LLM-as-judge is unreliable for the value users actually want.

## Empirical anchors

- **DeepSeek-R1 (Jan 2025).** Public RLVR training run; reasoning emerges from RL on verifiable rewards alone.
- **OpenAI o1 (Sep 2024).** Closed but widely-replicated; same recipe.
- **Lightman 2023 (Let's Verify Step by Step).** First strong PRM result; per-step verification beats outcome-only.
- **Cobbe 2021 (Training Verifiers).** Original dedicated math verifier; predates the RL era but anchors the verifier-as-trainable-artifact line.
- **Snell 2024 (TTC scaling).** Test-time compute scaling formalized; verifier is the missing factor in the original analysis.
- **Brown 2024 (Large Language Monkeys).** Pass@N curves with budget-vs-verifier interaction.
- **Qwen2.5-Math PRM ([97](97-qwen-prm.md)).** LLM-as-judge step labels actively harm PRMs — verifier hierarchy matters.
- **GDPval ([96](96-gdpval.md)).** Per-occupation automation rate ~ verifier density empirically.
- **AutoResearchBench ([101](101-autoresearchbench.md)).** Verifier-poor domain → 9% vs verifier-rich domain → 80%+ on adjacent shapes.
- **OSWorld ([95](95-osworld.md)), ClawBench ([34](34-clawbench-live-web-tasks.md)), Claw-Eval ([38](38-claw-eval.md)), ClawGym ([102](102-clawgym-scalable-claw-agents.md)).** Sandbox + verifier infrastructure as the *engineering substrate* under computer-use progress.
- **AlphaEvolve ([85](85-alphaevolve.md)).** Verifiability domain (matrix multiplication, scheduling) → real-world wins.
- **Memento ([106](106-memento-paper-theory.md), [109](109-memento-results-and-harness.md)).** Frozen-LLM agent on GAIA via retrieval-as-policy with verifier-driven memory updates.

## Variants and refinements

- **Verifier-from-LM (Constitutional AI, RLAIF).** Uses LLM to provide reward signal; works in domains with good few-shot judging.
- **Process vs Outcome verifiers.** PRMs (per-step) vs outcome verifiers (final answer); per-step gives denser signal but is harder to construct.
- **Self-verification / self-refine.** Agent grades own output; weak signal but compounds with sampling.
- **Co-evolutionary verification.** Verifier and policy evolve together ([169](169-coevoskills-co-evolutionary-verification.md)); risk of co-adaptation that drifts from ground truth.
- **Witness / provenance verification.** Trace + provenance enable post-hoc verification ([186](186-mnema-witness-lattice.md), [188](188-witness-provenance-memory-techniques-synthesis.md)).
- **Executable verifier wrappers.** Constrained decoding + schema enforcement as a *generative-time* verifier ([112](112-constrained-decoding.md)).
- **Probabilistic verifiers.** Confidence-driven routing as a soft verifier ([88](88-confidence-driven-router.md)).
- **Multi-verifier ensembles.** Combining hard + LLM-judge + retrieval-check; calibration is non-trivial.
- **3D-reconstructibility as a verifier.** World-R1 ([192](192-world-r1-3d-constraints-t2v.md)) uses 3D-from-2D-projection consistency as a generative-AI verifier; novel use of an *implicit* hard checker.

## Failure modes and limitations

- **Reward hacking.** Agent satisfies verifier without satisfying intent. Mitigation: spec hardening, mutation testing, multi-verifier ensemble.
- **Verifier drift.** Verifier was correct at training time, decays as agent distribution shifts. Mitigation: drift monitoring, regular re-anchoring.
- **LLM-as-judge collapse.** Judge and policy from same family share blind spots ([97](97-qwen-prm.md)); ensure cross-family judging or hard-checker anchor.
- **Verifier construction cost dominates.** For some domains, building the verifier is harder than building the agent. The honest answer is to delay automation, not pretend the verifier exists.
- **Spec gaming.** Test passes, intent fails. Mitigation: stronger specs, end-to-end checks ([135](135-trustworthy-generation.md)).
- **Verifier monoculture.** All teams use the same LLM-as-judge; collective gameability ensues.
- **Verifier-induced overfitting.** Policy optimizes the verifier rather than user value; deployment correlates poorly with verifier score.
- **Per-domain verifier impossibility.** In some domains (long-form taste, ethical judgment, novel research) the verifier *is* the question. Recognize and stop trying to automate.

## When to use this lens, when not

**Use it** when: (a) prioritizing roadmap items — verifier-rich tasks should lead; (b) deciding whether to RL — RLVR requires the verifier; (c) choosing Best-of-N budget — verifier quality determines payoff; (d) gating long-horizon autonomy — verifier checkpoints bound the trajectory; (e) advising founders — verifier density is the wedge predictor.

**Do not use it** when: (a) you need cognitive-science framings — the lens is engineering-first; (b) you face inherently verifier-poor problems where the right answer is to keep humans central, not to manufacture a verifier ([23](23-human-in-the-loop.md)); (c) the conversation is about safety threat-models — verifiers can be adversarial targets and need their own threat-modeling ([22](22-guardrails-prompt-injection.md), [49](49-agents-of-chaos-red-teaming.md)).

## Implications for harness engineering

1. **Lead with verifier construction.** For any new task family, the first artifact is the verifier; everything else (skills, prompts, RL data) is gated by it ([11](11-verifier-evaluator-loops.md), [115](115-evaluating-llm-systems.md)).
2. **Build verifier hierarchies, not single verifiers.** Combine hard checker (where possible) + LLM-judge + retrieval-grounded faithfulness ([21](21-llm-as-judge-trajectory-eval.md), [25](25-agentic-rag.md), [135](135-trustworthy-generation.md)).
3. **Anchor LLM-as-judge to hard ground truth.** Calibrate against a labeled subset; otherwise judges drift ([97](97-qwen-prm.md)).
4. **Bundle verifier with the agent loop, not as separate eval.** Loop terminator + Best-of-N selector + reward signal use the same verifier code path ([11](11-verifier-evaluator-loops.md), [18](18-chain-of-verification-self-refine.md), [223](223-verifier-and-best-of-n-scaling.md)).
5. **Budget verifier compute explicitly.** Verifier cost-per-call enters the same accounting as model cost-per-call ([86](86-frugalgpt.md), [87](87-routellm.md), [88](88-confidence-driven-router.md), [146](146-business-case-roi.md)).
6. **Score roadmap items by verifier density.** A simple per-task scorecard (verifier coverage, cost, gameability, deployment fit) beats LLM-vibe estimation ([149](149-sector-use-case-catalog.md), [127](127-loan-processing-multi-agent-case-study.md)).
7. **Treat verifier engineering as core SRE.** Coverage, false-positive, calibration, drift monitoring, gameability audit are recurring work, not one-shots.
8. **Use verifier checkpoints for long-horizon agents.** Bound trajectories; retry sub-trajectories; escalate to HITL on verifier disagreement ([23](23-human-in-the-loop.md), [27](27-horizon-long-horizon-degradation.md), [237](237-agent-trajectory-scaling.md)).
9. **Run verifier ensembles for high-stakes decisions.** Multi-verifier agreement is the analog of multi-region redundancy in distributed systems ([122](122-explainability-compliance.md), [123](123-robustness-fault-tolerance.md), [124](124-agent-level-production-patterns.md)).
10. **Distill verifier-driven trajectories into the model.** RLVR ([232](232-rl-on-reasoning-traces-scaling.md)) and skill internalization ([116](116-adapter-tuning-lora-peft-for-agents.md), [156](156-heavyskill-parallel-reasoning-deliberation.md)) are how 3.0 verifier work compounds into 2.0 capability.
11. **Plan for jagged occupational impact.** Sector roadmaps should explicitly map verifier landscape per task before sequencing ([96](96-gdpval.md), [149](149-sector-use-case-catalog.md), [141](141-e-commerce-marketing-agents.md), [127](127-loan-processing-multi-agent-case-study.md)).
12. **Recognize verifier-impossible tasks and route to humans.** Some tasks cannot be RL-eligible without redefining the value; use HITL not pretend-verifier ([23](23-human-in-the-loop.md), [122](122-explainability-compliance.md), [243](243-outsource-thinking-keep-understanding.md)).

## One-line takeaway for harness designers

**Verifiability is the load-bearing axis of the agent era — RL, Best-of-N, loop-termination, and CI all depend on a cheap, calibrated, gameability-audited verifier — so build the verifier first, score your roadmap by verifier density, and route work toward the verifier-rich peaks of the jagged frontier rather than chasing the verifier-poor troughs that look impressive in a chat window.**
