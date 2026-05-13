# 106 — Memento: Memory-Augmented MDPs and Soft-Q Learning for Agents That Learn Without Fine-Tuning

**Paper.** Huichi Zhou, Yihang Chen, Siyuan Guo, Xue Yan, Kin Hei Lee, Zihan Wang, Ka Yiu Lee, Guchun Zhang, Kun Shao, Linyi Yang, Jun Wang — *Memento: Fine-tuning LLM Agents without Fine-tuning LLMs* — arXiv:2508.16153v2 — submitted 22 Aug 2025, revised 25 Aug 2025 — categories cs.LG / cs.CL — DOI 10.48550/arXiv.2508.16153 — code at https://github.com/Agent-on-the-Fly/Memento (mirrored at https://github.com/Memento-Teams/Memento).

**One-line definition.** Memento reframes continual agent improvement as a **Memory-augmented Markov Decision Process (M-MDP)** — the agent state is augmented with an episodic case memory, the policy is a *neural case-selection* function, and *learning happens by rewriting and re-reading memory*, not by gradient updates of the underlying LLM; under a soft-Q objective, this yields an entropy-regularised policy whose improvement step is implemented by storing better cases and whose evaluation step is implemented by retrieving them.

## Why this paper matters

The dominant story of the last two years has been: *if you want a better agent, fine-tune the LLM*. SFT on agent trajectories, RLHF on tool-use rewards, GRPO/PPO over multi-turn rollouts — all of these change θ. They are expensive, they are gated by access to base weights, and they invite catastrophic forgetting whenever the agent's distribution drifts. The Memento paper argues you can recover most of the empirical benefit of fine-tuning by treating the agent's *memory* as the learnable object and leaving the LLM frozen.

The technical contribution is not "memory helps agents" — that has been shown a hundred times. It is the *formalism*: framing the deployed agent as an M-MDP whose state explicitly includes a case bank, so that classical RL machinery (Q-functions, soft-Q optimal policies, policy iteration) applies directly to memory operations. Once you accept the formalism, three concrete consequences follow. (1) The optimal policy under entropy regularisation has a closed form that is exactly *softmax retrieval* over case-Q values — i.e., the standard top-K retrieval move that everyone already does is a special case of soft-Q with a particular Q estimator. (2) "Improving the agent" reduces to *learning a better case-selection scorer* — i.e., training a small dual-encoder retriever, not an LLM. (3) "Online RL" maps to *rewriting memory* with new high-reward trajectories; offline RL maps to *re-training the retriever* on accumulated cases.

The empirical headline is the deep-research instantiation: 87.88% GAIA validation Pass@3 top-1, 79.40% GAIA test, 66.6 F1 / 80.4 PM on DeepResearcher beating training-based SOTA, 95.0% SimpleQA, 24.4% HLE — without a single LLM gradient step. And critically, the gain on *out-of-distribution* tasks is +4.7 to +9.6 absolute points, which is the regime where Reflexion-style C-engineering typically degrades.

## Problem it solves

Five gaps in prior agent-learning work that Memento closes — each one explains a section of the paper.

1. **Static reflection workflows are rigid.** Reflexion / Self-Refine / CoVe write *natural-language* lessons that the next session reads as in-context. They have no concept of *which* lesson is relevant to the current state, no notion of trajectory reward, and no policy improvement guarantee. Memento replaces the hand-coded retrieval rule with a learned soft-Q policy.
2. **Gradient-based agent learning is heavyweight.** Training-based deep-research methods (DeepResearcher, ReSearch, Search-R1) require GRPO-style rollouts and access to model weights. Memento matches or beats them on DeepResearcher with no gradient updates to the LLM.
3. **No principled treatment of memory as a learnable component.** A-MEM, mem0, MemGPT, Generative Agents all treat memory as engineering — embeddings, summaries, eviction heuristics. Nobody has written down "this memory operation is a policy step in *this* MDP" before. Memento does.
4. **Continual learning corrupts under distribution shift.** Fine-tuning loses old skills when new data arrives; in-context reflection loses old lessons when the context window fills. Memory-rewriting decouples capacity from working set: the case bank grows, and the retriever decides what to surface.
5. **Closed-source LLMs cannot be fine-tuned.** GPT-4-class models cannot be touched. Memento's framework is the most rigorous answer to "how do you make a closed model agent learn from experience?" published to date.

