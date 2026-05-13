# 89 — Voyager (Paper Deep-Dive): Open-Ended Embodied Agents with Skill Libraries

**Paper.** Guanzhi Wang, Yuqi Xie, Yunfan Jiang, Ajay Mandlekar, Chaowei Xiao, Yuke Zhu, Linxi “Jim” Fan, Anima Anandkumar — *Voyager: An Open-Ended Embodied Agent with Large Language Models* — *Transactions on Machine Learning Research* (TMLR) 2024; first released arXiv:2305.16291 (v2 19 Oct 2023) — NVIDIA, Caltech, UT Austin, and co-authors at Stanford and UW Madison. Project: <https://voyager.minedojo.org>. This note is a **paper-grounded complement** to the shorter pattern summary in [19-voyager-skill-libraries](19-voyager-skill-libraries.md).

**One-line definition.** Voyager is a **black-box GPT-4** lifelong agent in Minecraft that combines (1) a **bottom-up automatic curriculum** tuned for novelty and feasible difficulty, (2) an **iterative code-generation loop** with environment feedback, interpreter errors, and a **second GPT-4 critic for self-verification**, and (3) a **vector-indexed skill library** of verified Mineflayer JavaScript programs whose descriptions are embedded for top-k retrieval—enabling open-ended exploration without weight updates.

## Why this paper matters (the canonical reference for skill libraries, automatic curriculum, code-as-action)

Voyager is the **citable end-to-end recipe** predating productized “agent skills”: **curriculum from state → code actions → execute → verify → persist → retrieve** on the next task. It matters because baselines (ReAct, Reflexion, AutoGPT) were **re-implemented in the same MineDojo + Mineflayer** stack, so the **gaps** are design-driven, not protocol artifacts.

## Problem it solves

1. **Exploration in open-ended environments.** Minecraft has no fixed storyline; success is not a single reward. The agent must keep finding *new* worthwhile objectives instead of repeating easy behaviors.
2. **Sample efficiency at the level of *LLM calls*.** One-shot code generation fails often; the paper frames the fix as **iterative prompting** with multiple feedback channels, not more RL environment steps alone.
3. **Skill reuse without catastrophic forgetting of behavior.** Instead of overwriting weights, Voyager **accumulates executable code** in a library, indexed for retrieval, so later tasks compose earlier solutions.
4. **Fair comparison to NLP-native agents.** The authors re-implemented ReAct, Reflexion, and AutoGPT **inside** the same embodied API so failure modes reflect design, not protocol mismatch.

## Core idea in one paragraph

**Mineflayer (JavaScript)** is the action space: programs are compositional and long-horizon. **GPT-4** runs an **automatic curriculum** from state + done/fail lists; each task is solved by **iterative code synthesis** with **env feedback** (`bot.chat()`-style strings in primitives), **interpreter errors**, and a **second GPT-4** for **critic-style self-verification** (success + critique). Verified code enters a **vector DB** indexed by **text-embedding-ada-002** of a **GPT-3.5**-written description. The global goal is **novelty**: discover as many diverse things as possible (in-context **novelty search**).

## Mechanism (step by step)

**(a) Automatic curriculum prompt structure.** **GPT-4** sees: (1) directives—discover diversity; **do not** propose a task that is too hard for current **resources** and **skills**; (2) **state** (inventory, equipment, blocks/entities, biome, time, health, hunger, position); (3) **completed** and **failed** tasks; (4) **GPT-3.5** self-Q&A on state/progress (cheaper than GPT-4 for that step). It outputs **reasoning + task string** (Fig. 3). **Temp = 0.1** here only; **all other** completions **temp = 0** (Sec. 3.1).

**(b) Iterative prompting: code, execution, errors, self-verification.** Code prompt: reuse guidelines; **primitives**; **top-5** skills; prior **code**; **env feedback**; **stack traces**; **verifier critique**; **state**; **CoT** then code. After each run, feed feedback/errors back. Stop when the **verifier** GPT-4 says success, or after **4** failed rounds—then **new curriculum task** (Sec. 2.3). Verifier = **critic** (not just Reflexion): judges success and suggests fixes (e.g. mine **amethyst** for **spyglass**) (Fig. 6).

