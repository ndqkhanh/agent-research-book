# 90 — Reflexion (Paper Deep-Dive): Verbal Reinforcement for Language Agents

**Paper.** Noah Shinn, Federico Cassano, Edward Berman, Ashwin Gopinath, Karthik Narasimhan, Shunyu Yao — *Reflexion: Language Agents with Verbal Reinforcement Learning* — Northeastern University / MIT / Princeton — NeurIPS 2023 — arXiv:2303.11366 (2023). *Paper-grounded complement to* [14-reflexion](14-reflexion.md) *; dense PDF detail beyond the file-14 note.*

**One-line definition.** Reflexion is a **gradient-free, weight-free** optimization loop: an **Actor** rollouts a trajectory, an **Evaluator** emits a scalar reward, and a **Self-Reflection** LLM distills that signal into first-person **verbal reinforcement** that is **appended to episodic memory** and re-injected on the next trial—turning sparse/binary feedback into a *semantic* improvement direction without backprop or fine-tuning.

## Why this paper matters (the canonical reference for self-reflection in agents)

**Verbal RL** here means the effective policy is **\(\{M_a, mem\}\)**, not updated weights. Algorithm 1 / Fig. 2 fix the **Actor / Evaluator / Self-Reflection** template now standard in agent stacks. The same paper gives **negative results** (WebShop; Starchat-beta) with equal explicitness—important for production harness design.

## Problem it solves

LLM-based agents (ReAct, tool use, code execution) can interact with environments, but **classical RL** is costly: many samples, credit assignment in semantic action spaces, and **fine-tuning** is often unavailable for hosted frontiers. Reflexion targets **fast adaptation across a handful of trials** by converting environment feedback (binary or scalar) into **actionable natural-language hints** the next episode can read—analogous to human few-shot learning from mistake analysis rather than weight updates. The paper explicitly frames the difficulty of **credit assignment** and the need to **amplify** weak evaluators into rich verbal summaries.

## Core idea in one paragraph (verbal RL: reflections as gradient-free signal)

Reflexion treats **self-reflective text** as the “learning signal.” The Evaluator may only see *pass/fail* or exact match; the Self-Reflection model \(M_{sr}\) conditions on \(\{\text{trajectory}, r_t, \text{prior } mem\}\) and outputs \(sr_t\): a first-person, task-specific distillation of *what went wrong* and *what to do differently.* That string is a **stand-in for a policy gradient** in the sense that it nudges the next sampled trajectory without differentiating the network—**interpretable, inspectable, and auditable** compared to a dense reward vector, at the cost of no formal optimality guarantee and dependence on the LLM’s self-assessment quality.

## Mechanism (step by step). Cover the SPECIFIC architecture: (a) Actor (ReAct-based); (b) Evaluator (binary, programmatic, or LM); (c) Self-Reflection model; (d) episodic memory pruning (last k); (e) the three task families: decision-making (AlfWorld), reasoning (HotpotQA), programming (HumanEval, MBPP)

**(a) Actor \(M_a\).** The Actor is an LLM **conditioned on observations and on memory** \(mem\). The paper uses **ReAct** for AlfWorld and HotpotQA (explicit thoughts + actions) and **Chain-of-Thought** variants for HotpotQA ablations. Policy notation: \(\pi_\theta(a_t \mid s_t)\) with **\(\theta = \{M_a, mem\}\)**—memory is part of the effective policy, not a separate training phase.

**(b) Evaluator \(M_e\).** \(r_t = M_e(\tau_t)\) is **task-scaled to improve with correctness**. Instantiations:
- **Binary / EM:** HotpotQA uses **exact match** on answers between trials.
- **Programmatic heuristics (AlfWorld):** e.g. **repeat same action + same response \(>3\)** cycles, or **\(>30\)** steps ⇒ trigger reflection; environment only signals final success.
- **LM-as-judge / NL classification:** AlfWorld can use **binary classification via an LLM** instead of the heuristic to decide when/what failed.

**(c) Self-Reflection model \(M_{sr}\).** After trial \(t\), \(M_{sr}\) takes **\(\{\tau_t, r_t\}\)** (and the evolving memory) and outputs **verbal experience** \(sr_t\). The paper stresses turning **sparse** rewards into *nuanced* text—e.g. attributing errors to a **wrong early action** in a long trajectory. Programming adds **self-generated unit tests** and compiler feedback as part of the trajectory signal before reflection.

**(d) Episodic memory pruning.** **Long-term memory** stores \(\{sr_0, sr_1, \ldots\}\); **short-term** is the current trajectory. In practice, **\(|mem| \le \Omega\)** with **\(\Omega\) usually 1–3**; AlfWorld **truncates to the last 3** reflections; the **code loop uses max 1** stored experience for HumanEval/MBPP-style runs—explicitly to respect **context limits**. Appending is strictly **by trial**, not by step inside a trajectory.