## Core idea in one paragraph

Treat the agent's case bank as part of the MDP state. Define the action space at each step as a tool call or memory write. Let the *policy* over this augmented state be a soft-max over case-conditioned action values, and let the *Q-function* be parameterised by a small neural retriever that scores (query, case) pairs. Then (i) **policy evaluation** = retrieving and ranking cases for the current query, (ii) **policy improvement** = adding successful trajectories back to the bank and re-training the retriever on the resulting positives/negatives, and (iii) **planning** is just acting under the resulting soft-Q-greedy policy. The LLM is the deterministic environment dynamics function from (state, retrieved cases, action) to next observation; it is *not* part of the policy and is never updated. This is the cleanest reduction yet of "agent learning" to "small-network RL with the LLM as a fixed simulator."

## Mechanism (step by step)

### 1. M-MDP — the formal object

Standard MDP: ⟨S, A, P, R, γ⟩. Memento augments with an episodic memory bank **M** of past cases:

- **State** s_t = (q_t, h_t, M_t) where q_t is the current task / query, h_t is the trajectory history so far in this episode, and M_t is the (possibly large) memory of stored cases from prior episodes.
- **Action** a_t ∈ A combines two heads: a *tool action* (which MCP tool to call with what args) and an *implicit memory action* (which retrieved cases to condition on at this step). The paper's clean trick is to fold the memory choice *into* the policy distribution over tool actions, so it does not need a separate action head.
- **Transition** P(s_{t+1} | s_t, a_t): the LLM, given the current state plus retrieved cases, produces a tool call; the tool call's environmental result becomes the next observation; M_t is updated only at episode termination via the *memory rewrite* operator.
- **Reward** R: terminal reward equal to task success (binary or graded); zero or shaped per-step reward for budget control.
- **Discount** γ as standard.

Cases are tuples **(s_T, a_T, r_T)** — *final-step* state, action, and reward of past episodes — not the full trajectory. This is a deliberate compression choice: the final step encodes what worked, the rest is path-dependent and noisy. (The codebase confirms this: `memory.jsonl` stores final-step tuples.)

### 2. Soft-Q objective and the closed-form optimal policy

Memento uses entropy-regularised RL (max-ent / soft Q). The objective is:

J(π) = E_τ ~ π [ Σ_t γ^t ( r_t + αH(π(·|s_t)) ) ]

where H is Shannon entropy and α is the temperature. Standard soft-Q derivation gives:

- **Soft Bellman equation:** Q*(s,a) = r(s,a) + γ E_{s'} [V*(s')]
- **Soft value:** V*(s) = α log Σ_a exp(Q*(s,a)/α)
- **Optimal policy:** π*(a|s) = exp((Q*(s,a) − V*(s))/α) = softmax_a (Q*(s,a)/α)

The Memento paper's contribution is to *interpret* this softmax as **retrieval over the memory bank**. Specifically, when the action space is "pick which subset of cases to condition on," the Q-function over (state, case) pairs becomes a *retrieval scorer*, and the soft-Q-greedy policy becomes top-K softmax retrieval. The temperature α controls exploration vs exploitation — small α gives hard top-1 retrieval, large α gives uniform sampling.

The Appendix A derivation shows the optimal policy decomposes into:

π*(a|s) ∝ Σ_c π_LLM(a | s, c) · w*(c | s)