**(c) Skill library: Mineflayer JavaScript, embedding indexing, retrieval query.** **Vector DB**: value = **JS program**; key = **ada-002** embed of **GPT-3.5** program **description** (Fig. 4). **Query**: embed(**GPT-3.5** “how to do this task” **plus** env feedback) → **top-5** similarity—**RAG** over code with a cheap **query composer**.

**(d) GPT-4–specific (and API) “tricks”.** **gpt-4-0314** for curriculum + code + **verifier**; **gpt-3.5-turbo-0301** for auxiliaries; **text-embedding-ada-002**. **Diversity** mostly from **curriculum @ 0.1**; rest **@ 0**. Ablations: drop **self-verification** → **−73%** items (strongest feedback ablation). **GPT-3.5** for code → **5.7×** fewer items (Sec. 3.4).

**(e) Failure recovery.** **Iterate** on feedback; **4** bad rounds → **new task**; curriculum may **reattempt** later; bad **verifier** or **nonsense** craft recipes partially mitigated by **moving on** (not a full fix).

**Pseudocode (faithful to the paper’s control flow).**

```text
skill_db = empty vector DB
while within iteration budget:
  task, curriculum_reasoning = GPT4_curriculum(state, done_tasks, failed_tasks, self_QA, temp=0.1)
  for round in 1..4:
    plan_hint = GPT35_howto(task, state)
    query = embed(concat(plan_hint, env_feedback))
    skills = skill_db.topk(query, k=5)
    code = GPT4_code(state, task, skills, previous_code, env_feedback, exec_errors, critique, CoT, temp=0)
    env_feedback, exec_errors = Mineflayer_run(code)   # includes bot.chat() messages
    verdict, critique = GPT4_verifier(state, task)
    if verdict == success: break
  if verdict == success:
    desc = GPT35_describe(code)
    skill_db.add(embedding(desc), code)
  else:
    record failure; request new task from curriculum
  update state from environment
```

## Empirical results (table)

| Axis | Result (paper) |
|------|----------------|
| **Exploration** (160 iter.) | **63** unique items; **3.3×** vs baselines; **2.3×** map traversal (Fig. 7) |
| **Tech tree** wood→stone→iron→diamond (max 160 iter., 3 seeds) | ReAct, Reflexion: **0/3** on **any** tier. AutoGPT: wooden **92±72**, stone **94±72**, iron **135±103**; **0/3** **diamond**. **Voyager w/o lib**: no **diamond**. **Voyager**: **6±2** / **11±2** / **21±7** / **diamond 102** iter. **(1/3)**; **sole** diamond unlock. vs AutoGPT, **~15.3× / 8.5× / 6.4×** faster to wood / stone / iron. |
| **New world** (50 iter., Table 2) | Tasks: **diamond pickaxe, golden sword, lava bucket, compass**. ReAct/Reflexion/AutoGPT **0/3** each. **Voyager: 3/3**; **~19, 18, 21, 18** iter. **Voyager w/o lib** ~2× slower. **AutoGPT + Voyager library: 1/3–2/3** — library **plugs in**. |

## Variants and ablations

- **Random curriculum:** **−93%** discovered items (tasks out of order become impossibly hard).
- **Manual skill-less curriculum (human-designed):** underperforms the automatic one and needs **domain expertise** the automatic module avoids.
- **No skill library:** **plateau** in late exploration; still strong early, but fails to **compound** (Sec. 3.4, Fig. 9).
- **Ablate iterative feedback types:** **self-verification** is the **strongest** single signal (**−73%** items if removed). Environment feedback and execution errors each matter; the paper shows the full **triad** beats any subset (Fig. 9 right).
- **GPT-3.5 instead of GPT-4 for code:** **5.7×** fewer unique items.
- **AutoGPT + Voyager’s library** (Table 2): clear **zero-shot** gains vs plain AutoGPT, isolating the **library** as a plug-in asset.

## Failure modes and limitations

