# 114 — May-2026 Landscape Update: Synthesis of Chapters 100–113

**This chapter synthesizes the new entries (100–113) and updates the April-2026 landscape ([76-ten-links-synthesis](76-ten-links-synthesis.md), [99-papers-deep-dive-synthesis](99-papers-deep-dive-synthesis.md)) for May 2026.** Treat it as a "what changed since file 99" map.

## TLDR

The 12 months since the Lyra synthesis split visibly into three directions:

1. **Reasoning models became the substrate.** [100-deepseek-r1-rl-reasoning](100-deepseek-r1-rl-reasoning.md)'s pure-RL recipe is now the open default; harnesses budget reasoning tokens, not just user-facing tokens.
2. **Multi-turn RL on agents shipped** ([101-ragen](101-ragen.md), [102-artist-agentic-rl-tools](102-artist-agentic-rl-tools.md), [103-agent-lightning](103-agent-lightning.md)) — and produced a diagnostic vocabulary (the *Echo Trap*, *tool-result masking*, the *proxy-as-training-surface*).
3. **The framework stack consolidated** around three production options — [110-langgraph](110-langgraph.md) (explicit graph), [108-openhands-codeact](108-openhands-codeact.md) / [109-smolagents](109-smolagents.md) (code-as-action), and [111-magentic-one](111-magentic-one.md) (orchestrator-+-specialists) — with [105-letta-stateful-agents](105-letta-stateful-agents.md) and [104-mem0-production-memory](104-mem0-production-memory.md) defining the memory axis.

OS-level / browser computer-use ([106-computer-use-agents](106-computer-use-agents.md), [107-browser-use](107-browser-use.md)) is the steepest open capability curve. Evaluation has caught up: [112-browsecomp](112-browsecomp.md) and the test-time-compute literature ([113-test-time-compute-agents](113-test-time-compute-agents.md)) now constrain *which* gains are real.

## What changed since file 99 (April 2026 → May 2026)

| Axis | April 2026 (file 99) | May 2026 (this update) |
|------|----------------------|------------------------|
| Reasoning model recipe | R1 + GRPO documented | Now a baseline assumption; harnesses presume `<think>` channels |
| Multi-turn agentic RL | Mostly closed-source | RAGEN, ARTIST, Agent Lightning open and reproducible; *Echo Trap* a named pattern |
| Memory architecture | Memory files / claude-mem | Mem0 (extract+consolidate) and Letta (self-managed) split the axis |
| Computer use | Anthropic 22% OSWorld | Sonnet 4.5 at 61.4%; closed agents past 80%; Browser-Use OSS leading on web |
| Multi-agent | ChatDev / MetaGPT roles | Magentic-One generalist orchestrator; subgraphs in LangGraph |
| Evaluation | OSWorld, ClawBench | + BrowseComp for deep research; AutoGenBench for variance discipline |
| TTC | Per-task BoN | Adaptive 4-dimensional lever; memory as TTC infra |

## The four axes the new chapters explore

### Axis 1 — Training (chapters 100–103)

[100-deepseek-r1-rl-reasoning](100-deepseek-r1-rl-reasoning.md) is the upstream supply: pure RL + verifiable reward elicits reasoning. [101-ragen](101-ragen.md) extends the recipe to multi-turn agents and *names the failure modes*. [102-artist-agentic-rl-tools](102-artist-agentic-rl-tools.md) closes the loop with real tools. [103-agent-lightning](103-agent-lightning.md) makes any of the above retrofittable to existing codebases.

The harness consequence: **agent training is now plumbing-feasible**. A team with a verifier and a sandbox can RL their own agent in days. Reward design — not infra — is the open problem.

### Axis 2 — Memory and statefulness (chapters 104–105)

[104-mem0-production-memory](104-mem0-production-memory.md) (harness-driven extraction + consolidation) and [105-letta-stateful-agents](105-letta-stateful-agents.md) (self-managed memory tools, persistent identity) are architectural counterparts. The choice is whether **the harness or the model** decides what to remember.