where π_LLM is the (frozen) LLM policy conditioned on retrieved case c, and w* is the soft-Q-optimal *case-selection weight*. This is the formal statement that "all the learning happens in w*, not in π_LLM."

### 3. Memory writing — the policy improvement step

After each episode, Memento appends the final-step tuple (s_T, a_T, r_T) to M. The naive write is unfiltered, but the paper proposes two refinements that the codebase implements:

- **Reward-thresholded writes.** Only successful (or sufficiently high-reward) trajectories enter the bank. This biases the case distribution toward positive demonstrations and makes contrastive training feasible.
- **Failed-case retention.** A subset of failed cases is also retained, marked as negatives, used as hard negatives in retriever training. This is what gives the agent something to *avoid* — without negatives, the retriever has no signal that a similar-looking past attempt actually crashed.

Memory writing is the M-MDP's analog of policy improvement: each new case shifts the estimated Q-distribution, and the retriever (when re-trained) picks up the shift.

### 4. Memory reading — the policy evaluation step

At inference time, given a new query q_t, the retriever computes Q̂(q_t, c_i) for each c_i ∈ M and returns the top-K. The K=4 ablation finding (peak F1/PM at K=4) is reported as the empirical sweet spot — beyond K=4, retrieval noise begins to drown out the planner's reasoning signal; below K=4, coverage of the relevant case region is too sparse.

The retriever has two implementations:

- **Non-parametric CBR.** Embedding similarity (e.g. dense embeddings from a pretrained encoder) plus a fixed scoring rule. No learning. Equivalent to BM25-style retrieval but in dense space.
- **Parametric CBR.** A *learned* dual-encoder retriever trained with contrastive loss on the accumulated (positive case, query) and (negative case, query) pairs. This is the soft-Q machinery in practice — Q̂(q, c) is exactly the dual-encoder score.

### 5. The two-stage planner-executor loop

The deep-research instantiation runs a **planner** (GPT-4 by default) that consumes the query plus retrieved cases and produces a structured plan, followed by an **executor** (o3 by default) that consumes each plan step plus a tool registry and emits tool calls. Cases inform the planner; the executor is largely case-agnostic because tool-call logic is more local. This split mirrors the M-MDP factorisation: high-level case-conditioned strategy lives in the planner; low-level tool execution dynamics live in the executor + environment.

```text
                ┌─────────────────────┐
   query q ───▶ │   retriever (Q̂)    │ ──▶ top-K cases
                └─────────────────────┘
                          │
                          ▼
                ┌─────────────────────┐
                │   planner (GPT-4)   │ ──▶ structured plan
                └─────────────────────┘
                          │
                          ▼
                ┌─────────────────────┐
                │   executor (o3)     │ ──▶ tool calls (MCP)
                └─────────────────────┘
                          │
                          ▼
                ┌─────────────────────┐
                │  environment (MCP)  │ ──▶ observations
                └─────────────────────┘
                          │
                  (terminate? → reward)
                          │
                          ▼
                ┌─────────────────────┐
                │  memory rewrite     │ ──▶ M_{t+1}
                └─────────────────────┘
```

### 6. Why soft-Q (and not vanilla Q)?

Two practical reasons. **Exploration:** entropy regularisation prevents the retriever from collapsing to a single nearest neighbour, which is fatal when the bank is small. **Probabilistic retrieval:** softmax retrieval composes cleanly with the LLM's own probabilistic decoding — the joint distribution over (case, action) is a single softmax with two coupled axes, which makes likelihood-weighting and beam search natural extensions. The paper sidesteps the maximum-entropy debate by simply adopting the SAC-style framework that has worked elsewhere.

## Empirical anchors

The paper backs the framework with five empirical results, all with no LLM gradient updates:

- **GAIA validation:** 87.88% Pass@3 top-1 — the headline number, beating training-based agents.
- **GAIA test:** 79.40% — top-1 on the leaderboard at submission time.
- **DeepResearcher:** 66.6 F1, 80.4 PM — exceeds the SOTA *training-based* method (DeepResearcher's own RL-trained agent).
- **DeepResearcher OOD:** +4.7 to +9.6 absolute F1/PM — the regime where in-context-only memory is supposed to fail; case-based memory transfers because cases encode *strategies* not *answers*.
- **SimpleQA:** 95.0% accuracy.
- **HLE (Humanity's Last Exam):** 24.4% — limited by tool capabilities, not memory.

The K=4 ablation is the cleanest interpretability result: peak performance with 4 retrieved cases per planner step, monotone degradation beyond K=8 due to context dilution. Also reported (paper / repo): "small, high-quality memory beats large case banks" — a counter-intuitive finding given the field's RAG-bigger-is-better instinct.

## Variants and counter-arguments addressed

The paper explicitly positions against four neighbouring lines of work.

- **Reflexion (Shinn et al. 2023).** Reflexion writes natural-language reflections to a buffer, prepends them next round. Memento's case bank is structured (s, a, r) tuples with a *learned* retriever; Reflexion's buffer is unstructured text with naive prepend. Memento generalises Reflexion: under the M-MDP, Reflexion is the special case where K = ∞ (always read everything), the retriever is identity, and there is no negative-case channel.
- **A-MEM (Yan et al. 2024).** A-MEM proposes hierarchical memory with summarisation. Memento argues hierarchy is unnecessary if the retriever is good enough; soft-Q on flat memory recovers the hierarchical benefits.
- **MemGPT (Packer et al. 2023).** MemGPT pages context in and out of a virtual memory. It is *capacity engineering*, not policy learning. Memento's bank is unbounded; the retriever's softmax handles capacity implicitly.
- **Voyager (Wang et al. 2023).** Voyager builds a *code-skill* library and selects skills via embedding match. This is essentially the non-parametric CBR variant of Memento, applied to procedural memory. Memento generalises to declarative case-based memory and adds the parametric retriever and soft-Q framing.

The key claim is that all four are *instances* of the M-MDP framework, with different retriever choices and different memory structures. Memento's contribution is the unifying framework plus the parametric instantiation.

## Failure modes and limitations

1. **Frozen base policy.** The LLM is fixed. Whatever it cannot do in-context, no amount of case retrieval will fix. Paper [100](100-contextual-memory-is-a-memo.md)'s Compositional Sample Complexity Separation theorem applies directly: Memento is a state-of-the-art *C-engineering* system, and the Ω(k²) coverage demand for compositional novelty still binds. The +4.7–9.6 OOD gain is real but bounded.
2. **Case-bank scaling.** As M grows, retriever training cost grows, and the retriever itself can overfit to bank-specific surface features. The repo's "small, high-quality memory works best" finding is partly an admission of this; large banks have not been stress-tested.
3. **Reward signal is task-dependent.** GAIA, SimpleQA give clean binary rewards. In open-ended deployment (research assistant, code agent), the reward function is the hardest part of the loop, and Memento doesn't solve it — it inherits whatever reward you can construct.
4. **GAIA Level-3 still hard.** The paper reports compounding errors on long-horizon Level-3 tasks. The single-step (s_T, a_T, r_T) case representation under-specifies multi-step strategies; the trajectory shape itself is information the bank discards.
5. **Reproducibility.** Benchmarks depend on tool quality (search engine, code interpreter, document parser). The repo's README explicitly notes HLE performance is "limited by tool capabilities alone." Numbers are not portable across tool stacks.
6. **No negative-case theory.** The paper writes that failed cases help, but does not formalise *which* failures help. Random failure inclusion can poison the retriever.
7. **Closed-source executor dependency.** Default config uses GPT-4 + o3. Open-source replication is acknowledged as limited.

## When to use, when not

**Use Memento's framework when** you have a frozen / closed-source LLM, a stream of tasks with verifiable success signal, an out-of-distribution generalisation goal that pure recall cannot satisfy, and engineering capacity to maintain a retriever training pipeline. The deep-research setting is the canonical fit: queries are expensive, prior research traces are reusable, and reward (did the answer match ground truth?) is automatable.

**Skip it when** you can fine-tune the base model and have the data; tasks are i.i.d. and well-covered by pretraining (use plain retrieval); the success signal is unreliable (the retriever will learn the wrong thing); or you need single-session adaptation without an offline retraining loop.

## Implications for harness engineering

- **Promote retrieval from utility to policy.** Most harnesses today treat retrieval as a utility tool — `search_memory()` returns a list, the agent reads it. Memento says retrieval *is* the policy. The harness contract should treat the retriever as a first-class learnable component with its own training schedule, checkpoints, and rollback. See [42-langchain-deep-agents](42-langchain-deep-agents.md) for the closest existing pattern.
- **Add memory rewrite as a deterministic post-episode hook.** The post-episode write is not an LLM action; it is harness-deterministic. This is exactly the contract of [05-hooks](05-hooks.md). A `PostEpisode` hook that scores the trajectory and conditionally appends a (s_T, a_T, r_T) tuple to memory is the cleanest implementation.
- **Bridge Reflexion → Memento.** Many harnesses have Reflexion-style reflection buffers ([14-reflexion](14-reflexion.md), [81-reasoningbank](81-reasoningbank.md)). The Memento upgrade path is: (i) keep the buffer, (ii) add structured reward annotations, (iii) train a small retriever on (positive, negative) pairs, (iv) replace the prepend-everything rule with soft-Q top-K. This is incremental and does not require touching the base model.
- **Compositional caveat.** Memento delivers without θ-engineering, but [100](100-contextual-memory-is-a-memo.md) is explicit that learned-retrieval still binds to ᾱ on compositional novelty. The honest framing is: Memento is the best you can do *while staying frozen*; real expert-level generalisation still wants a consolidation channel from cases to weights. See [109-memento-results-and-harness](109-memento-results-and-harness.md) for the head-to-head reading.
- **Two-stage planner/executor as a harness pattern.** The planner-executor split mirrors several existing patterns ([16-plan-and-solve](16-plan-and-solve.md), [42-langchain-deep-agents](42-langchain-deep-agents.md), [02-subagent-delegation](02-subagent-delegation.md)). Memento adds a clean rule: cases inform the planner, tools inform the executor; the executor stays case-blind. This makes planner caching tractable and keeps tool execution deterministic.
- **MCP as the action space.** The M-MDP's action space *is* the MCP tool registry. This is the first published work that takes this identification seriously. See [07-model-context-protocol](07-model-context-protocol.md) and [108-memento-codebase-mcp](108-memento-codebase-mcp.md) for the implementation.
- **Soft-Q temperature is a harness knob.** α (the entropy temperature) is exposed in code as the retriever's softmax temperature. It is the single most important parameter for the explore-exploit trade-off in case retrieval and should be a harness-level config, not an internal constant.

The one-sentence takeaway: **Memento says the LLM is the simulator, not the agent — the agent is the case bank plus a tiny retriever, and that flips which component you optimise.**

## See also

- [107-memento-cbr-memory](107-memento-cbr-memory.md) — the CBR mechanism, parametric retriever architecture, and training pipeline.
- [108-memento-codebase-mcp](108-memento-codebase-mcp.md) — the GitHub repo walkthrough.
- [109-memento-results-and-harness](109-memento-results-and-harness.md) — benchmarks, ablations, and a head-to-head reading against [100](100-contextual-memory-is-a-memo.md).
- [14-reflexion](14-reflexion.md), [81-reasoningbank](81-reasoningbank.md), [19-voyager-skill-libraries](19-voyager-skill-libraries.md) — neighbouring memory designs Memento generalises.
- [25-agentic-rag](25-agentic-rag.md) — how agentic RAG with self-critique compares to Memento's case bank.
