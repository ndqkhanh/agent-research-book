# 91 — MetaGPT (Paper Deep-Dive): SOPs for Multi-Agent Collaboration

**Paper.** Sirui Hong, Mingchen Zhuge, Jiaqi Chen, Xiawu Zheng, Yuheng Cheng, Ceyao Zhang, Jinlin Wang, Zili Wang, Steven Ka Shing Yau, Zijuan Lin, Liyang Zhou, Chenyu Ran, Lingfeng Xiao, Chenglin Wu, Jürgen Schmidhuber — *MetaGPT: Meta Programming for A Multi-Agent Collaborative Framework* — **ICLR 2024** (published as a conference paper) — arXiv:2308.00352 (v7 dated 1 Nov 2024). This note is a **paper-grounded complement** to the workflow-oriented summary in [20-metagpt-role-based-workflows](20-metagpt-role-based-workflows.md), which orients the reader to roles and artifacts; here we keep the same conceptual spine but align claims, numbers, and mechanisms to the ICLR text.

**One-line definition.** Encoding **software development SOPs** as multi-agent **communication protocols** whose hand-offs are **structured outputs** (PRD, system design, tasks, code, tests), plus a **shared message pool** and **executable feedback**, so intermediate artifacts stay verifiable and cascaded hallucination from naïve LLM chains is reduced.

## Why this paper matters

Before structured workflows, “societies of LLM agents” for engineering often resembled **unconstrained multi-turn chat**: information distorts like a telephone game, roles drift, and **cascading hallucinations** compound when models are naïvely chained. MetaGPT is an early, influential demonstration that **human SDLC SOPs** are not just metaphor but an implementable inductive bias: fixed roles, document-shaped interfaces, and pub/sub over a **global store** can outperform chat-centric multi-agent scaffolds on **both** small coding benchmarks and a **self-built SoftwareDev** suite. The work also isolates a reusable primitive—**executable feedback** (run tests, read stderr)—whose ablation is quantitatively large on Pass@1. For harness engineering, MetaGPT is the reference implementation of “**assembly-line agents** with typed artifacts between stations.”

## Problem it solves (cascading hallucinations in multi-agent SDLC, unstructured comms)

The paper’s diagnosis is twofold. First, **logic inconsistencies** and **cascading hallucinations** appear when multiple LLMs interact without **checkpointed, schema-like artifacts**—each hop amplifies error. Second, **unstructured natural language** as the sole inter-agent bus makes **telephone-game** drift inevitable over many hops; the authors contrast this with **dialogue-style** multi-agent code systems that trade messages without stable structure. They also flag operational pathologies in prior work—**repeated instruction**, **infinite message loops**—as motivation for a regulated workflow rather than open-ended role-play. MetaGPT’s bet is: **SOPs + structured I/O** turn collaboration into a sequence of **verifiable** stages instead of an unstructured debate.

## Core idea in one paragraph

Model a software company: agents are **ReAct-style** (Yao et al.) but each has a **profile** (name, role, goal, constraints) and **role-specific skills** (e.g., Product Manager with optional web search, Engineer with code execution). Work proceeds **down an SDLC SOP**—roughly **PRD → architecture / API & flow artifacts → project/task decomposition → implementation → QA**—so each stage consumes **typed documents** and emits the next, rather than ad hoc chat. All structured messages go to a **shared message pool**; agents **subscribe** to the prerequisites their role needs, reducing pairwise chatter. After code exists, the **Engineer** loop closes the loop with **executable feedback**: run code and tests, retry up to **3** times, using memory and requirements/design context for debug—raising **runnable** correctness over pure LLM “review” or reflection on non-executing text.

## Mechanism (step by step)

### (a) SOP encoded as roles

**Five roles** (Fig. 1/3, Sec. 3.1): **PM, Architect, Project Manager, Engineer, QA**—each with **profile/constraints** and **ReAct** (observe pool → act). **Order:** **PRD** → **system design** (files, APIs, flow) → **tasks** → **code** → **tests**. Appendix B: full GUI example (PRD with **requirement pool** P0/P1; Architect file list + diagrams; Project Manager **task list** + shared knowledge).

**Pseudocode — coarse role transition (dependencies gate activation):**

```text
SOP: [PM, AR, PJ, EN, QA]
pool ← ∅
for role R in SOP (in order):
  wait until all prerequisite artifacts for R are in pool  # subscription satisfied
  out ← R.run(context = subscribed(pool), tools = skills(R))
  pool.publish(structured(out))  # PRD, design, tasks, code, tests per schema
# Engineer inner loop may iterate with executable feedback before QA finalizes
```

### (b) Structured output formats (PRD, design docs, code, tests)