This is also a privacy-and-compliance axis: persisted user data needs first-class deletion, scoping, and audit. The existence of [22-guardrails-prompt-injection](22-guardrails-prompt-injection.md) attacks against memory makes this a security axis too.

### Axis 3 — Surface (chapters 106–107)

OS-level pixels ([106-computer-use-agents](106-computer-use-agents.md)) and DOM-aware browsers ([107-browser-use](107-browser-use.md)) are the two most general agent surfaces ever shipped. The 22% → 61.4% → 80%+ trajectory on OSWorld is the steepest open capability curve of 2025–26. **Vision grounding** (not reasoning) is the bottleneck; **HITL gates** (not training) are the safety mechanism.

### Axis 4 — Frameworks and orchestration (chapters 108–111)

The frameworks consolidated:

- **Code-as-action**: [108-openhands-codeact](108-openhands-codeact.md) (heavy, sandboxed) and [109-smolagents](109-smolagents.md) (minimal, pluggable) are the OSS defaults.
- **Explicit state graph**: [110-langgraph](110-langgraph.md) is the production-enterprise pick.
- **Orchestrator + specialists**: [111-magentic-one](111-magentic-one.md) is the generalist multi-agent reference.

These are not interchangeable; they cover different points in a 3-axis space (control flow explicitness × action format × multi-agent topology).

## Cross-cutting themes

### Theme A — Verifier-centrism

The thread connecting [100-deepseek-r1-rl-reasoning](100-deepseek-r1-rl-reasoning.md), [101-ragen](101-ragen.md), [97-qwen-prm](97-qwen-prm.md), [112-browsecomp](112-browsecomp.md), [113-test-time-compute-agents](113-test-time-compute-agents.md): **verifier quality is the upper bound on every other axis**. RL needs verifiable rewards; TTC needs reliable selection; eval needs trustworthy grading. Investing in verifiers (PRMs, judges, sandboxed unit tests) compounds across all of them.

### Theme B — Sandboxing as harness primitive

[108-openhands-codeact](108-openhands-codeact.md), [109-smolagents](109-smolagents.md), [102-artist-agentic-rl-tools](102-artist-agentic-rl-tools.md), [106-computer-use-agents](106-computer-use-agents.md), [107-browser-use](107-browser-use.md), [113-test-time-compute-agents](113-test-time-compute-agents.md) all need sandboxes. *Reversibility and isolation* are now infrastructure, not afterthoughts.

### Theme C — HITL as gradient

[111-magentic-one](111-magentic-one.md) (Magentic-UI), [110-langgraph](110-langgraph.md) (interrupts), [106-computer-use-agents](106-computer-use-agents.md) (login/payment pauses) all converge on the same model: **HITL is a runtime annotation**, not a special mode. The unified vocabulary inherits from [03-plan-mode](03-plan-mode.md), [06-permission-modes](06-permission-modes.md), [23-human-in-the-loop](23-human-in-the-loop.md).

### Theme D — Memory as TTC infrastructure

[81-reasoningbank](81-reasoningbank.md), [104-mem0-production-memory](104-mem0-production-memory.md), [113-test-time-compute-agents](113-test-time-compute-agents.md) (agentic-coding TTC) jointly argue: **persistent memory of past trajectories cheapens future selection**. The right memory makes 1 well-curated rollout beat 5 raw rollouts.

## What's still open (May 2026)

1. **Reward design at the open frontier.** Infra for agentic RL is solved; *what to reward* in open-ended domains (creative writing, customer support, research synthesis) is not.
2. **Verifiers for non-verifiable domains.** The PRM literature is math-heavy; reliable judges for taste, ethics, and writing quality are scarce.
3. **Multi-tenant memory and security.** Mem0/Letta-class systems at hyperscale need crisp tenancy + GDPR-class deletion.
4. **OS-level computer use general safety.** Pixel-level agents on real desktops + prompt-injection-via-screen-text is a near-future incident category.
5. **Cross-framework standards.** [07-model-context-protocol](07-model-context-protocol.md) covers tools but not memory, not RL training contracts; convergence pressure is mounting.
6. **TTC routers.** Adaptive selection of parallel-vs-sequential-vs-verifier-vs-diversify, per-task, is open infrastructure.
7. **Long-horizon agents.** Autonomy windows are still single-task or single-session; multi-day, multi-stakeholder agent operation is largely unsolved.

