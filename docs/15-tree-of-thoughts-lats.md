# 15 — Tree of Thoughts & LATS

**Definition.** Tree of Thoughts (ToT, Yao et al. 2023) generalizes chain-of-thought by letting the model explore multiple candidate reasoning steps at each stage, evaluate them, and search the resulting tree (BFS or DFS) for the best solution. LATS (Language Agent Tree Search, Zhou et al. 2023) extends this to agents with tools, using Monte Carlo Tree Search over action trajectories with an explicit value function. Both trade more compute for better reasoning on hard problems.

## Problem it solves

Chain-of-Thought commits to one reasoning path. If that path goes wrong three steps in, the model can't back up — the error propagates. This is fine for problems where one path obviously works; it is badly broken for problems requiring look-ahead (puzzles, planning, multi-hop QA, math with branching cases). Humans solve those by trying a path, realizing it won't work, backtracking, and trying another. ToT gives the model the same affordance.

LATS addresses the agent version of the same problem. A ReAct agent commits to a single tool-call sequence; if a tool returns nothing useful at step 4, the whole trajectory is wasted. LATS treats action sequences as tree paths and uses search to explore alternatives.

## Mechanism

### Tree of Thoughts

Four primitives:

1. **Thought decomposition.** Define what one "thought" is for the problem — an equation step, a move in a game, a sentence of an essay plan.
2. **Thought generation.** At each node, prompt the model for *k* candidate next thoughts.
3. **State evaluation.** Prompt the model (or a separate evaluator) to score each state ("sure", "maybe", "impossible") or produce a scalar value.
4. **Search algorithm.** BFS or DFS with pruning, controlled by evaluator scores.

Loop:

```
root = initial_state(task)
frontier = [root]
while frontier:
    node = pop(frontier)          # BFS or best-first
    if is_solution(node): return node
    candidates = model.propose_next_thoughts(node, k=5)
    scored = [(c, model.evaluate(c)) for c in candidates]
    frontier.extend(top_n(scored, n=3))  # prune
```

### LATS (Language Agent Tree Search)

LATS lifts ToT into the agent setting with full MCTS:

- **State** = an agent trajectory (Thought/Action/Observation sequence so far).
- **Actions** = model-proposed next steps.
- **Selection** uses UCB1 on existing nodes: exploit nodes with high value, explore nodes with few visits.
- **Expansion** asks the model for new candidate actions at promising nodes.
- **Simulation** continues the rollout greedily (or with the base policy).
- **Backpropagation** sends rewards (from environment or LLM evaluator) up the tree.
- **Reflection** (Reflexion-style) augments the value function with verbal lessons.

The result: LATS outperforms ReAct, Reflexion, and plain CoT on HotpotQA, HumanEval, and WebShop in the original paper's experiments.

## Concrete pattern

ToT on a simple "Game of 24" task (4 numbers, use +, −, ×, ÷ to make 24):

```
root: {4, 5, 7, 9}
level 1 (propose partial operations):
  - 4 + 5 = 9 → {9, 7, 9}      eval: "maybe"
  - 9 - 5 = 4 → {4, 4, 7}      eval: "maybe"
  - 9 - 7 = 2 → {4, 5, 2}      eval: "sure"  ← expand
  - 4 × 9 = 36 → {36, 5, 7}    eval: "impossible" (overshoot) ← prune
level 2 from {4, 5, 2}:
  - 5 - 2 = 3 → {4, 3}         eval: "sure" (4 × 3 = ... no 12)
  - 4 × 2 = 8 → {8, 5}          eval: "maybe"
  - 4 + 2 = 6 → {6, 5}          eval: "sure"
  ...
```

Evaluator prunes branches the model judges dead; the search continues on survivors.

## Variants & related techniques

- **Graph of Thoughts** (Besta et al.) adds edges for merging and backtracking across branches.
- **Self-consistency** is the cheap cousin: sample N chain-of-thoughts, majority-vote the answer. No tree, no evaluator, but surprisingly effective.
- **Best-of-N / rejection sampling** samples multiple trajectories and picks the highest-scoring one — a one-level tree.
- **LATS + Reflexion.** Value function updated with reflections across trials; stronger than either alone on stateful tasks.
- **MCTS with learned value models** (AlphaCode, AlphaGo-style). Expensive to train but much stronger for problems with clear reward.
- **Plan-and-Solve** ([16-plan-and-solve.md](16-plan-and-solve.md)) is the sequential, non-search version.

## Failure modes & anti-patterns

- **Compute blowup.** ToT/LATS can call the model 50–500× more than a single CoT run. Quantify the win vs. cost before using.
- **Weak evaluators.** If the evaluator can't reliably distinguish promising branches from dead ones, the search wastes compute. Fix: ground the evaluator in external checks (a test runner, a calculator) when possible.
- **Dominant root mistake.** If the initial decomposition of "what is a thought" is wrong, no amount of search fixes it. Fix: experiment with granularity on a small set first.
- **Over-pruning.** Greedy evaluators kill correct-but-unusual branches. Fix: keep some exploration (ε-greedy, UCB1).
- **Under-pruning.** Tree explodes and you run out of budget. Fix: hard depth/width caps.
- **Non-deterministic evaluators.** The same branch is scored differently on different visits. Fix: temperature 0 for evaluation, or cache scores.
- **Ignoring cheap baselines.** Self-consistency (N parallel CoT samples + vote) often closes most of the ToT gap at a fraction of the cost; always benchmark the cheap version first.

## When to use (and when not to)

**Use** ToT/LATS when:

- The task rewards look-ahead (puzzles, planning, theorem-proving, multi-hop QA).
- You have a reliable evaluator (tests, a verifier, a strong critic).
- You can afford significantly more model calls.
- Failure cost is high enough to justify the compute.

**Don't** use them when:

- A single CoT call solves the task reliably — no need for search.
- Your evaluator is as noisy as the generator — search amplifies noise.
- The task is latency-sensitive; users won't wait for a 500-call tree search.
- Self-consistency or best-of-N already closes the gap.

## References

- Yao et al., "Tree of Thoughts: Deliberate Problem Solving with LLMs", arXiv:2305.10601 — <https://arxiv.org/abs/2305.10601>
- Zhou et al., "Language Agent Tree Search Unifies Reasoning, Acting and Planning", arXiv:2310.04406 — <https://arxiv.org/abs/2310.04406>
- Besta et al., "Graph of Thoughts", arXiv:2308.09687 — <https://arxiv.org/abs/2308.09687>
- Wang et al., "Self-Consistency Improves Chain of Thought Reasoning in Language Models", arXiv:2203.11171 — <https://arxiv.org/abs/2203.11171>
- LangChain LATS implementation and tutorial — <https://python.langchain.com/docs/tutorials/lats/>