**Structured communication interfaces** (Sec. 3.2): each role must emit per-schema content—e.g. PM: **User Stories**, **Requirement Pool**; Architect: **file lists**, **data structures**, **API / interface** definitions, **sequence** views; Project Manager: **task list**, per-file responsibility mapping; Engineer: code files aligned to the design; QA: **unit tests** and review. The paper contrasts MetaGPT with **ChatDev**-style *dialogue*: here, **“documents and diagrams”** are the message body so irrelevant or missing fields are less likely to hide in small talk. Appendix B’s PRD and system-design blocks are concrete instantiations of these schemas (including “Anything UNCLEAR” sections to surface ambiguity early).

### (c) Publish–subscribe message routing

**Shared message pool** (Fig. 2 left): every agent **publishes** structured messages into a **global** pool; any agent can **read** them without waiting on synchronous pairwise replies—avoiding a dense **N×N** dialog topology. **Subscription** filters overload: the Architect *primarily* follows the PM’s PRD; a role **activates** when its **prerequisites** are present (Sec. 3.2). This implements **directional** information flow while keeping a **single** source of truth for shared artifacts (PRD remains addressable to all downstream roles without re-asking the PM each time).

### (d) Executable feedback (compile, test)

**Executable feedback** (Sec. 3.3; Fig. 2 right): after initial code, the **Engineer** does not rely solely on an LLM critic. They **write and run unit tests**; on failure, they **debug** using **execution memory** and cross-checks against **PRD**, **system design**, and code—iterating until tests pass or **max 3 retries**. This directly targets “**non-executable** code review” limits of prior work; the paper notes early MetaGPT without execution missed errors that “review-only” passed (hallucinated OK).

### (e)–(f) Pool as state; iterative debug

The **pool** is the ReAct **environment** (Sec. 3.1): observe **messages** → act → **publish** new structured messages (**logs/replay**). The Engineer’s loop ties **stdout/stderr + test output** to memory and specs, **patch → re-run**, cap **3**; **QA** in Appendix B adds end-stage tests—main text stresses **Engineer**-driven **executable** iteration.

## Empirical results (specific table — head-to-head where defined)

**HumanEval / MBPP (Pass@1, %).** **85.9%** (**HumanEval**) and **87.7%** (**MBPP**) (Fig. 4, Introduction/Sec. 4.2). Appendix **Table 7** shows **GPT-4** alone is **sensitive to prompt/parser**: settings A–C span **~0.72–0.81** on HumanEval—**scaffold > raw model.**

**SoftwareDev (subset of 7 tasks in main comparison; human-rated executability + stats).** **Table 1** is the key multi-agent / framework comparison **among MetaGPT configurations and ChatDev**:

| Statistical Index | ChatDev | MetaGPT w/o Feedback | MetaGPT (full) |
|---:|---:|---:|---:|
| (A) Executability (1–4) | **2.25** | **3.67** | **3.75** |
| (B) Running time (s) | **762** | **503** | **541** |
| (B) Token usage | **19,292** | **24,613** | **31,255** |
| (C) Total code lines | **77.5** | **194.6** | **251.4** |
| (D) Productivity (tokens / line) | **248.9** | **126.5** | **124.3** |
| (E) Human revision cost (count) | **2.5** | **2.25** | **0.83** |

**Interpretation for “vs ChatDev”:** MetaGPT is **higher** on **executability** and **lower** on **human revisions**; it uses **more tokens** and produces **more code** (more files/lines) but **better productivity** in tokens-per-line. **W/o feedback** is faster and cheaper in tokens but weaker on final quality and revisions than the full system.

**SoftwareDev: broader baselines (Table 4, App. C.2, 11 tasks).** **AutoGPT, LangChain, AgentVerse** average executability **1.0**; **ChatDev 2.1**; **MetaGPT 3.9**. The ICLR text names **AutoGPT** / **AgentVerse**-style systems; it does **not** report **AutoGen** (Microsoft) in these tables—use **Table 1** for **MetaGPT vs ChatDev** on cost/quality, and **Table 4** for “general agents vs SOP MetaGPT” on the same mini-app suite.

**Executable feedback ablation (Pass@1, %).** Adding executable feedback yields **+4.2%** on **HumanEval** and **+5.4%** on **MBPP** (Sec. 4.4); Table 1 shows **3.67 → 3.75** executability and **2.25 → 0.83** human revisions when feedback is enabled.

## Variants and ablations

- **Role ablation (Table 3, Engineer+others):** moving from **Engineer-only** (executability **1.0**, **$0.915**, **10** human revisions) up through adding **Product Manager**, then **Architect**, **Project Manager**, monotonically improves **revisions** and **executability** (best row: **4** roles, **executability 4.0**, **2.5** revisions) at higher **$** expense.  
- **MetaGPT w/o executable feedback** vs **full** (Table 1, Fig. 4) isolates the runtime test/exec loop.  
- **LLM backend (Table 5, App.):** **GPT-4** **3.8** executability vs **GPT-3.5** **2.8** vs **Deepseek Coder 33B** **1.4**—**SOPs do not erase** the backbone.  
- **Prompt level (Table 6):** **detailed** user prompts can reach **4.0** executability with **more lines**; **high-level** still reaches **3.8** on average in their 5-task mini-study.

