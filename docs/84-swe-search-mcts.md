# 84 — SWE-Search: MCTS-Augmented Software Engineering Agents

**Paper.** Antonis Antoniades, Albert Örwall, Kexun Zhang, Yuxi Xie, Anirudh Goyal, William Wang — *SWE-Search: Enhancing Software Agents with Monte Carlo Tree Search and Iterative Refinement* — ICLR 2025 — arXiv:2410.20285 (v6, 2 Apr 2025). Code: `github.com/aorwall/moatless-tree-search`; demo: https://streamlit.moatless.ai

**One-line definition.** Intra-attempt Monte Carlo tree search (MCTS) over an explicit agent action tree, with an LLM Value Agent supplying numeric scores plus natural-language “hindsight” feedback and a Discriminator Agent debating up to five terminal patches—yielding a **mean 23% relative** resolve-rate gain over the same backbone across five models on **SWE-bench Lite** (not SWE-Bench Verified in the published experiments).

## Why this paper matters (the +23% claim, applicability to all coding agents, search vs sampling)

The headline result is a **23% mean relative improvement** in resolve rate (Table 1, “mean % ∆”) when pairing five different base LMs (GPT-4o, GPT-4o-mini, Qwen2.5-72B-Instruct, DeepSeek V2.5, Llama-3.1-70B-Instruct) with SWE-Search versus the same **Moatless-Adapted** agent without MCTS. Individual relative deltas range from +17% (GPT-4o) to +27% (Qwen), showing the pattern is not a single-model artifact.

The contribution is **scaffold- and model-agnostic in principle**: any repository-level agent that can expose a branching state (code snapshot + file context + tool transcript) and re-expand from a parent after feedback can, in theory, sit under the same MCTS loop. That contrasts with **i.i.d. sampling** of full trajectories (e.g. “large language monkeys” style repeated rollouts), which treats each attempt as independent and throws away partial structure. Here, **search allocates inference budget to the tree**: which branch to expand next is decided by a UCT-style rule using learned/value-model signals, not only by resampling the policy.

The paper frames **inference-time scaling** (deeper search, richer value) as an alternative to **more pretraining** or task-specific RL.

## Problem it solves

1. **Single linear trajectories discard partial progress.** Standard LM agents follow essentially one path; if they commit early to a suboptimal edit or declare “finished” when tests are incomplete, they cannot rewind to a decision point and try an alternative without an external control loop.
2. **Replanning and flexible transitions are expensive without structure.** The authors’ Moatless-Adapted relaxes strict FSM order (e.g. Plan can transition to any state), which helps autonomy but **raises loop risk**; Moatless-v0.0.2 → Moatless-Adapted gains only ~1.4 pp on Lite, so flexibility alone is insufficient.
3. **MCTS is an alternative to policy RL training** for this domain: instead of learning a new π, maintain a **search tree** over the existing policy’s actions, using a value model for backup and a discriminator for final selection—trading more LLM and environment calls for higher pass@1.

## Core idea in one paragraph

SWE-Search represents each agent step as a **node** (codebase state, span-limited file context, trajectory history) and each tool/edit/plan choice as an **edge**. Monte Carlo tree search with a **modified UCT** score decides which (state, action) to expand, biasing early breadth and late exploitation. After each expansion, a **Value Agent** outputs both a scalar utility and an explanation string ε; ε is fed back as **hindsight** when the parent re-expands, nudging the Action Agent away from repeated mistakes. When several candidate patches exist, a **Discriminator** runs multi-agent debate over up to **five** solutions and picks the winner—improving on raw value alone (73% vs 84% correct selection in their reported analysis). The result is a **search-guided** rather than **greedy** coding episode inside one budgeted “attempt,” orthogonal to best-of-N across independent runs.

## Mechanism (step by step)

**(a) State and action representation.**  
States \(s_{t,\tau}\) and actions \(a_{t,\tau}\) form a two-tier hierarchy: action **type** τ ∈ {Search, Plan, Edit, …} and concrete **instantiation** (tool args, edit payload). The environment is a git-backed clone: each node references a **commit** and diff from the task’s initial state, so backtracking is checkout-to-parent, not “prompt-only undo.” File context is chunked into **spans** (functions, classes) to cap prompt size; tests run in SWE-bench Docker/Kubernetes pods. Fail-to-pass tests are **not** revealed, matching the benchmark harness—only “run tests” is available.

**(b) MCTS expansion.**  
Selection does **not** use an LM for UCT; it uses a closed-form UCT-like score (inspired by AlphaZero-style heuristics). The paper gives:

\[
\mathrm{UCT}(s,a) = V(s,a) + C\sqrt{\frac{\ln N(s)}{N(s,a)}} + \alpha e^{-\beta(d-1)} - \gamma d
\]