**(e) Task families.**
- **AlfWorld:** 134 envs, six task types; **130/134** with heuristic self-eval; **+22%** absolute in abstract vs strong baselines over **12** trials.
- **HotpotQA:** 100-question runs; ReAct or CoT; **EM**; retry until **3** consecutive fails per item.
- **Code (HumanEval, MBPP, LeetcodeHardGym):** zero-shot bodies; **MultiPL-E** for Rust; **CoT** unit tests, **AST** filter, **\(n \le 6\)** tests. **LeetcodeHardGym:** 40 post–Oct 2022 “hard” items, 19 langs (post GPT-4 cutoff).

**Algorithmic loop (pseudocode, after Algorithm 1):**

```text
Init Ma, Me, Msr; mem <- []
t <- 0
Run τ0; r0 <- Me(τ0); sr0 <- Msr(τ0, r0, mem); mem <- [sr0]  # trim to Ω
while Me(τt) not pass and t < max_trials:
  t <- t + 1
  τt <- roll_out(Ma, task, mem)           # ReAct/CoT inside
  rt <- Me(τt)
  srt <- Msr(τt, rt, mem)
  mem <- append_and_trim(mem, srt, Ω)    # e.g. last 1-3
return τt
```

## Empirical results (table — Reflexion vs base ReAct on each task; specific HumanEval 91% pass@1 for Reflexion+GPT-4 vs 80% baseline)

*Baselines are task-appropriate: AlfWorld and HotpotQA use **ReAct**-family agents as in the paper; code benchmarks compare **Reflexion + GPT-4** to **single-shot GPT-4** (not ReAct), as Table 1 reports.*

| Setting | Baseline (paper) | Reflexion (paper) | Notes |
|--------|------------------|-------------------|--------|
| **AlfWorld** | ReAct (plateau; **~22%** halluc. tail, Fig. 3b) | **130/134** + Reflexion, **12** trials | Heuristic or GPT loop detect |
| **HotpotQA** (N=100) | ReAct + GPT-4 **0.39** (Table 5) | **0.51** + Reflexion | CoT ablations higher in other figures |
| **HumanEval (PY) pass@1** | **80.1%** (GPT-4 one-shot, Table 1) | **91.0%** | Not ReAct; CodeT+GPT-3.5 65.8% in paper row |
| **MBPP (PY)** | **80.1%** | **77.1%** | FP self-tests **~16%** vs HE **1.4%** |
| **HumanEval (RS, 50 hard)** | **60.0%** | **68.0%** | MultiPL-E |
| **MBPP (RS)** | **70.9%** | **75.4%** | — |
| **Leetcode Hard (PY)** | **7.5%** | **15.0%** | LeetcodeHardGym |

**Reasoning (paper text):** CoT(GT) still wrong on **39%**; Reflexion **+14%** without gold answer. **EPM vs Reflexion ablation (Fig. 4c):** trajectory memory alone worse by **~8%** abs. than + verbal reflection.

## Variants and ablations (programmatic vs LM reflection, k value)

- **AlfWorld:** Heuristic (stuck/length) vs **GPT** self-eval; heuristic run hits **130/134**.
- **Memory \(\Omega\):** Globally **1–3** reflections stored; **code** often **1**; **AlfWorld** **last 3**; HotpotQA **3**-experience memory in the main Reflexion runs.
- **HotpotQA ablation (CoT (GT) + EPM vs + Reflexion):** Storing the prior trajectory (EPM) helps, but **self-reflection** adds **~8%** absolute over EPM—evidence the *verbal* lesson is not reducible to “more context.”
- **HumanEval Rust (50 hardest) — Table 3:** **Full** Reflexion **68%**; **no test generation** (reflection only) **52%** (below **60%** baseline); **no self-reflection** (tests but no NL lesson) **60%** (no gain)—both components needed on hard code.
- **Starchat-beta (Table 4):** On HumanEval Python, **Baseline 0.26 vs Reflexion 0.26** (avg over 8 trials)—**self-correction is an emergent capability of stronger models**; small open weights may not benefit.

## Failure modes and limitations

- **Binary / sparse eval dependency:** If \(M_e\) is uninformative, \(M_{sr}\) can hallucinate causes; programming suffers **false positives** when **flaky** self-tests **pass** on wrong code—MBPP’s **higher FP rate (16% vs 1.4%)** explains the **worse** Reflexion pass@1 than GPT-4 on MBPP Python despite HumanEval gains.
- **Local minima & exploration:** The authors state Reflexion can **plateau in non-convex** policy space; **WebShop** experiments show **no** improvement after a few trials and **unhelpful** reflections—**diverse, precise search** behavior is not fixed by short verbal memory.
- **Memory and context:** Sliding-window memory is a **crude** cap; the paper points to **vector DBs / SQL** as future work. Episodic growth still matters for long campaigns.
- **TDD limitations:** **Non-deterministic, impure, hardware-dependent, or parallel** code breaks reliable self-tests—named explicitly in Sec. 5.
- **Safety / misuse:** Sec. 6 notes amplified autonomy risk; also claims interpretability may help monitor tool intent—conditional upside.