## Failure modes and limitations (rigid SOPs, overhead for simple tasks, prompt engineering brittleness)

- **SOP rigidity:** The **linear** company workflow is a feature for coherence and a **bug** for tasks that need **exploration**, backtracking, or cross-cutting edits (Sec. 3.1/Outlook).  
- **Overhead for small jobs:** **Five roles** and long artifact chains are **wasteful** for one-shot functions; Table 1 shows **~31k tokens** for real apps—often >> a single `HumanEval` completion.  
- **Prompt / parser sensitivity:** Table 7 shows **GPT-4**’s **HumanEval** score swings **~0.72–0.81** under settings A–C—**scaffolding** around the same backbone matters a lot.  
- **Diversity / convergence:** A fixed SOP can **collapse** agent diversity—every run walks the same stations; if **every** role shares a **homogeneous** inductive prior, the system risks **repeated failure modes** (resonant with [98-diversity-collapse-mas](98-diversity-collapse-mas.md)). The paper’s pub/sub and structured artifacts **mitigate** chat drift but **do not** guarantee ideation diversity.  
- **App limits (Appendix D):** no dedicated **UI/multimodal** stack at publication; **user checkpointing** (pause/resume per agent) is listed as a human-side pain point.

## When to use, when not

**Use** when: you need **end-to-end** small/medium app generation with **traceable** intermediate artifacts; you can pay **multi-role token and wall-clock** cost; you have **runnable** tests or can synthesize them; you benefit from **document-shaped** interfaces for hand-off between “teams.” **Avoid** for: **single-function** coding benchmarks where one strong codegen call suffices; **researchy** open-ended design where a **single** agent with tools beats a **pipeline**; or when you cannot **execute** code safely to close the feedback loop.

## Implications for harness engineering

[20-metagpt-role-based-workflows](20-metagpt-role-based-workflows.md) — pattern; this file — **ICLR-anchored** mechanism and metrics. **MetaGPT** is the **canonical SOP-driven** multi-agent SDLC framework. [02-subagent-delegation](02-subagent-delegation.md) — compressing many roles into a **few** subagents. [92-chatdev](92-chatdev.md) — **chat**-centric peer; **Table 1** is the in-paper **SoftwareDev** head-to-head. [98-diversity-collapse-mas](98-diversity-collapse-mas.md) — homogeneity under shared protocols. **Modern agent CLIs** (e.g. Cursor agents, “**gstack** 23 roles”-style rosters) inherit the same **role-based** + **phased** pattern—**system prompts** as job definitions and **structured** hand-offs—even when the codebase is not MetaGPT.

## Connections to other work in this corpus

[20-metagpt-role-based-workflows](20-metagpt-role-based-workflows.md); [02-subagent-delegation](02-subagent-delegation.md); [92-chatdev](92-chatdev.md); [98-diversity-collapse-mas](98-diversity-collapse-mas.md); [77-meta-tts-agentic-coding](77-meta-tts-agentic-coding.md) (test-time RTV/PDR on rollouts *vs* MetaGPT SOP/artifacts).

## Key takeaways

1. **SOPs as protocols** — **prompt + schema + order**, not open chat.  
2. **Artifacts in a pool** — **PRD / design / tasks / code / tests** are the **message types**.  
3. **Pub/sub** — **prerequisite** gating, fewer **N×N** dialog edges.  
4. **Executable feedback** — **+4.2%** HumanEval / **+5.4%** MBPP Pass@1; Table 1 **0.83** vs **2.5** human revisions (vs ChatDev).  
5. **Scores** — **85.9% / 87.7%** HumanEval/MBPP; **SoftwareDev** **3.75** vs ChatDev **2.25** executability, **31,255** tokens, **124.3** tok/line vs ChatDev **248.9**.  
6. **Limits** — linear **SOP** rigidity, **LLM** ceiling (Table 5/7), **diversity** risk under fixed roles.

## References

- Hong, Zhuge, Chen, et al., *MetaGPT: Meta Programming for A Multi-Agent Collaborative Framework*, ICLR 2024, arXiv:2308.00352 — <https://arxiv.org/abs/2308.00352>  
- MetaGPT (code) — <https://github.com/geekan/MetaGPT>  
- Yao et al., *ReAct: Synergizing Reasoning and Acting in Language Models* — (ReAct behavior for agents)  
- Qian et al., *Communicative Agents for Software Development* (ChatDev) — arXiv:2307.07924  
- Software engineering / SOPs context as cited: Belbin team roles; Agile Manifesto; DeMarco & Lister, *Peopleware* (see paper’s introduction)
