# 68 — Atomic Skills: Scaling Coding Agents by Basis-Vector Decomposition (arXiv:2604.05013)

**Definition.** *Scaling Coding Agents via Atomic Skills* (Ma, Liu, Yang, Li, Fu, Miao, Xie, Wang, Cheung — HKUST / NUS / PKU / SJTU / BUPT; arXiv:[2604.05013](https://arxiv.org/abs/2604.05013), submitted 6 Apr 2026) is a paradigm paper that replaces the dominant "train on composite benchmarks (bug-fixing, SWE-bench)" scaling recipe with a **basis-vector decomposition** of coding work into five *atomic skills* — **code localization, code editing, unit-test generation, issue reproduction, code review** — then scales agent capability via **joint reinforcement learning over a single shared policy**, letting skills improve together through positive transfer rather than in isolation. Trained on GLM-4.5-Air-Base (106B total / 12B active), the resulting Base-SFT-RL agent achieves an **18.7 % average absolute gain across 5 atomic skills + 5 composite OOD tasks** — including monotonic improvements on all five atomic skills *and* all five held-out composite benchmarks (SWE-bench Verified/Multilingual, Terminal-Bench 2.0, Code Refactoring, SEC-Bench). The paper is the clearest published demonstration that **what you train on at the skill layer matters more than what you train on at the task layer**, and it provides both a named taxonomy and an infrastructure blueprint (10,000+ concurrent ephemeral Kubernetes sandboxes, 25,000+ pre-built Docker images) for anyone attempting the same move.

This file sits alongside [19-voyager-skill-libraries.md](19-voyager-skill-libraries.md) (skills as a *library*), [04-skills.md](04-skills.md) (skills as a *harness construct*), and [31-glm-5-agentic-engineering.md](31-glm-5-agentic-engineering.md) (GLM-5 training for agentic engineering). Where Voyager treats skills as durable artifacts an agent accumulates *at inference time*, and SKILL.md treats them as *harness capabilities*, Atomic Skills treats them as **training-time reward targets** — the reward functions that shape what the policy becomes. That distinction is worth stating clearly up front because "skills" is by now an overloaded word in the 2026 literature.

## Problem it solves

The paper opens with a concrete, embarrassing observation that every practitioner has felt and most have not named: **training on a high-profile composite benchmark does not generalize to other composite benchmarks of the same "skill family"**. The authors show (Fig. 1) that RL on SWE-bench-style bug-fixing improves bug-fixing but *underperforms* on code refactoring, issue reproduction, code review, and unit-test generation — all of which share basic infrastructure (understand code, edit code, run tests) with bug-fixing. The policy has memorized the *shape of the benchmark's reward surface*, not the *capabilities the benchmark supposedly measures*.

Two structural reasons are offered:

1. **The reward signal is black-box.** Composite tasks reward the final outcome (test suite passes) without shaping the intermediate steps. A policy that exploits benchmark idiosyncrasies (file layouts, test-harness quirks, patch shape priors) gets the same reward as one that genuinely understood the bug. RL has no way to prefer the latter.
2. **Composite tasks are an infinite-dimensional space.** Real-world software work has unbounded variety — there is no scalable path that enumerates all possible high-level tasks and designs dense rewards for each. Pick any three composite benchmarks and their intersection is narrower than you think.

This is the failure mode [27-horizon-long-horizon-degradation.md](27-horizon-long-horizon-degradation.md) documents at inference time (capabilities degrade as horizons grow) read backwards onto training time: *training on long horizons with sparse rewards amplifies the degradation because there's no dense signal to hold the policy together*.

The paper's bet is that complex coding **decomposes into a small, closed set of atomic skills that** (a) have precise I/O specs, (b) admit unambiguous execution-based rewards, (c) recombine into any composite task. If that bet is right, training a single policy jointly over the five of them produces an agent whose capability *compounds* across tasks rather than overfits to one.

## The three atomic-skill design principles

Not every capability qualifies. The paper imposes three constraints that function as a filter:

1. **Precise I/O specification.** The task's input and output are formally definable. "Fix bug X" is not atomic — "output a ranked file list that exactly matches the files the human PR modified" is. Specification is what makes reward functions writable.
2. **Independent, unambiguous evaluation.** The skill can be graded in a sandbox without human judgment, without dependency on other skills, and without ambiguity about what "success" means. This is the execution-grounded-reward principle echoed in [21-llm-as-judge-trajectory-eval.md](21-llm-as-judge-trajectory-eval.md) and [11-verifier-evaluator-loops.md](11-verifier-evaluator-loops.md) — but earlier in the stack, before the judge has to be invoked.
3. **Reusable as a building block.** The skill recurs across composite tasks. Fixing a bug requires localization + editing; refactoring requires editing + test generation; reviewing a PR requires review + reproduction. If a candidate skill appears in exactly one composite task, it is too narrow to carry generalization.

These three principles are the clearest articulation I have seen of what "atomic" actually means for agent training, and they generalize beyond coding. Anyone building a domain-specialized agent (see [30-gpt-rosalind-domain-specialized.md](30-gpt-rosalind-domain-specialized.md), [28-radagent-agentic-radiology.md](28-radagent-agentic-radiology.md)) can apply them to find their domain's basis vectors: precise I/O, unambiguous evaluator, appears in ≥ 2 composite tasks.

## The five atomic skills — reward functions and rationale

### 1. Code Localization

- **Input.** Natural-language issue description + codebase.
- **Output.** Ranked list of file paths most relevant to the issue.
- **Reward.**
  \[
  r_{loc}(x, y) = \begin{cases} +1 & \text{if } \hat{F}(y) = F^* \\ -1 & \text{otherwise} \end{cases}
  \]
  where \(F^*\) is the set of files the ground-truth PR modified.
- **Supervision source.** GitHub issue → PR pairs; the PR's modified-file set becomes the reference.
- **Design choice.** **Strict equality** (not overlap, not Jaccard, not F1). The authors defend this as avoiding "ambiguous partial credit" — the agent must learn to approximate the *distribution of file selections human developers make*, not identify "vaguely related" regions. This is a strong stance. It sacrifices training signal density (fewer positive examples) for reward unambiguity, and it shifts the generalization burden onto sample efficiency.

### 2. Code Editing

- **Input.** Code context + explicit edit instruction.
- **Output.** A patch.
- **Reward.** `1` iff **all** repository unit + regression tests pass after applying the patch; else `0`.
- **Design choice.** Reward is **functional correctness under execution**, not string similarity to a reference patch. This is the SWE-bench-style reward and shares its strengths (unambiguous, real-world-aligned) and weaknesses (tests must exist; tests must be strict enough to not accept broken code; tests must be passable).
- **Note.** *The agent is rewarded for patches that pass tests*, not for producing the human-written patch. This is important and underdiscussed: two semantically different patches can both pass the same test suite. The policy is therefore free to discover alternative solutions, which is probably *good* for generalization.

### 3. Unit-Test Generation

- **Input.** Target function + behavioral spec (natural-language docstring).
- **Output.** A test suite for that function.
- **Reward.** `1` iff the generated tests (i) pass on the original correct implementation *and* (ii) catch every injected buggy variant in the variant set \(B(f)\).
- **Buggy variants.** Generated via LLM-based semantic mutation (Kimi-K2-Thinking produces 16 variants per target). Variants are filtered so only those the *original golden tests catch* are used for reward.
- **Design choice.** Rewards based on **fault-detection capability, not coverage**. Coverage metrics reward tests that touch lines without verifying behavior; fault detection rewards tests that actually check semantics. This is an important inversion of the usual "percent coverage" metric that QA teams obsess over.

### 4. Code Review

- **Input.** Issue description + a candidate PR (generated by independent coding agents — Claude Code, SWE-agent, OpenHands — then validated).
- **Output.** (i) Natural-language review summary; (ii) binary judgment: does this PR fully address the issue?
- **Reward.** `1` iff binary judgment matches ground truth label, else `0`. The NL review is not directly rewarded but is a structural output.
- **Design choice.** **No reward on the review text itself** — only on the binary conclusion. This is a deliberate simplification: text quality is hard to grade automatically. A downstream deployment would want to add a judge-based reward for the text, but as an atomic training signal the binary outcome is sufficient to push the policy toward reasoning about semantic completeness.

### 5. Issue Reproduction

- **Input.** Issue description + codebase.
- **Output.** A minimal executable script or command sequence that reproduces the failure.
- **Reward.** `1` iff (i) script triggers the failure on the original code *and* (ii) the failure disappears after applying the ground-truth patch.
- **Judge.** An LLM-based log judge (Kimi-K2-Thinking) compares pre/post patch execution logs. The paper defends this judge use as narrow: it is comparing two logs for a specific failure pattern, not evaluating code quality — a task that reduces well to a closed binary question.
- **Design choice.** **Two-sided verification** (failure present pre-patch, absent post-patch). Single-sided verification would accept false reproductions that fail for unrelated reasons. The two-sided check is tight but requires access to the ground-truth patch at reward time, which is a training-time-only signal the deployed agent never has access to — exactly as it should be.

## Joint reinforcement learning — the training mechanism

### Unified trajectory sampling

Rollouts **do not separate by skill**. A unified skills buffer mixes task instances; rollout workers sample (skill, input, context) triples at random; all resulting trajectories write into a single buffer that feeds a single policy update. No skill-specific heads, no skill-specific optimizers, no skill-gated mini-batches. The policy sees "code editing task, then localization task, then test-gen task, then editing task" in whatever order the buffer produced.

This design choice is the paper's central bet. Its theoretical justification is that **shared representations for code understanding, execution reasoning, and tool usage** should emerge under pressure from all five reward functions simultaneously, and these shared representations should be what transfers to OOD composite tasks. Empirically, Fig. 4 shows that all five atomic skills improve monotonically from the SFT checkpoint under joint RL — no trade-offs, no capacity fights, no skill dominates another.

### GRPO with relative advantages

The optimizer is **Group-based Relative Policy Optimization** (Shao et al., 2024 — the same GRPO that DeepSeek used). For each input \(x\), the rollout workers generate a group of \(N\) candidates \(\{y_i\}\); the group's rewards are normalized within the group:

\[
A_i = r_i - \frac{1}{N} \sum_{j=1}^{N} r_j
\]

and the loss is

\[
L_{GRPO}(\theta) = \mathbb{E}\left[\sum_{i=1}^{N} A_i \log \pi_\theta(y_i \mid x)\right]
\]

optionally regularized by a KL term to the SFT initialization.

Why this matters for atomic skills specifically: **reward magnitudes differ across skills** (code editing's binary-all-tests-pass reward has different variance than localization's ±1 exact-match reward). PPO-style absolute-advantage estimation would mix these scales in ways that distort the gradient. GRPO's within-group normalization removes the scale mismatch — the policy learns the *relative* quality of its rollouts for each sampled task, decoupling the scalar magnitudes.

### Decoupled rollout/training workers

Rollout generation is asynchronous from policy optimization. Rollout workers pull tasks from the unified buffer and run agent trajectories in the sandbox; trainer workers consume completed trajectories and run batched GRPO updates. This is standard distributed-RL scaffolding but matters here because **atomic skills have heterogeneous execution costs** — a localization task completes in seconds; an editing task runs a full test suite. Synchronous pipelines would let slow skills stall the fast ones. Decoupling lets both churn at their natural rates.

## Infrastructure — the part that makes this reproducible

Most agentic-RL papers say "we trained with execution-based rewards" and leave the infrastructure as an exercise. This paper is unusually explicit, and the numbers are worth reading twice.

- **10,000+ concurrent sandboxes** on Kubernetes hybrid-cloud clusters, each ephemeral (create-on-demand, destroy-after-use).
- **25,000+ pre-built Docker images** covering environments for unit testing, regression testing, issue reproduction, etc. Using a sidecar container architecture means the agent runs in an isolated primary container while evaluator tooling runs in the sidecar — standard but non-trivial at this scale.
- **Reward-hacking defenses baked into the sandbox**: network access is *disabled* during training, and `.git` history is *removed* from the repo checkout. The network block prevents the policy from learning to google the answer; the git wipe prevents it from discovering the ground-truth patch by walking history.

This is the infrastructure counterpart to the "closed loop" idea in [36-autogenesis-self-evolving-agents.md](36-autogenesis-self-evolving-agents.md) — you cannot run a closed learning loop without a container substrate that can spin up 10k isolated environments per minute. The paper's willingness to put numbers on this will be directly useful to anyone attempting to replicate the approach on a smaller budget.

### Minimal tool scaffolding

The agent's action space is **two tools**: `bash` and `str_replace`. That's it. The paper argues this minimal set is sufficient for all five atomic skills — and more importantly, the uniform tool interface across skills enforces consistent interaction patterns that make parameter sharing work. If different skills had different tool APIs, the shared-representation argument would weaken.

This connects to a broader 2026 thesis visible across the corpus: **small, uniform tool sets beat large heterogeneous ones for training stability** (see [05-hooks.md](05-hooks.md)'s observation that hook behavior generalizes better when hooks share a uniform event shape, and [07-model-context-protocol.md](07-model-context-protocol.md)'s goal of a uniform tool surface across integrations).

## Empirical results — what the numbers actually say

Two tables carry most of the evidence. Table 1 is atomic skills; Table 2 is OOD composite tasks. The comparison is three-way: GLM-4.5-Air (strong reference), GLM-4.5-Air-Base + SFT (our SFT-only baseline), GLM-4.5-Air-Base + SFT + RL (full joint atomic-skills RL).

### Atomic skills (Avg@3)

| Skill | GLM-4.5-Air | Base + SFT | Base + SFT + RL | Δ SFT → SFT+RL |
|---|---|---|---|---|
| Code Location | 0.666 | 0.665 | **0.712** | +4.7 pp |
| Code Editing | 0.556 | 0.458 | **0.611** | +15.3 pp |
| Issue Reproduce | 0.555 | 0.542 | **0.605** | +6.3 pp |
| Unit Test Gen | 0.423 | 0.359 | **0.472** | +11.3 pp |
| Code Review | 0.536 | 0.563 | **0.622** | +5.9 pp |

### OOD composite tasks (Avg@3)

| Benchmark | GLM-4.5-Air | Base + SFT | Base + SFT + RL | Δ SFT → SFT+RL |
|---|---|---|---|---|
| SWE-bench Verified | 0.559 | 0.507 | **0.585** | +7.8 pp |
| SWE-bench Multilingual | 0.358 | 0.300 | **0.389** | +8.9 pp |
| Terminal-Bench 2.0 | 0.187 | 0.151 | 0.182 | +3.1 pp |
| Code Refactoring | 0.159 | 0.146 | **0.171** | +2.5 pp |
| SEC-Bench | 0.163 | 0.136 | **0.169** | +3.3 pp |

The headline number — **18.7 % average improvement across 5 atomic skills + 5 composite tasks** — is the macro-average over both tables. What's more informative than the average is the **shape**:

- **SFT alone underperforms the reference model on most tasks.** SFT on atomic-skill data is not enough; the agent regresses relative to a general-purpose instruct-tuned model. The gain lives in the RL step.
- **The biggest gains are on editing and unit-test generation** (+15.3 and +11.3 pp). These are the skills with the strictest, most execution-grounded rewards. The reward surface shape matters enormously — binary "all tests pass" beats any kind of partial-credit / text-similarity reward.
- **OOD benchmarks improve without any OOD training signal.** The agent never saw SWE-bench Verified, Terminal-Bench, SEC-Bench during RL — yet all five improve. This is the generalization claim, and it holds.
- **The ceiling on Terminal-Bench 2.0 is low.** 0.187 → 0.182 is arguably noise. The paper acknowledges this: Terminal-Bench exercises shell / OS reasoning that atomic coding skills do not directly train. *Generalization is real but not unlimited* — it transfers across code-adjacent tasks but not to orthogonal ones.

### The ablation that actually proves the claim

Joint atomic-skills RL vs **single-task RL** (editing-only, verified-only) is the critical ablation (Fig. 5). Verified-only RL optimizes for SWE-bench-Verified reward directly and does improve that benchmark — but:

- **It improves less on refactoring, issue reproduction, code review** than joint RL does. The specialization to bug-fixing is real but narrow.
- **On the trained target itself (SWE-Verified), single-skill and joint RL are comparable late in training.** Joint RL pays no obvious cost for being broader.

Translated: *joint RL gets you the specialist's benchmark score and the generalist's capability transfer*. The paper's claim is that this is not a happy coincidence — it is what the shared-policy shared-representation theory predicts. The ablation is consistent with the theory.

## Connections back to the corpus

This paper is a rare case where a training-methodology paper directly informs harness design. Several existing files in the corpus can be read through its lens:

- **[04-skills.md](04-skills.md).** The SKILL.md pattern is *inference-time skill invocation*; this paper is *training-time skill reward shaping*. They are complementary. An ECC-style skill library ([62-everything-claude-code.md](62-everything-claude-code.md)) could in principle feed its skill definitions back into an atomic-skill RL pipeline — curated `SKILL.md` files become candidate atomic skills whose reward functions then have to be written.
- **[19-voyager-skill-libraries.md](19-voyager-skill-libraries.md).** Voyager accumulates skills at inference. Atomic Skills scaffolds *which skills a policy should be good at* at training. A Voyager-style agent on top of a policy trained with Atomic Skills RL would get both benefits — native capability plus a library to extend further.
- **[11-verifier-evaluator-loops.md](11-verifier-evaluator-loops.md) & [21-llm-as-judge-trajectory-eval.md](21-llm-as-judge-trajectory-eval.md).** The paper's reward functions *are* verifier/evaluator loops compiled into a training signal. Code review's binary judgment is exactly an LLM-as-judge pattern, but grounded by ground-truth PR correctness labels derived from downstream tests.
- **[27-horizon-long-horizon-degradation.md](27-horizon-long-horizon-degradation.md).** Atomic skills are *short-horizon* tasks by construction. Joint RL over atomic skills is therefore training the agent on primitives that do not exhibit horizon degradation, then hoping the primitives compose into long-horizon capability. The empirical transfer to OOD composite tasks is weak evidence that this hope is partially justified, at least over code-adjacent horizons.
- **[31-glm-5-agentic-engineering.md](31-glm-5-agentic-engineering.md).** Atomic Skills sits naturally alongside GLM-5's async RL + long-horizon trajectories. Where GLM-5 tackles the long-horizon directly, Atomic Skills argues for a decomposition-based alternative. Both might be complementary; nothing in the paper argues they are mutually exclusive.
- **[66-meta-harness-landscape.md](66-meta-harness-landscape.md).** The atomic-skill decomposition is *also* a candidate vocabulary for a harness IR — if five atomic skills recur across all coding tasks, a harness IR could type its operations against those skills, making trace analyzers ([67-recommended-breakthrough-project.md](67-recommended-breakthrough-project.md)) natively aware of which skill failed.

## Limitations, failure modes, and what the paper does *not* answer

A careful read surfaces several open questions worth naming:

1. **Are these five skills the right basis?** The paper proposes them as a working set motivated by real-world SE workflows but does not attempt a formal proof of completeness. A sixth skill ("refactoring" is the obvious candidate — currently a composite OOD task rather than an atomic) could easily be the next frontier. The atomic skill concept is defensible; the specific five are the current best guess, not a theorem.
2. **Reward hackability.** Code localization's strict equality is robust; code editing's "all tests pass" is hackable if the test suite is incomplete (the agent can pass tests without fixing the bug). The sandbox defenses address external leaks but not internal test-weakness — that is a dataset-curation problem the paper acknowledges implicitly (issues are selected where tests are strict) but does not solve.
3. **Scale of evidence.** Results are on GLM-4.5-Air-Base (106B total). Claims about smaller models, much larger models, or non-GLM families are out of sample. The Anthropic / OpenAI / Gemini families may not behave the same. In particular, reasoning-heavy models with thinking traces (O1-family) may get different marginal gains — the paper does not try.
4. **Training-data bleed.** The RL stage uses 300 judge-verified trajectories per skill (1,500 total). At joint-RL scale that's small; but the *initial* SFT uses trajectories generated by gpt-oss-120b. The final policy therefore inherits a gpt-oss-120b distribution in some sense. Whether that matters for downstream commercial deployment is left to the reader.
5. **Generalization is finite.** Terminal-Bench gains are weak. Anything that requires shell / OS / long-range-planning reasoning that is *not* a compositional function of the five skills will not improve. The paper's claim is "compositional OOD transfer across code-adjacent tasks," and that claim holds. A stronger claim would not.
6. **Reproduction cost.** 10,000+ concurrent K8s sandboxes and 25,000+ pre-built Docker images is a ~$100K-$1M infra commitment depending on region / duration. The paper demonstrates the principle works but does not make it cheaply replicable for an academic lab without cloud credits.

## Takeaways — practical implications for harness engineering

1. **Train on basis vectors, evaluate on compositions.** The strongest evidence-based training recipe for coding agents as of April 2026. If you control your agent's training at all, spend the budget on atomic skills rather than one more SWE-bench pass.
2. **Reward functions want execution-based grounding.** Binary pass/fail > string similarity > LLM-judge > human label, roughly in that order for reliability. The paper's strongest skills (editing, test-gen) have the strongest reward functions.
3. **Uniform tool API + shared policy = transfer.** A small, uniform tool set is not a limitation — it is a prerequisite for shared-representation learning to work. Resist the temptation to give the agent 100 tools during training.
4. **Infrastructure is the moat.** 10k concurrent sandboxes is a real engineering effort. A team that wants to train atomic skills is paying an order-of-magnitude more for infrastructure than for model compute. Budget accordingly.
5. **Atomic skills as a harness-design vocabulary.** The five skills are almost perfectly mappable onto the bounded subagent decomposition in [46-components-of-coding-agent.md](46-components-of-coding-agent.md) (Raschka's six components). A harness that has a localizer subagent, an editor subagent, a test-writer subagent, a reviewer subagent, and a reproducer subagent — each tuned to one atomic skill — is one natural architectural distillation of this paper into inference-time practice. The ECC agent taxonomy ([62-everything-claude-code.md](62-everything-claude-code.md)) partially anticipates this structure.
6. **Chain training and harness design.** The atomic-skills training gives you a policy strong at primitives; the harness (loop, memory, subagent, verifier — docs 01–25) composes primitives into long-horizon work. These are separate layers. A team that treats them as separate — improving each independently — will compound gains.
7. **This is the wedge that replaces SWE-bench-obsession.** The SWE-bench benchmark is over-fit upon. The paper's framing — "composite tasks are the *evaluation*, atomic skills are the *training signal*" — is a cleaner separation between training-time reward and evaluation-time metric, and it deserves to become the default methodology.

## Production-readiness assessment

This is a **research paper** as of April 2026, not a deployable system. Its production readiness breaks down:

- **Green.** The taxonomy of five atomic skills and the three design principles are immediately usable as a harness-design heuristic. Any team can audit their agent's training data against this checklist today.
- **Green.** The infrastructure patterns (ephemeral K8s sandboxes, no-network / no-git reward-hacking defenses, minimal tool scaffolding) are lifted directly and are industry-grade.
- **Amber.** Reproducing the exact RL results requires GLM-4.5-Air-Base access (open-weights-available) plus 10k sandbox infrastructure. Feasible for a funded lab; unrealistic for a solo builder.
- **Red.** No open-source code or model weights released with the paper as of the preprint date (2026-04-06). Follow the authors' subsequent releases; in the meantime the methodology is extractable but the artifact is not.

## References

- **Primary.** Ma, Liu, Yang, Li, Fu, Miao, Xie, Wang, Cheung. *Scaling Coding Agents via Atomic Skills*. arXiv:[2604.05013](https://arxiv.org/abs/2604.05013) [cs.SE], v1, 6 Apr 2026. 612 KB, 20+ pages. Affiliations: HKUST, NUS, PKU, SJTU, BUPT.
- **Related methodology.**
  - Shao et al. 2024 — GRPO, *DeepSeekMath* (the optimizer used here).
  - Team et al. 2025a — GLM-4.5 (the base model family).
  - Team et al. 2025b — Kimi-K2-Thinking (the semantic-mutation and log-judge model).
  - Jimenez et al. 2023 — SWE-bench (the "wrong way to scale" this paper reacts against).
  - Ma et al. 2024; Guo et al. 2025; Yang et al. 2025c — composite-task RL scaling that this paper critiques.
- **Benchmarks evaluated on.**
  - SWE-bench Verified (OpenAI 2024) — Python bug-fixing, verified.
  - SWE-bench Multilingual (Yang et al. 2025b) — multi-language bug-fixing.
  - Terminal-Bench 2.0 (Merrill et al. 2026) — bash-only, mixed SE/MLE/security.
  - SEC-Bench (Lee et al. 2025b) — security PoC reproduction.
  - Code Refactoring (this paper) — 300 curated refactoring tasks.

## Cross-references in this corpus

- [04-skills.md](04-skills.md) — SKILL.md as inference-time skill invocation.
- [11-verifier-evaluator-loops.md](11-verifier-evaluator-loops.md) — verifier patterns that rewards functions here compile into.
- [19-voyager-skill-libraries.md](19-voyager-skill-libraries.md) — accumulated inference-time skill libraries.
- [21-llm-as-judge-trajectory-eval.md](21-llm-as-judge-trajectory-eval.md) — judge-based evaluation underlying several rewards.
- [27-horizon-long-horizon-degradation.md](27-horizon-long-horizon-degradation.md) — short-horizon atomic skills as the antidote.
- [31-glm-5-agentic-engineering.md](31-glm-5-agentic-engineering.md) — GLM-5 agentic RL approach.
- [36-autogenesis-self-evolving-agents.md](36-autogenesis-self-evolving-agents.md) — closed-loop learning infrastructure.
- [46-components-of-coding-agent.md](46-components-of-coding-agent.md) — bounded subagent decomposition that maps onto atomic skills.
- [62-everything-claude-code.md](62-everything-claude-code.md) — reviewer/resolver subagent pairs as inference-time analog.
- [66-meta-harness-landscape.md](66-meta-harness-landscape.md) — meta-harness landscape this paper sits alongside.
- [67-recommended-breakthrough-project.md](67-recommended-breakthrough-project.md) — Gnomon's harness-IR vocabulary.