**Cost** — GPT-4 **~15×** GPT-3.5; paper treats frontier code as needed (Sec. 4). **Stuck / bad verifier** — can fail to learn a skill; **spider string** false negative example (Sec. 4). **Hallucinations** — curriculum: **copper sword**-type non-items; code: **cobblestone as fuel**, missing APIs. **No pixels** (text API); **Mineflayer** = **high-level** plan/code test, not VPT-style visuomotor (Sec. 3.2). **No skill consolidation** in-paper: **append-only** library → long-run **bloat** risk.

## When to use, when not to

**Use** when the environment is **scriptable** with **executable** actions, you can **surface rich textual feedback** from execution, and you have a **credible pass/fail or critic** (here: GPT-4 on state, which works best when the state is **truthful and complete**). The pattern **transfers** to **codebase agents**, **browser automation**, and **devops** that expose logs and stack traces.

**Do not** assume automatic success when: feedback is **sparse or misleading**; the API is **unstable** (stored skills **rot**); **verification** is cheaper or safer with **symbolic** checks but you still use a **fragile** LLM judge; or tasks are **one-off** with **no** reusable **primitives**—the library and curriculum **won’t** amortize.

## Implications for harness engineering

Canonical reference for **curriculum + executable skills + verification**; [19-voyager-skill-libraries](19-voyager-skill-libraries.md) = short entry, this file = **citations + numbers**. **[04-skills](04-skills.md)** (human **SKILL.md**) vs **autodiscovered** vetted code—shared **retrieval/compose** shape. **[09-memory-files](09-memory-files.md)**: machine skills vs **human-audited** files—**trust** differs. **[68-atomic-skills-scaling-coding-agents](68-atomic-skills-scaling-coding-agents.md)**: **inference-** vs **training-time** “skills.” **[69-agent-world-self-evolving-training-arena](69-agent-world-self-evolving-training-arena.md)**: **multi-env training** vs Voyager’s **one** sim **+** **black-box LLM** loop.

**Lineage:** **Claude Skills**, specialist routing, **gstack** multi-specialist setups ([75-gstack-garry-tan-claude-code-setup](75-gstack-garry-tan-claude-code-setup.md)) trace the **retrieved, composable capability** design Voyager **proved** with **Table 1/2**-style ablations.

## Connections to other work in this corpus

[13-react](13-react.md), [14-reflexion](14-reflexion.md) = Mineflayer baselines (**0/3** tech tree in **160** iter.) — need **+ curriculum + library**. [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [18-chain-of-verification-self-refine](18-chain-of-verification-self-refine.md), [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md) = **verifier** critical; **spider** case = **judge** hazard. [25-agentic-rag](25-agentic-rag.md), [79-skill-rag](79-skill-rag.md) = **RAG** over code skills. [36-autogenesis-self-evolving-agents](36-autogenesis-self-evolving-agents.md), [55-hermes-agent-self-improving](55-hermes-agent-self-improving.md), [20-metagpt](20-metagpt.md), [42-langchain-deep-agents](42-langchain-deep-agents.md) = broader agent stacks; Voyager = **three-role** **minimal** design. [40-harness-engineering-principles](40-harness-engineering-principles.md), [44-four-pillars-harness-engineering](44-four-pillars-harness-engineering.md) = **feedback + memory + progression** exemplar.

## Key takeaways

1. **Code actions + vector library** = **storable**, **re-executable** compositional behavior; **diamond** needs **full** system (**lib** ablation).
2. **Curriculum** delivers **~15.3/8.5/6.4×** tool-tier speedups; **random** curriculum **−93%** items.
3. **Critic/verifier** **−73%** if removed; **RAG** query = **GPT-3.5** plan **+** state; **code** = **GPT-4**.
4. **Table 2** = **knowledge in code** (**AutoGPT +** library); limits: **no vision**, **~15×** cost, **no consolidation**.

## References

- Wang et al., *Voyager: An Open-Ended Embodied Agent with Large Language Models*, TMLR 2024 / arXiv:2305.16291 — <https://arxiv.org/abs/2305.16291> — <https://voyager.minedojo.org>
- MineDojo; Mineflayer. Baselines: ReAct, Reflexion, AutoGPT (see [13-react](13-react.md), [14-reflexion](14-reflexion.md)).
