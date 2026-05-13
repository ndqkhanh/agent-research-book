# 103 — Agent Lightning: Train Any Agent with Reinforcement Learning Without Code Rewrites

**Paper.** *Agent Lightning: Train ANY AI Agents with Reinforcement Learning* — arXiv:2508.03680 — Microsoft Research — 2025. Code and framework: open-sourced via Microsoft Research.

**One-line definition.** Agent Lightning is a **framework-agnostic** training stack that adds RL to existing agent codebases (LangChain, AutoGen, custom Python) by **decoupling agent execution from model training** — the agent code does not change; a thin RL adapter intercepts model calls, records trajectories, and pushes gradients.

## Why this paper matters

[100-deepseek-r1-rl-reasoning](100-deepseek-r1-rl-reasoning.md), [101-ragen](101-ragen.md), and [102-artist-agentic-rl-tools](102-artist-agentic-rl-tools.md) all assume you control the training loop. Agent Lightning answers a different question: **how do you RL-train an agent your team has already built in LangGraph or AutoGen, without rewriting it as a training script?** This matters because most production agent code in 2026 lives in agent frameworks ([110-langgraph](110-langgraph.md), AutoGen, [109-smolagents](109-smolagents.md)) — and those frameworks were written for inference, not training.

## Problem it solves

1. **Training-inference code divergence.** Researchers maintain one codebase for the agent (LangChain/AutoGen) and another for RL training (TRL/OpenRLHF). They drift; production behavior diverges from trained behavior.
2. **Trajectory capture is invasive.** To record `(state, action, reward)` for RL, naive approaches require monkey-patching the framework's LLM client.
3. **Multi-step credit assignment** in framework-mediated agents (where the framework hides the prompt-completion loop) is opaque.
4. **Heterogeneous agent stacks.** A real product mixes a LangGraph planner with a custom Python tool layer and an AutoGen sub-agent — no single RL framework speaks all three.

## Core idea in one paragraph

Add a **proxy LLM endpoint** that the agent code points at instead of OpenAI/Anthropic. The proxy logs every `(prompt, completion)` pair, tags it with a trajectory ID, and passes the call to the actual model server. At trajectory end (signaled by an explicit `agent.done()` call or end-of-task heuristic), the trajectory is graded by a reward function and queued for an RL update. The RL trainer is *separate* — it consumes trajectories from a replay buffer and produces new model weights, which the proxy hot-reloads. The agent code remains literally unchanged.

## Mechanism (step by step)

### (a) The proxy interception layer

```python
# What the agent sees:
client = OpenAI(base_url="http://lightning-proxy:8000/v1")
response = client.chat.completions.create(...)

# What the proxy does:
def proxy_call(req):
    traj_id = current_trajectory_id()  # threadlocal
    completion = backend_model.generate(req)
    log(traj_id, req, completion)      # for RL replay
    return completion
```

`current_trajectory_id()` is a context-managed counter. Boundaries can be marked with `with lightning.trajectory():` blocks or inferred from session/conversation IDs.

### (b) Reward attachment

```python
@lightning.reward_fn
def grade(trajectory):
    # trajectory.steps is the list of (prompt, completion) pairs
    # task_outcome is supplied by the agent host
    return 1.0 if task_outcome.success else 0.0
```

Reward is computed *outside* the agent code. The trajectory carries side-channel metadata (e.g. `task_outcome`) populated by the host application.

### (c) Hierarchical credit (for multi-agent)

When multiple agent processes participate, each emits a *sub-trajectory* with its own reward; Agent Lightning composes them via a configurable aggregator (sum, max, weighted). This is how it handles a LangGraph orchestrator + AutoGen sub-agent setup: each is RL'd against its own slice with the right reward signal.

### (d) Training step

The RL trainer (built on TRL/Verl) pops a batch of trajectories, runs GRPO or PPO updates, writes the new LoRA adapter to a shared volume. The proxy reloads adapters on file-change events — typically <500ms. Agents in flight at hot-reload time complete with old weights; new trajectories use new weights. *No agent process restarts.*

### (e) Selective trainability