exploitation via \(V\), standard exploration, an **early-depth bonus** (wide exploration near the root), and a **late-depth penalty** to curb endless depth. A separate rule **abandons** nodes after consecutive **low** rewards. At each step, the algorithm expands the argmax UCT child subject to **max MCTS iterations** (100 in reported experiments) and **per-node child cap** (3 in Sec. 4; Appendix Table 2 lists defaults including max expansions 5 and max depth 20—operational code may differ).

**(c) Value estimation (LLM-based “oracle” hybrid).**  
The Value Agent implements \( (v_t, \varepsilon_t) = V(s, a, \{s_i\}, \{a_i\}) \): trajectory-conditioned, **non-Markov** by design. Inputs include file context, test outcomes, and instructions with **state-specific** evaluation prompts (Search vs Plan vs Edit). The scalar \(v_t\) is used in UCT backups; the string \(\varepsilon_t\) is critical—without it, re-expansions from the same parent **collapse in diversity** (the paper’s ablation theme). The authors report mis-estimation: the value function picked the eventually correct terminal patch in **73%** of cases on average, whereas the Discriminator reached **84%** when a correct patch existed among candidates (Figure 5a).

**(d) Iterative refinement loop (hindsight feedback).**  
When a child is evaluated, ε is stored and passed back when expanding **sibling** or **alternate** actions from the parent—Figure 2’s SymPy example: the Action Agent “finishes” after passing tests, but the Value Agent assigns **low** reward, citing **missing edge-case tests**; after re-expansion, new tests fail, forcing further edits to the true fix. This is the paper’s main qualitative argument for MCTS + feedback over a single pass.

**(e) Integration with existing scaffolds (moatless / moatless-tree-search).**  
The Action Agent is **Moatless-Adapted**: moatless-tools FSM extended so states spawn a **tree**, not a line, with git revert. The public `moatless-tree-search` repository implements the search layer; Table 2 lists `TreeSearchSettings` hyperparameters (c = 1.41, value temperature 0.2, action temperature 0.2, discriminator: 5 agents, 3 rounds, temperature 1.0). Baselines are Moatless-Adapted **without** search—same prompts, same models—so gains isolate **search + value + debate**.

**MCTS loop (pseudocode, paper-congruent).**

```text
init root s0 from issue context
for iter in 1..max_mcts_iter:
  leaf = select(root)                    # UCT with depth bonus/penalty, visit counts
  if leaf is terminal: break
  child = expand(leaf, action_policy)  # Action Agent proposes next typed action; env step
  (v, ε) = ValueAgent(trajectory_to(child))
  backup(child, v)                    # update N, aggregate values along path
  optionally inject ε as hindsight for parent re-expansion
candidates = up_to_5_terminal_patches()
return DiscriminatorDebate(candidates) # multi-round, multi-agent, judge concludes
```

## Empirical results

**Setting:** **SWE-bench Lite** (300 instances), resolve rate = Pass@1 unless noted. All numbers below are from the paper’s tables and Figure 3.

| Model | Moatless-Adapted Pass@1 | SWE-Search Pass@1 | Rel. Δ (%) |
|-------|------------------------:|------------------:|----------:|
| GPT-4o | 25.7 | 31.0 | +17 |
| GPT-4o-mini | 13.0 | 17.0 | +24 |
| Qwen2.5-72B-Instruct | 18.0 | 24.7 | +27 |
| DeepSeek V2.5 | 16.3 | 21.0 | +22 |
| Llama-3.1-70B-Instruct | 13.6 | 17.7 | +23 |
| **Mean rel. improvement** | — | — | **+23** |

**Pass@5** (Figure 3, Lite): e.g. GPT-4o 34.0; Qwen 25.7. **Table 4** (compute-matched): SWE-Search Pass@5 often beats Moatless Pass@5 from **5 independent runs** (e.g. Qwen 25.7 vs 22.3). **Cost (Table 3):** e.g. GPT-4o **$40.86 → $576** on Lite. Abstract says “SWE-bench” broadly; **Verified** is not tabulated here.

## Variants and ablations (depth of tree, expansion budget, value model choice)

- **Search depth / iterations:** Figure 4b shows monotonic **more issues resolved** as transition count increases, under conservative caps (100 iterations, 3 expansions per node in Sec. 4). The paper contrasts this with game MCTS, often run for thousands of iterations—software episodes are **shallow-budget** by necessity.
- **Expansion budget:** Per-node child limit (3 in experiments vs 5 in default Table 2) and max depth 20 jointly bound branching factor; together with 100 MCTS steps this determines wall-clock and API cost (Table 3).
- **Value model and prompts:** **State-specific** value prompts are critical: without them, the Value Agent undervalues useful early actions (e.g. “get context”) because they lack immediate diffs (Figure 4a). This is the clearest “ablation” in the main text.
- **Discriminator vs value-only:** The gap 73% → 84% on picking the correct final patch motivates debate even when UCT has already focused search.

## Failure modes and limitations

