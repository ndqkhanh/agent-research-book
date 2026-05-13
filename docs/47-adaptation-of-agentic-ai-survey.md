# 47 — Adaptation of Agentic AI: Post-Training, Memory, Skills

**Definition.** "Adaptation of Agentic AI: A Survey of Post-Training, Memory, and Skills" (arXiv:2512.16301, March 2026, 89 pages) organizes the fast-growing field of *agent adaptation* — all the ways an agent system can be improved **after pretraining** — into a **four-paradigm taxonomy** across two axes: agent-side vs tool-side, and tool-execution-signaled vs agent-output-signaled.

## Problem it solves

The techniques a team can use to make their agent better after deployment are proliferating: SFT on trajectories, preference optimization, RL with verifiable rewards, retrieval-augmented memory, skill libraries, persistent tool adaptations, subagent curricula. These lived in scattered papers. The 89-page survey puts them on a common map, which helps teams:

- Pick techniques relevant to their constraints (own model vs. API-only, feedback channel available, compute budget).
- Identify uncovered combinations (interesting research directions).
- Communicate which paradigm a proposed change belongs to.

## The Four-Paradigm Taxonomy

### Agent-side adaptations

**A1. Tool-Execution-Signaled.** Improve the agent using signals from tool execution — test pass/fail, environment rewards, runtime errors. Methods: SFT on successful trajectories, preference optimization using pairs, RL with verifiable rewards. Example: GLM-5 ([31-glm-5-agentic-engineering.md](31-glm-5-agentic-engineering.md)) style post-training on agent trajectories.

**A2. Agent-Output-Signaled.** Improve the agent using signals from its own outputs — self-evaluation, LLM-judge scores, consistency checks. Methods: same classes (SFT, DPO, RL), but reward is produced without tool-execution ground truth. Example: Reflexion ([14-reflexion.md](14-reflexion.md))-style verbal reinforcement scaled into a training dataset.

### Tool-side adaptations

**T1. Agent-Agnostic.** Build reusable modules that any agent can mount — memory services, skill libraries, MCP servers with pre-indexed domain content. Example: Cloudflare Agent Memory; persistent skill libraries distributed across teams.

**T2. Agent-Supervised.** Use the agent's outputs to train tool-side components — memory systems that learn what to store, skill libraries that learn what to promote into reusable skills, lightweight subagents trained on the main agent's delegations. Example: Voyager-style skill library ([19-voyager-skill-libraries.md](19-voyager-skill-libraries.md)) where the agent writes and verifies skills.

## Concrete pattern — picking the right paradigm

A decision tree for teams:

```
Do you own the base model?
  Yes → Agent-side paradigms available (A1, A2).
    Verifiable reward signal?
      Yes → A1 (tool-execution-signaled). Preferred when strong.
      No  → A2 (agent-output-signaled). Use LLM-judge carefully.
  No  → Tool-side paradigms (T1, T2) or prompt/skill changes.
    Is the capability reusable across agents?
      Yes → T1 (agent-agnostic): build or adopt shared modules.
      No  → T2 (agent-supervised): train tool-side on your agent's traces.
```

This is decision support, not dogma; many real systems span paradigms.

## Variants & related techniques

- **Post-training (A1/A2).** Covered in the survey across SFT, DPO, RLAIF, RLVR; see GLM-5 ([31-glm-5-agentic-engineering.md](31-glm-5-agentic-engineering.md)).
- **Memory (T1/T2).** Covered extensively; see [09-memory-files.md](09-memory-files.md) and "Memory in the Age of AI Agents" (arXiv:2512.13564).
- **Skills (T1/T2).** Skills as agent-agnostic modules ([04-skills.md](04-skills.md)); Voyager-style growing libraries ([19-voyager-skill-libraries.md](19-voyager-skill-libraries.md)).
- **Autogenesis** ([36-autogenesis-self-evolving-agents.md](36-autogenesis-self-evolving-agents.md)) — protocol-level substrate that can host any of the four paradigms.
- **Reflexion** ([14-reflexion.md](14-reflexion.md)) — prototype of A2 with no gradient updates.

## Failure modes & anti-patterns

- **Reward hacking in A1.** Test-pass rewards create models that write tests that pass themselves. Fix: hold-out tests, LLM judge over trajectories.
- **Self-reference collapse in A2.** Agent judges its own output; preferences reinforce existing biases. Fix: diversified judges (different models, different prompts).
- **Premature T1 investment.** Building an elaborate agent-agnostic memory service before you know what one agent needs. Fix: T2 first on one agent; generalize to T1 later.
- **Ignoring cost of adaptation loops.** A1 / A2 loops are training-compute-expensive. Budget carefully.
- **Copying survey taxonomy without tailoring.** Four paradigms is a map, not a plan. Your system's constraints drive which paradigm fits.

## When to use (and when not to)

The survey is **reference material** for:

- Teams designing their agent-adaptation roadmap.
- Researchers surveying the field for gaps.
- Engineers justifying investment in one paradigm over another to stakeholders.

Less useful for:

- Small teams with limited adaptation ambition — use prompt + skills.
- One-off tools where adaptation isn't a design axis.

## References

- arXiv:2512.16301 — "Adaptation of Agentic AI: A Survey of Post-Training, Memory, and Skills" (March 2026). <https://arxiv.org/abs/2512.16301>
- GLM-5 (arXiv:2602.15763). <https://arxiv.org/abs/2602.15763>
- Voyager (arXiv:2305.16291). <https://arxiv.org/abs/2305.16291>
- Reflexion (arXiv:2303.11366). <https://arxiv.org/abs/2303.11366>
- "Memory in the Age of AI Agents: A Survey" (arXiv:2512.13564). <https://arxiv.org/abs/2512.13564>