Mark which spans of a trajectory are *trainable* (the policy's outputs) vs *frozen* (e.g. responses from a sub-agent backed by a different model, or canned tool descriptions). The masking is applied at the loss level, identical in spirit to ARTIST's tool-result masking ([102-artist-agentic-rl-tools](102-artist-agentic-rl-tools.md)).

## Empirical results

The paper demonstrates RL-training of agents originally built in **LangChain**, **AutoGen**, and **custom Python**, with single-digit lines of integration code per framework. Headline empirical claim (paraphrased from the paper): on the same end-task benchmark, an agent originally written in LangChain and RL-trained via Agent Lightning matches the performance of an end-to-end re-implementation in TRL — at a small fraction of the engineering effort.

The paper does *not* claim a new SOTA on any benchmark. The contribution is **infrastructure**, not algorithm.

## Variants and ablations

- **Single-agent vs multi-agent RL:** multi-agent setups benefit most — the engineering cost saved (vs hand-coded multi-agent training) is the largest.
- **LoRA vs full-parameter:** LoRA is the default because hot-reload of full weights is operationally heavy.
- **Proxy overhead:** ~5–15ms latency per call; negligible vs LLM inference. Throughput drop ~3% under load.
- **Trajectory capture sampling rate:** at high traffic, only sample a fraction of trajectories (e.g. 5%) to keep replay buffer manageable; reward signal sparser but unbiased.

## Failure modes and limitations

1. **Stateless-prompt assumption.** Agents that depend on opaque framework-internal state (e.g. AutoGen's group-chat memory) need explicit serialization for the proxy to capture the full prompt context.
2. **Reward signal latency.** Long-horizon trajectories (hours) sit in the buffer waiting for grading — RL throughput suffers.
3. **Hot-reload race conditions.** Concurrent in-flight trajectories with mid-trajectory weight reload yield ambiguous attribution; the paper recommends locking reloads to inter-trajectory windows.
4. **Reward function design is still on you.** The framework solves *how to train*, not *what to optimize* — most teams' bottleneck is reward, not infra.
5. **Vendor lock-in pressure.** Some hosted agents (closed-source SaaS) refuse the proxy redirect; only self-hosted or BYO-key setups apply.

## When to use, when not

**Use Agent Lightning when** you have an agent already in production (or near-production) that you want to keep improving via RL without rewriting it; especially compelling for multi-agent stacks built across multiple frameworks. Also useful when ML engineers and product engineers are different teams: the proxy is a clean contract.

**Don't reach for it** when starting from scratch — building directly in TRL/Verl/OpenRLHF is fine. Also avoid when reward design is genuinely the open problem; no infra fixes that.

## Implications for harness engineering

- **The proxy *is* the harness.** Every agent harness already has a model client; making it interceptable for capture/replay is a one-time cost that buys both [24-observability-tracing](24-observability-tracing.md) and RL.
- **Reward functions become first-class harness artifacts.** Like hooks ([05-hooks](05-hooks.md)), they live next to the agent definition.
- **Hot-reload weight updates** make "the model your agent uses" a runtime variable — opens the door to per-tenant fine-tunes, time-of-day model switching, and continuous learning loops.
- **Multi-agent reward composition** is the lever you'd want for [02-subagent-delegation](02-subagent-delegation.md): each subagent can have a slice of the reward signal that reflects its scope.

## Connections to other work in this corpus

- **[101-ragen](101-ragen.md), [102-artist-agentic-rl-tools](102-artist-agentic-rl-tools.md):** RAGEN and ARTIST provide the **algorithm**; Agent Lightning provides the **plumbing** to apply that algorithm to existing code.
- **[42-langchain-deep-agents](42-langchain-deep-agents.md), [110-langgraph](110-langgraph.md), [109-smolagents](109-smolagents.md):** the frameworks Lightning trains *into*.
- **[24-observability-tracing](24-observability-tracing.md):** the proxy is also a tracing point; one architecture, two payoffs.
- **[55-hermes-agent-self-improving](55-hermes-agent-self-improving.md):** Hermes is a self-improvement *loop*; Lightning is the *infra* a Hermes-style loop would use.

## Key takeaways

1. **You don't need to rewrite your agent to RL it.** Proxy + reward function + RL trainer is a contract that fits any framework.
2. **Trajectory capture and selective masking** generalize across frameworks; they are the lingua franca of multi-framework RL.
3. **Hot-reloading LoRA adapters** without restarts makes "training" continuous rather than episodic.
4. **The hard problem remaining is reward design**, not infrastructure.
5. **Multi-agent RL is the largest engineering-cost win** — Agent Lightning collapses what would otherwise be N coupled training pipelines.

## References

- *Agent Lightning: Train ANY AI Agents with Reinforcement Learning.* arXiv:2508.03680. https://arxiv.org/abs/2508.03680
- Microsoft Research blog: https://www.microsoft.com/en-us/research/blog/agent-lightning-adding-reinforcement-learning-to-ai-agents-without-code-rewrites/
- Underlying RL stacks: TRL (https://github.com/huggingface/trl), Verl (https://github.com/volcengine/verl).
- Companion frameworks targeted: LangChain, AutoGen, smolagents.