- **Value model bottleneck:** Misjudged action purpose leads to **undervalued** good branches; the process is only as good as V and ε. The paper documents systematic mis-interpretation before state-specific prompts.
- **State / action explosion:** Long-horizon repos can still blow up tree size; caps truncate search, reintroducing suboptimality. MCTS in games benefits from fast simulators; here each **edge is a real LLM + container step**, so **throughput** is the binding constraint.
- **Cost and latency:** Table 3 shows order-of-magnitude API cost increases for the strongest model—unusable without budget controls for many products.
- **Discriminator and debate overhead:** 5 agents × 3 rounds adds call volume; failures in debate could **harm** already-good value rankings (GPT-4o-mini is the exception where discriminator vs value gap is small in Figure 5a).
- **Benchmark scope:** Results are **SWE-bench Lite**; generalization to Verified, org-codebases, or non-Python stacks is not established in this paper.

## When to use, when not to

**Use** when: (1) you already run a **git-checkpointed** SWE agent and can pay **5–20×** API cost for high-stakes fixes; (2) **early wrong commits** and **premature stop** are observed failure modes; (3) you can maintain **state-specific** evaluation prompts; (4) a **visual tree** (their Streamlit demo) is acceptable for debugging.

**Avoid** when: latency or spend must track a **single linear rollout**; the environment **cannot** cheaply branch and revert; or the task horizon is so long that even capped MCTS rarely completes a meaningful search before budget exhaustion.

## Implications for harness engineering

- **[77-meta-tts-agentic-coding](77-meta-tts-agentic-coding.md):** Meta TTS scales compute **across** rollouts (RTV/PDR on **summaries**). SWE-Search is **intra-rollout** (MCTS + value + debate on a **state tree**). **Orthogonal, stackable** if budget allows—outer TTS with inner MCTS, or the reverse.
- **Alignment with [15-tree-of-thoughts-lats](15-tree-of-thoughts-lats.md) and [13-react](13-react.md):** Like Tree-of-Thoughts, SWE-Search is explicit **search over language/action steps**; unlike pure ToT on text, actions **mutate a repo** and **tests** provide semi-oracle signal. The ReAct *pattern* (thought, action, observation) is preserved at each edge; MCTS **schedules** which ReAct thread to extend.
- **Verifier loops [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md):** Tests and the Value Agent act as a **soft verifier stack**; the Discriminator is a second-stage **evaluator** over final patches. The harness should record **per-node** ε and UCT metadata for audit—critical when CE turns opaque.

**Harness design tips:** persist **commit ID per node**; log **(v, ε)**; expose **MCTS max iterations** as an SLO knob; split **product** vs **Table 3–scale** research budgets.

## Connections to other work in this corpus

- **Best-of-N / sampling (Brown et al.):** Independent rollouts vs **tree-structured** feedback reuse.
- **Koh et al. (web agent tree search):** SWE-Search avoids **LM-chosen** UCT for tractability; uses closed-form UCT.
- **Debate (Du, Khan, Amayuelas):** Discriminator is **post-search** over candidates, not a substitute for MCTS.
- **SWE-agent line (Yang et al., Agentless, AutoCodeRover, OpenDevin):** SWE-Search is a **search layer** on a Moatless-class backbone, not a new repo tool suite.

## Key takeaways

1. **+23% mean relative** resolve rate on **SWE-bench Lite** across **five** models vs Moatless-Adapted without MCTS (Table 1).
2. **MCTS + LLM value + text feedback** yields **hindsight** that breaks premature success and **loop** behavior (qualitative and appendix-D figures).
3. **UCT** uses **depth-biased** terms plus **abandon** rules, **not** LM-chosen UCT, for tractability.
4. **Discriminator** lifts final pick accuracy **73% → 84%** when a correct candidate exists, at non-trivial added cost.
5. **Cost scales sharply**; gains trade **dollars and latency** for pass@1—appropriate for high-value, batch-offline **harness** jobs more than sub-second IDE assists.
6. **Intra- vs inter-rollout scaling:** pairs naturally with **Meta TTS**-style **across-attempt** methods.

## References

- Antoniades, A., Örwall, A., Zhang, K., Xie, Y., Goyal, A., & Wang, W. (2025). *SWE-Search: Enhancing Software Agents with Monte Carlo Tree Search and Iterative Refinement.* ICLR 2025. arXiv:2410.20285. https://arxiv.org/abs/2410.20285
- Jimenez, C. E., et al. SWE-bench (GitHub issue resolution benchmark).
- Örwall, A. Moatless tools / tree-search codebase (GitHub).
- Kocsis, L., & Szepesvári, C. (2006). Bandit based Monte-Carlo planning.
- Silver, D., et al. AlphaZero / MCTS (as cited in SWE-Search).
- Yao, S., et al. (2023). Tree of Thoughts. Brown, B., et al. (2024). Inference scaling via repeated sampling.
- Multi-agent debate: Du et al.; Khan et al.; related citations in SWE-Search.
- SWE-bench Lite (300 instances) — primary evaluation set in Antoniades et al.
