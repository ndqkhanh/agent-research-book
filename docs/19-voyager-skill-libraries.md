# 19 — Voyager-Style Skill Libraries

**Definition.** Voyager (Wang et al. 2023) is an open-ended agent that learns in Minecraft by (a) an automatic curriculum that proposes ever-harder tasks, (b) an iterative code-prompting loop that writes executable skills, and (c) a growing *skill library* of vetted code snippets the agent can retrieve and compose later. The architecture generalizes well beyond Minecraft — anywhere an agent can write code, verify it against an environment, and want to reuse it.

## Problem it solves

Most agents are amnesiac. They solve a task, discard everything they figured out, and start from scratch next time. This is especially wasteful in environments with a long tail of tasks that share primitives — "mine stone" is a prerequisite for "craft furnace", "find food", "build shelter". A skill library lets the agent *accumulate capability* without any weight update — capability grows by code, not by gradient.

Voyager also solves the exploration problem in open-ended tasks. Left to its own devices, a model plateaus on easy-but-pointless activities. An automatic curriculum that asks "what could this agent do next given what it can already do?" pushes it up the complexity ladder.

## Mechanism

Three interacting components:

1. **Automatic curriculum.** An LLM prompted with the agent's current state, inventory, and skill library proposes the next task — one that is *achievable but non-trivial* given current capabilities. Bias toward novelty: don't repeat recent tasks.

2. **Iterative prompting with environment feedback.** For each task, the agent writes code (JavaScript for Mineflayer in the original paper) that attempts it. The code runs in the environment; errors and observations are fed back; the code is refined. Iteration continues until a self-verification model deems the task solved.

3. **Skill library.** Verified solutions are saved as callable functions, each indexed by an embedding of its docstring. On a new task, relevant skills are retrieved by similarity and included in the code-generation context — so the agent writes *compositions* of existing skills rather than reinventing each primitive.

Key loop:

```
while True:
    task = curriculum.propose(state, skill_library)
    code = None
    for attempt in range(max_attempts):
        context = retrieve_similar_skills(task, skill_library, k=5)
        code = model.write_code(task, state, context, prior_errors=code_errors)
        result = execute_in_env(code)
        if verifier.task_complete(task, result):
            break
        code_errors.append(result.error)
    if success:
        skill_library.add(task.name, code, docstring=model.describe(code))
    state = result.new_state
```

The paper reports Voyager unlocking ~3× more tech-tree items than prior Minecraft agents and transferring its skill library to entirely new worlds.

## Concrete pattern

Skill library entry:

```python
# skill: mine_three_cobblestone
# description: Mine three cobblestone blocks using a wooden pickaxe,
#              assuming pickaxe is in inventory and stone is nearby.
async def mine_three_cobblestone(bot):
    await equip(bot, "wooden_pickaxe")
    for _ in range(3):
        stone = find_block(bot, "stone", max_distance=32)
        if not stone:
            await explore(bot, direction="any", distance=16)
            continue
        await mine_block(bot, stone)
```

A later task — "craft a furnace" — retrieves `mine_three_cobblestone` plus `craft_recipe`, and the code generator composes:

```python
async def craft_furnace(bot):
    await mine_three_cobblestone(bot)
    await craft_recipe(bot, "furnace", count=1)
```

No new primitive was learned; a new *composition* was added as its own skill for the next level up.

## Variants & related techniques

- **Claude Code Skills** ([04-skills.md](04-skills.md)) are the production cousin: authored by humans (not discovered by agents) but same "indexed, retrieved, composed" pattern.
- **ACT-R / SOAR** (classical cognitive architectures) have long used procedural memory libraries in similar ways; Voyager is the LLM-era incarnation.
- **AgentBench / environments with verifiable rewards** give Voyager-style agents a natural home; without verifiable completion, the skill-library quality degrades.
- **CodeAct** (arXiv:2402.01030) has the agent *always* produce code as actions, overlapping with Voyager's core loop but without curriculum or skill library.
- **Retrieval-augmented tool selection** at the agent level — tools indexed and retrieved like skills — is a shared idea.

## Failure modes & anti-patterns

- **Skill library poisoning.** An incorrect skill gets added; subsequent tasks build on it and fail. Fix: strict verification before saving; periodic audit; versioning.
- **Over-retrieval.** Top-k retrieval pulls in irrelevant skills and bloats context. Fix: threshold on similarity; curate descriptions carefully.
- **Under-retrieval.** The agent rewrites a skill already in the library because the query didn't match. Fix: rich docstrings; index on multiple paraphrases; use hybrid search (semantic + keyword).
- **Curriculum collapse.** The curriculum proposes the same task family over and over because it's "just above" current skills. Fix: novelty penalty on recent tasks.
- **Curriculum runaway.** The curriculum proposes tasks far beyond current ability; the agent fails forever. Fix: heuristic scaffolding ("propose a task that reuses at least 1 existing skill").
- **Verifier weakness.** The environment provides no reliable completion signal and the verifier model approves failures. Fix: use environment rewards wherever possible; LLM verification is a last resort.
- **Closed-domain brittleness.** Voyager relies on a scriptable environment (Mineflayer). Harder to apply in environments without stable APIs.

## When to use (and when not to)

**Use** Voyager-style skill libraries when:

- The environment is stable and scriptable (a codebase, a game, a workflow automation platform).
- Tasks share primitives — investing in shared skills pays off.
- You have a reliable verification signal (tests, game state, task completion API).
- You want the agent's capability to grow over time without retraining.

**Don't** use them when:

- Tasks are one-offs with no shared substructure.
- Verification is unreliable — bad skills poison the library fast.
- The environment's API is unstable, so saved skills rot.
- The problem is reasoning-heavy and not code-expressible.

## References

- Wang et al., "Voyager: An Open-Ended Embodied Agent with Large Language Models", arXiv:2305.16291 — <https://arxiv.org/abs/2305.16291>
- Voyager project site — <https://voyager.minedojo.org/>
- Wang et al., "Executable Code Actions Elicit Better LLM Agents" (CodeAct), arXiv:2402.01030 — <https://arxiv.org/abs/2402.01030>
- Lilian Weng, "LLM Powered Autonomous Agents" — <https://lilianweng.github.io/posts/2023-06-23-agent/>
- MineDojo / Mineflayer (environment used by Voyager) — <https://minedojo.org/>