## Implications for the harness engineer in May 2026

- **Default stack** for new OSS coding agent: [108-openhands-codeact](108-openhands-codeact.md) on Claude Sonnet 4.x or Qwen2.5-Coder-32B; sandbox via Docker; memory via [104-mem0-production-memory](104-mem0-production-memory.md); orchestration via [110-langgraph](110-langgraph.md) only when control flow demands it; otherwise [109-smolagents](109-smolagents.md).
- **For research agents**: Browser-Use ([107-browser-use](107-browser-use.md)) + reasoning-model backbone + verifier-guided self-check; benchmark against [112-browsecomp](112-browsecomp.md).
- **For learning loops**: Agent Lightning ([103-agent-lightning](103-agent-lightning.md)) over your existing agent if it's already in a framework; otherwise RAGEN-style stack ([101-ragen](101-ragen.md)) for greenfield.
- **For multi-agent**: LangGraph subgraphs OR Magentic-One pattern, not ad-hoc orchestration. Audit logs and HITL gates from day one.

## Read-across to existing chapters

- The classic harness layer ([01-agent-loop-architecture](01-agent-loop-architecture.md), [03-plan-mode](03-plan-mode.md), [05-hooks](05-hooks.md), [06-permission-modes](06-permission-modes.md), [07-model-context-protocol](07-model-context-protocol.md), [08-context-compaction](08-context-compaction.md), [09-memory-files](09-memory-files.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [12-todo-scratchpad-state](12-todo-scratchpad-state.md), [23-human-in-the-loop](23-human-in-the-loop.md), [24-observability-tracing](24-observability-tracing.md)) is **largely intact**. The new chapters are mostly *instantiations* of those primitives at scale, with two genuine additions: **agent training as a runtime concern** ([103-agent-lightning](103-agent-lightning.md)) and **memory invalidation as a security boundary** ([104-mem0-production-memory](104-mem0-production-memory.md)).
- The April-2026 file 99 synthesis remains valid; this update is **additive**, not corrective.

## Key takeaways

1. **Reasoning is now substrate, not feature.** Plan harnesses around it.
2. **Agentic RL is plumbing-feasible.** Reward design is the bottleneck.
3. **Memory architecture has bifurcated.** Pick harness-driven (Mem0) or model-driven (Letta) deliberately.
4. **Code-as-action is the OSS coding-agent default.** JSON tool-call agents persist for thin tasks.
5. **HITL is a runtime annotation**, not a separate mode — pattern stable across LangGraph, Magentic-One, and the computer-use stacks.
6. **Verifier quality is the universal cap.** Invest there first.
7. **The 22% → 61% → 80%+ OSWorld curve** is the headline open-capability story of the year.

## References (representative; full citations in chapters 100–113)

- Chapters 100–113 of this corpus.
- [76-ten-links-synthesis](76-ten-links-synthesis.md) and [99-papers-deep-dive-synthesis](99-papers-deep-dive-synthesis.md) — the prior synthesis vantage points.
- DeepSeek-AI (2025), arXiv:2501.12948.
- Wang et al. (2024), CodeAct, arXiv:2402.01030.
- Chhikara et al. (2025), Mem0, arXiv:2504.19413.
- Wei et al. (2025), BrowseComp, arXiv:2504.12516.
- Wang et al. (2025), RAGEN, arXiv:2504.20073.
- Fourney et al. (2024), Magentic-One, arXiv:2411.04468.
- Snell et al. (2024), TTC, arXiv:2408.03314.