## When to use, when not to

**Use** when: (1) the environment returns **reliable, cheap feedback** (unit tests, EM, success bit); (2) **multiple attempts** are allowed; (3) the model is **strong enough** to generate corrective language (GPT-4 family shows large deltas; Starchat does not). **Avoid** or temper when: one-shot, no eval; **noisy** judges; **exploration-heavy** web/commerce search with **ambiguous queries** (WebShop evidence); or **weak** base models for which reflection is vacuous. For code, **invest in test quality** first—ablations show **tests + reflection** are interdependent on hard tasks.

## Implications for harness engineering. Reference [14-reflexion](14-reflexion.md), [13-react](13-react.md), [09-memory-files](09-memory-files.md), [81-reasoningbank](81-reasoningbank.md), [77-meta-tts-agentic-coding](77-meta-tts-agentic-coding.md). Position as: Reflexion is the ancestor of ReasoningBank, claude-mem, and agent self-improvement; PDR refinement context (Meta TTS) is structurally similar.

- **[14-reflexion](14-reflexion.md)**: operator pattern (evaluator + reflector + last **k**); paper backs **k** and expected deltas.
- **[13-react](13-react.md)** is the **Actor** scaffold for two of three families; Reflexion’s “policy = LLM + mem” is **ReAct**-compatible out of the box.
- **[09-memory-files](09-memory-files.md)** is the **out-of-session** analogue: the paper’s in-context \(mem\) is the **intra-task** version of persistent **reflection logs** across runs.
- **[81-reasoningbank](81-reasoningbank.md)**-style **structured experience banks** are an evolution of the same bet—**verbalized failures → reusable priors**—with richer indexing than a sliding list.
- **[77-meta-tts-agentic-coding](77-meta-tts-agentic-coding.md) (PDR):** **Parallel-Distill-Refine** conditions fresh rollouts on **compressed prior outcomes**; Reflexion conditions the **next trial** on **distilled text**. Both are **“summarize or reflect, then re-attempt”** without weight updates, but PDR’s summaries are **multi-rollout, tournament-scaled TTS**; Reflexion is **lighter, trial-local verbal RL**—a conceptual ancestor at smaller budgets.

**Positioning sentence:** In this corpus, **Reflexion** is the **ancestral paper** for **agent self-improvement without training**: later systems (including **ReasoningBank**, **claude-mem**-style file memory, and **PDR**-style reuse of prior attempts) keep the same move—**persist what you learned in language**—while engineering better **storage, retrieval, and selection**.

## Connections to other work in this corpus

**[13-react](13-react.md)** supplies the trajectory scaffold. The paper’s related-work table contrasts **Self-Refine** (no memory, no long-horizon decision-making) with Reflexion; **in-context policy iteration (Brooks et al.)** motivates memory-augmented **\(M_a\)**. **CodeT / AlphaCode / Self-Debugging / CodeRL**: test execution without the paper’s **error→NL→next trial** bridge; **hidden tests** break **pass@1** story in several cited lines. **LeetcodeHardGym** = post-cutoff **OOD** code stressor.

## Key takeaways

1. **Three-role decomposition** (Actor, Evaluator, Reflexion) is the portable blueprint; your harness swaps **\(M_e\)** per domain (judge, heuristic, EM, pytest).
2. **Verbal RL** = **no gradients**; gains on **HumanEval** hit **91% pass@1** vs **~80%** one-shot **GPT-4**—**large on strong models, zero on Starchat** in the paper’s own table.
3. **Last‑\(\mathbf{k}\)** memory is not laziness; it is a **necessary** cap under **context** limits—documented to **1** for their code setting.
4. **Self-tests can lie:** MBPP shows **higher** false-positive rates than HumanEval; **reflection cannot fix** confidently wrong evaluators.
5. **Reflexion is not universal:** **WebShop** is a **negative result**; **exploration-bound** tasks need different machinery (or larger search), not more prose.

## References

- Shinn, N., Cassano, F., Berman, E., Gopinath, A., Narasimhan, K., Yao, S. (2023). *Reflexion: Language Agents with Verbal Reinforcement Learning.* NeurIPS 2023. arXiv:2303.11366. https://arxiv.org/abs/2303.11366  
- Code: https://github.com/noahshinn024/reflexion  
- Yao, S. et al. (2023). *ReAct: Synergizing Reasoning and Acting in Language Models.* ICLR. (Actor baseline family.)  
- Wei, J. et al. (2022). *Chain-of-Thought Prompting Elicits Reasoning in Large Language Models.* (CoT ablations in HotpotQA.)  
- Shridhar et al. (2021). *ALFWorld*; Yang et al. (2018). *HotpotQA*; Chen et al. (2021). *HumanEval*; Austin et al. (2021). *MBPP*; Cassano et al. (2022). *MultiPL-E* (Rust translation).  
- See also: OpenAI (2023). *GPT-4 Technical Report* (Leetcode cutoff baseline).
