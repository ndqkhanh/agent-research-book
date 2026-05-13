# 173 — Offline-Simulation Skill Discovery: A Third Feedback Signal Source

**Source.** Paiheng Xu, Gang Wu, Xiang Chen, Tong Yu, Chang Xiao, Franck Dernoncourt, Tianyi Zhou, Wei Ai, Viswanathan Swaminathan. *Skill Discovery for Software Scripting Automation via Offline Simulations with LLMs*. arXiv:[2504.20406](https://arxiv.org/abs/2504.20406) (v1 Apr 2025; v2 Feb 2026). Adobe Research + DeepSeek + Meta + UMD.

**One-paragraph thesis.** Offline simulators are a *third* feedback signal source for skill discovery, sitting between EvoSkill's ground-truth oracle ([168](168-evoskill-coding-agent-skill-discovery.md)) and CoEvoSkills' surrogate verifier ([169](169-coevoskills-co-evolutionary-verification.md)). For software-automation surfaces where live tool access is expensive, rate-limited, or unsafe — Photoshop, Illustrator, Office, IDE plugins — the system pre-validates LLM-proposed scripts against a controlled simulation harness before any production execution. The paper shows headline gains in success rate, response time, and runtime token cost compared to runtime code generation. The contribution that's specifically interesting for harness designers is the **API-relationship Graph Neural Network**: instead of treating the API surface as a flat list, the system models inter-API dependencies and uses the graph at task-creation time to generate semantically coherent multi-API tasks the simulator can validate.

## §1 — Where this paper sits in the design space

`docs/171` plotted four corners of the skill-evolution design space along axis 1 (feedback signal source):

```
ground-truth ←——→ surrogate ←——→ adversarial ←——→ no signal
oracle           verifier        self-play        (judgement only)
```

This paper adds **a fifth point** between *ground-truth oracle* and *surrogate verifier*:

```
ground-truth ← offline simulator ← surrogate ← adversarial ← no signal
              (this paper)         verifier   self-play     (judgement)
```

An *offline simulator* is:

- **More faithful** than a surrogate verifier — it executes the candidate skill against a real (or near-real) implementation of the target environment.
- **Less authoritative** than a ground-truth oracle — the simulator can drift from production over time, and edge cases the simulator masks may surface in deployment.
- **Cheaper** than either — no production rate limits, no licence costs per call, no human-in-the-loop confirmation for destructive operations.

The trade-off Polaris and Lyra care about is *fidelity vs. cost*, and the offline-simulator point is where that trade-off is tunable per domain.

## §2 — Mechanism

The system's pipeline has two stages, both LLM-driven, with a graph-structured input layer connecting them.

### §2.1 — Task creation via API exploration

Adobe Illustrator's scripting surface is several thousand functions across a dozen object families (Documents, Pages, PathItems, TextFrames, GraphicStyles, etc.). Naive task generation — "write a script that does X" with no structural prior — produces tasks the simulator either trivially solves or trivially rejects. The paper instead:

1. Builds an **API-relationship graph** offline from the Illustrator scripting docs. Nodes are functions; edges encode "this function's output is consumed by that function's input" plus reference relationships ("this function operates on objects of type X").
2. Trains a **GNN** that scores API pairs / triples / longer sequences for semantic coherence.
3. At task-creation time, samples connected sub-graphs from the API graph and asks the LLM to emit a natural-language task description for the sampled APIs. This guarantees the generated tasks exercise multi-API patterns rather than single-call lookups.

The GNN is the load-bearing piece. Without it, task quality degrades to "rotate the selected layer" — interesting on its own, but not the kind of compound script that demonstrates skill.

### §2.2 — Skill generation with execution feedback

For each generated task:

1. The LLM proposes a candidate script.
2. The script is executed inside the **offline simulator** (a headless, sandboxed Illustrator implementation that mirrors the live application's scripting surface).
3. The simulator returns success / fail plus a state diff (which document properties changed, which throw expected vs. unexpected errors).
4. On failure, the diff plus the error trace is fed back to the LLM for refinement.
5. On success across N independent task variants exercising the same API-pattern, the script is promoted into the **verified skill library**.

The promotion bar — multi-instance success across task variants — is what distinguishes this from EvoSkill's per-task Pareto search. EvoSkill keeps the best-performing program *for each task*; this paper keeps the most-generalisable program *across a task class*.

## §3 — Results

The paper reports significant gains in (a) automation success rate on a held-out task suite, (b) response time vs. runtime code generation, and (c) total runtime token cost. The exact numerical headline is not stated in the abstract excerpt available; harness implementers should pull the full PDF for the precise pp gains. The qualitative claim is robust: **runtime code generation is wasteful when the API surface is stable and a simulator exists**.

## §4 — Comparison vs. AutoSkill / EvoSkill / CoEvoSkills

|  | **AutoSkill** ([167](167-autoskill-experience-driven-lifelong-learning.md)) | **EvoSkill** ([168](168-evoskill-coding-agent-skill-discovery.md)) | **CoEvoSkills** ([169](169-coevoskills-co-evolutionary-verification.md)) | **This paper** (2504.20406) |
|---|---|---|---|---|
| Feedback signal | LLM judge (no GT) | Ground-truth task scores | Surrogate verifier (info-isolated LLM) | **Offline simulator** (env replica) |
| Domain | Dialogue / personal preference | Coding agents (any task with tests) | Coding agents (any task) | Software automation (Adobe, Office) |
| Promotion criterion | Durability marker | Pareto frontier on benchmark | Generator–Verifier consensus | **Multi-instance variant success** |
| Task generation | User-driven | Benchmark-driven | Benchmark-driven | **GNN-sampled API sub-graph** |
| What's reusable beyond the paper's domain | The four-axis comparison rubric | The Pareto-frontier search | The information-isolated verifier | **The simulator-as-gate pattern + GNN-typed task generation** |

## §5 — What harness implementers should steal

Three patterns transfer cleanly outside the Adobe-scripting domain:

1. **Offline simulator as a gate, not a replacement.** Polaris and Lyra both already have ground-truth gates (TDD plugin, ClaimGate, `BL-PROMOTE-SKILL`). An offline simulator is an additional pre-flight check that filters out *most* bad candidates before they touch the expensive ground-truth path. The simulator does not replace the ground-truth gate; it *protects* it.
2. **API-relationship graph as a typed task generator.** This is the strongest single contribution. Most skill-discovery papers sample tasks from a benchmark or from user logs; this paper samples them from a *typed graph of the tool surface*. The pattern generalises: any agent harness with a non-trivial tool catalog (`polaris-mcp`, `lyra_skills`) can build the equivalent graph and use it to drive failure-rich task generation. This is the load-bearing extension of [docs/171](171-skill-self-evolution-2026-synthesis.md)'s "skills are files" thesis: *the API graph is also a file*.
3. **Promote on multi-instance variant success, not per-instance pass.** The simulator makes this cheap. The promotion criterion shifts from "this script solved task N" (over-fits) to "this skill solved a class of tasks the GNN considers semantically coherent" (generalises).

## §6 — Polaris integration slot

```text
packages/polaris-skills/src/polaris_skills/research/
  offline_sim_gate.py       # NEW — wraps a domain-specific simulator;
                            # composes with v2.2 three-gate AND as a
                            # pre-flight stage before ClaimGate fires.
  api_graph.py              # NEW — typed API-relationship graph;
                            # sub-graph sampler for failure-driven task
                            # generation in v2.3 P33's tree-search
                            # experiment manager.
```

Bright-line addition (proposed):

```
BL-OFFLINE-SIM-DRIFT      The offline simulator's behaviour cannot
                          drift from production by more than a
                          configured tolerance without re-validation.
                          Detected drift triggers HITL review of any
                          skills promoted since the last check.
```

## §7 — Lyra integration slot

```text
packages/lyra-core/src/lyra_core/skills/
  offline_validator.py      # NEW — Sandbox-typed skill validator;
                            # composes with the existing curator.
  api_graph.py              # NEW — typed graph for the tool catalog.
```

The pattern is particularly natural inside Lyra's existing Sandbox abstraction (`lyra-core/terminal/`): the offline simulator is just another sandbox backend whose output is a state diff rather than a process exit code.

## §8 — Where this fits

Read after the four-corners synthesis at [171](171-skill-self-evolution-2026-synthesis.md). Pairs with:

- [168 — EvoSkill](168-evoskill-coding-agent-skill-discovery.md) — the *ground-truth* corner this paper extends.
- [169 — CoEvoSkills](169-coevoskills-co-evolutionary-verification.md) — the *surrogate-verifier* corner; offline simulator sits between this and EvoSkill.
- [171 — Skill Self-Evolution Synthesis](171-skill-self-evolution-2026-synthesis.md) — extend the four-corner table with this paper as the fifth corner.
- [177 — Strongest 2026 Techniques Synthesis](177-skills-discovery-curator-strongest-2026-techniques.md) — folds the offline-sim corner into the recommended stack.
