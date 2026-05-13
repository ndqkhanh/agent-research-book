# 161 — Paper-Researcher-AI-Agent: A Minimal CrewAI + Gemini Reference Implementation

**Repository.** https://github.com/manpreet171/Paper-Researcher-AI-Agent — author: **Manpreet** (manpreet171) — Python (~100% of file types) — Streamlit-served — built on **CrewAI** for orchestration, **Google Gemini 1.5 Flash** for LLM, **SerperDev** for web search, and **`langchain_google_genai`** for the model wrapper. Educational, single-file-per-concern project (5 Python files: `agents.py`, `tasks.py`, `crew.py`, `tools.py`, `app.py`) plus a `requirements.txt` and a `Sample run.pdf`.

**One-line definition.** Paper-Researcher-AI-Agent is the *minimum-viable* CrewAI deep-research agent: a two-agent **Researcher → Writer** pipeline that, given a topic, finds four academic papers published after 2022 via Serper-API web search, summarises them, and produces a markdown article — all in roughly 100 lines of Python with a Streamlit UI in front of it. As a piece of production software it is small; as a *reference implementation of the dynamic-multi-agent DR pattern from Huang et al.* ([158](158-deep-research-agents-survey-huang-et-al.md)), it is exactly the right size: small enough to read end-to-end in 15 minutes, complete enough to run, and structured cleanly enough to clone as the starting point for a domain-specific DR agent.

## Why a tiny educational repo deserves a deep-dive

The agent-engineering canon often skews toward the most architecturally sophisticated systems (DeerFlow 2.0 at [163](163-deer-flow-revisited.md), Feynman at [155](155-feynman-multi-agent-research-harness.md), crewAI itself at [164](164-crewai-multi-agent-framework.md)). But for builders new to the field, a tiny, complete, *readable* repo is more pedagogically valuable than a thousand-page framework. Paper-Researcher-AI-Agent is exactly that: it shows the *minimum scaffolding* required to get a working DR agent into a user's hands.

Three reasons it is worth studying:

1. **It is a faithful instantiation of the static-workflow DR pattern.** The two-agent sequential `Researcher → Writer` flow is the canonical static-workflow architecture from Huang et al. §3.3.1 — a fixed pipeline that decomposes the research process into specialised sequential subtasks. Educational systems benefit from static workflows because their structure is legible.
2. **It demonstrates CrewAI as a DR-agent runtime in <100 lines.** The implementation reads as: "here is the agent abstraction, here is the task abstraction, here is the crew abstraction, kick it off." For someone evaluating CrewAI ([164](164-crewai-multi-agent-framework.md)) as a framework choice, this is the smallest end-to-end working example.
3. **It is the stepping stone, not the destination.** Most production DR agents are built by taking a structure like this one and *extending* it — more agents, more tools, persistence, verification, MCP integration. Reading the simple version first makes the elaborated versions much easier to reason about.

## Repository structure

```
Paper-Researcher-AI-Agent/
├── README.md                        # 80 lines; setup + explanation
├── requirements.txt                 # Dependencies pinned
├── .env                             # GOOGLE_API_KEY, SERPER_API_KEY (local)
├── agents.py                        # Researcher + Writer agent definitions (~50 lines)
├── tasks.py                         # research_papers_task + write_article_task (~30 lines)
├── crew.py                          # Crew assembly + run_crew(topic) function (~15 lines)
├── tools.py                         # SerperDevTool initialisation (~10 lines)
├── app.py                           # Streamlit UI front-end
├── article-based-on-papers.md       # Writer's output destination
└── Sample run.pdf                   # Captured demo run
```

Five small Python files. One UI file. The total Python LoC is well under 200. This is the design discipline of a teaching repo.

## The agents (agents.py)

```python
from crewai import Agent
from tools import tool
from langchain_google_genai import ChatGoogleGenerativeAI

llm = ChatGoogleGenerativeAI(
    model="gemini-1.5-flash",
    verbose=True,
    temperature=0.5,
    google_api_key=os.getenv("GOOGLE_API_KEY")
)

paper_researcher = Agent(
    role="Researcher",
    goal='Find the most recent and relevant academic papers on {topic}',
    verbose=True,
    memory=True,
    backstory=("You are an expert in academic research, tasked with finding the latest and most impactful papers in your field."
               "Your goal is to identify and summarize key findings from recent research."),
    tools=[tool],
    llm=llm,
    allow_delegation=True
)

article_writer = Agent(
    role='Writer',
    goal='Write a comprehensive article based on the summaries of academic papers on {topic}',
    verbose=True,
    memory=True,
    backstory=("You are a skilled writer with a talent for distilling complex research into engaging and accessible articles."
               "Your goal is to create a narrative that highlights the key findings and implications of recent research."),
    tools=[tool],
    llm=llm,
    allow_delegation=False
)
```

Three things to note:

### 1. The `role / goal / backstory` triad

CrewAI agents are defined by three Markdown-y fields:
- **role**: the agent's professional identity ("Researcher", "Writer")
- **goal**: the parameterised objective (note `{topic}` placeholder)
- **backstory**: the natural-language persona that shapes how the LLM behaves

This is one of CrewAI's clearer design choices: agents are *characters with goals*, not function objects. The LLM responds to "you are a skilled writer with a talent for distilling complex research" differently from "you are a function that takes inputs and produces outputs". Deeper analysis at [164](164-crewai-multi-agent-framework.md).

### 2. `memory=True` and `allow_delegation`

Both agents have `memory=True` enabled. CrewAI's memory abstraction handles short-term context within a crew run; for longer-horizon memory, you would extend with a MEMTIER-style external substrate ([151-153](151-memtier-why-flat-memory-breaks-at-72-hours.md)).

The Researcher has `allow_delegation=True`, meaning it can hand off subtasks to the Writer mid-task. The Writer has `allow_delegation=False` — terminal in the chain. This is the *hierarchical-with-leaf-restriction* pattern: only intermediate agents may delegate.

### 3. Both agents share the same single tool

`SerperDevTool` (web search) is the only external capability. Both agents have access to it. This is intentional simplicity: there is no MCP layer, no code execution sandbox, no PDF parser, no citation database. The agent capabilities are bounded by the LLM's reasoning over web-search results.

## The tasks (tasks.py)

```python
research_papers_task = Task(
    description=(
        "Search for four relevant academic papers published after 2022 on the topic {topic}."
        "The papers should be the most recent and highly cited in the field."
        "Summarize the key findings and relevance of each paper."
    ),
    expected_output='A summary of four academic papers published after 2022 on {topic}.',
    tools=[tool],
    agent=paper_researcher,
)

write_article_task = Task(
    description=(
        "Compose an insightful article based on the summaries of the four academic papers on {topic}."
        "Focus on synthesizing the information into a coherent and engaging narrative."
        "The article should highlight the key findings and their implications."
    ),
    expected_output='A comprehensive article on {topic} advancements formatted as markdown.',
    tools=[tool],
    agent=article_writer,
    async_execution=False,
    output_file='article-based-on-papers.md'
)
```

Two tasks, ordered:

1. **research_papers_task**: assigned to `paper_researcher`. Description hard-codes "four papers" and "after 2022". Output: a summary string.
2. **write_article_task**: assigned to `article_writer`. Reads the previous task's output. Output: written to `article-based-on-papers.md`.

The `output_file` parameter is the file-based-handoff primitive ([155-feynman-multi-agent-research-harness](155-feynman-multi-agent-research-harness.md) makes this normative). The article is durable on disk; the user can review or edit before publishing.

`async_execution=False` is the default; tasks run sequentially. CrewAI also supports parallel async execution for tasks that don't depend on each other, but a research-then-write pipeline must be sequential.

## The crew (crew.py)

```python
from crewai import Crew, Process
from tasks import research_papers_task, write_article_task
from agents import paper_researcher, article_writer

crew = Crew(
    agents=[paper_researcher, article_writer],
    tasks=[research_papers_task, write_article_task],
    process=Process.sequential,
)

def run_crew(topic):
    result = crew.kickoff(inputs={'topic': topic})
    return result
```

The crew is the orchestrator. Three components:

- **agents list**: who is in the crew.
- **tasks list**: what the crew does, in order.
- **process**: `Process.sequential` (default) — task 2 starts after task 1 finishes. CrewAI also supports `Process.hierarchical` for boss-and-workers patterns.

`crew.kickoff(inputs={'topic': topic})` is the entry point. The `{topic}` placeholders in agents and tasks get filled here.

## The Streamlit UI (app.py — inferred)

Not shown above, but the pattern is standard:
- Text input for the topic.
- Button to trigger `run_crew(topic)`.
- Markdown rendering of the resulting article.
- Optional: progress indicators and verbose log streaming.

The UI is incidental. The interesting code is in agents/tasks/crew.

## What this repo demonstrates

Five concrete patterns that generalise far beyond this repo:

### 1. CrewAI's lean abstraction layer

CrewAI strips agent-system primitives down to four objects: **Agent, Task, Tool, Crew**. The orchestration semantics are declarative; you specify the agents, the tasks, the order, and the framework handles the rest. Compare with LangChain's Deep Agents ([42](42-langchain-deep-agents.md)) where you explicitly construct planner / executor / subagents and wire them together — CrewAI's `Crew(...)` is a one-liner. This minimalism matters for educational repos and for rapid prototyping.

### 2. Static workflow as the right starting point

The repo is a static workflow in Huang et al.'s taxonomy ([158](158-deep-research-agents-survey-huang-et-al.md) §3.3.1). Two agents, fixed sequence, no replanning. This is *not* what production deep research looks like — but it's *exactly* what an educational MVP should look like. The discipline is: get a static workflow working end-to-end before introducing dynamic re-planning. Most production DR agents started as static workflows that were then opened up to dynamic re-planning at specific decision points.

### 3. Markdown is the output format

`output_file='article-based-on-papers.md'` is a small detail with a big implication: the deliverable is a Markdown file, on disk, that humans can read and version-control. This is the same discipline that Feynman ([155](155-feynman-multi-agent-research-harness.md)), HeavySkill skill files ([156](156-heavyskill-parallel-reasoning-deliberation.md)), and Deep Researcher Agent ([160](160-deep-researcher-agent-24x7.md)) follow. *Files outlive sessions.*

### 4. Roles and goals as character design

Defining agents by `role / goal / backstory` puts the work into the natural-language description of the agent's identity, not into hand-tuned prompts at the call site. Once you have a Researcher agent with a clear backstory, you can ask it to research many topics — the prompt engineering is centralised.

This generalises: agent design is character design. See [20-metagpt-role-based-workflows](20-metagpt-role-based-workflows.md) for the role-as-design-primitive pattern at scale.

### 5. Tools as shared, discoverable capability

Both agents share the SerperDevTool. New tools can be added by writing a Python class with a `_run` method (the `crewai_tools` library has dozens already; custom tools are a few lines).

The MCP-compatible version of this pattern would expose tools as an MCP server, letting any agent invoke them via JSON-RPC — that's [07-model-context-protocol](07-model-context-protocol.md) and the direction the field is moving.

## Limitations as a production system

Honest enumeration of what this repo doesn't do — useful as a checklist of what to add when extending it:

1. **No source verification.** The Researcher's "papers" come from a Serper web search. The Writer is told to write based on these summaries. Nothing checks that the cited papers exist, that the URLs resolve, or that the claims attached to citations are supported. Compare with Feynman's mandatory Verifier subagent ([155](155-feynman-multi-agent-research-harness.md)).
2. **No persistent memory.** `memory=True` is short-term within a crew run. Subsequent runs on the same topic start fresh. There is no MEMTIER-style episodic / semantic substrate ([151-153](151-memtier-why-flat-memory-breaks-at-72-hours.md)).
3. **No structured plan.** The two tasks are hardcoded. The agent does not produce a research plan that the user can approve. Compare with Feynman's `outputs/.plans/<slug>.md` discipline.
4. **No iteration or refinement.** The Researcher runs once; the Writer runs once. There is no feedback loop where the Writer requests more papers, or the Reviewer flags weaknesses, or a Verifier removes unsourced claims.
5. **No async execution.** Sequential only. For high-throughput research workloads, this leaves performance on the table.
6. **No multi-LLM orchestration.** Both agents use Gemini 1.5 Flash. A production system might use a strong reasoner for the researcher and a fast instruction-follower for the writer (per HeavySkill's deliberator-strength insight at [156](156-heavyskill-parallel-reasoning-deliberation.md)).
7. **No cost controls.** No prompt caching, no token budgets, no anti-burn protection. Fine for educational use, problematic for 24/7 deployment.
8. **No domain skills.** All Researcher knowledge comes from in-prompt instructions and web search. There is no skill library encoding "how to evaluate a paper's methodology" or "how to write a literature review" (compare with Feynman's 19 research skills at [155](155-feynman-multi-agent-research-harness.md), and Ctx2Skill's adversarial skill construction at [154](154-ctx2skill-self-evolving-context-skills.md)).

These are not flaws of the repo — they are *deliberate omissions for educational clarity*. If you cloned this repo and added all of (1)–(8), you would have something resembling Feynman.

## How to extend it

A natural progression for someone who has cloned this repo:

### Stage 1 — Add a Verifier agent

Create a third agent that reads the Writer's output and adds inline citations + verifies URLs. Use Feynman's Verifier prompt ([155](155-feynman-multi-agent-research-harness.md)) as a reference. Run it as a third sequential task.

### Stage 2 — Add structured plan generation

Add a *Planner* agent that runs first, produces an outline plan in `outputs/.plans/<slug>.md`, and waits for user confirmation before the rest of the crew proceeds. This converts the workflow from static to dynamic-with-intent-to-planning per Huang et al. §3.3.2.

### Stage 3 — Replace memory with MEMTIER

Swap `memory=True` for an episodic JSONL store + LLM-distilled semantic tier ([151-153](151-memtier-why-flat-memory-breaks-at-72-hours.md)). Now the agent retains research from prior runs.

### Stage 4 — Add MCP tool servers

Replace `SerperDevTool` with MCP-compatible tool servers (Tavily MCP, Exa MCP, alphaXiv MCP, etc.). Now the agent's tool set is portable across runtimes ([07-model-context-protocol](07-model-context-protocol.md)).

### Stage 5 — Internalise reasoning as a skill

If the Researcher repeatedly performs heavy parallel reasoning (HeavySkill pattern at [156](156-heavyskill-parallel-reasoning-deliberation.md)), distil the workflow into a `HeavySkill.md` skill file and load it into the agent's context. Eventually, this can become RLVR training data.

By Stage 5, you have a system that is architecturally comparable to production-grade DR agents — built from a 100-line educational repo as the seed.

## How this repo fits the broader picture

In the August 2025 / June 2025 surveys ([158](158-deep-research-agents-survey-huang-et-al.md), [159](159-deep-research-survey-zhang-et-al.md)) Paper-Researcher-AI-Agent would be classified as:

- **Static workflow** (Huang §3.3.1)
- **Multi-agent** (Huang §3.3.3) — two specialised agents
- **Planning-only** strategy (Huang §3.3.2) — no user clarification before research
- **API-based search** (Huang §3.1) — Serper API, not browser
- **Prompt-driven, no fine-tuning** (Huang §3.4)
- **Pipeline stages**: Question Developing handled implicitly by Researcher; Web Exploration via Serper API; Report Generation by Writer (Zhang four-stage framing at [159](159-deep-research-survey-zhang-et-al.md))

This classification is exact: the repo demonstrates the *baseline* of each axis. Extensions to the repo (Stages 1–5 above) move it along each axis toward the more sophisticated end.

## When to use this repo

Use this repo as your starting point when:

- You are learning CrewAI and want a complete working example to read.
- You are building a *domain-specific* educational tutorial DR agent (e.g., "research papers on dermatology", "research papers on a specific software library").
- You want a Streamlit-served MVP for a small team, not a production system.
- You will iterate by adding capabilities, not by replacing the whole thing.

Don't use this repo as the basis when:

- You need verification, provenance, or citation accuracy as default behaviours (use Feynman at [155](155-feynman-multi-agent-research-harness.md)).
- You need parallel async execution at scale (use DeerFlow 2.0 at [163](163-deer-flow-revisited.md)).
- You need an autonomous experimentation loop (use Deep Researcher Agent at [160](160-deep-researcher-agent-24x7.md)).
- You need persistent memory across sessions (use MEMTIER architecture at [151-153](151-memtier-why-flat-memory-breaks-at-72-hours.md)).
- You need source-grounded research with mandatory verification (use Feynman at [155](155-feynman-multi-agent-research-harness.md)).

## Where this fits in the canon

- **Read alongside**: [164-crewai-multi-agent-framework](164-crewai-multi-agent-framework.md) — the framework this repo uses.
- **Compare with**: [155-feynman-multi-agent-research-harness](155-feynman-multi-agent-research-harness.md) — the production-grade source-grounded equivalent.
- **Pattern references**: [02-subagent-delegation](02-subagent-delegation.md), [20-metagpt-role-based-workflows](20-metagpt-role-based-workflows.md), [42-langchain-deep-agents](42-langchain-deep-agents.md), [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md).
- **Survey context**: [158-deep-research-agents-survey-huang-et-al](158-deep-research-agents-survey-huang-et-al.md), [159-deep-research-survey-zhang-et-al](159-deep-research-survey-zhang-et-al.md).
- **Adjacent simple-multi-agent repos**: see [161](161-paper-researcher-ai-agent.md) (this), [165-ralph-autonomous-loop](165-ralph-autonomous-loop.md) (autonomous coding loop).
- **Synthesis**: [166-research-agent-frameworks-synthesis](166-research-agent-frameworks-synthesis.md).

## References

1. Repository: https://github.com/manpreet171/Paper-Researcher-AI-Agent.
2. CrewAI framework: https://github.com/crewAIInc/crewAI — main framework deep-dive at [164-crewai-multi-agent-framework](164-crewai-multi-agent-framework.md).
3. SerperDev: https://serper.dev/ — Google Search API used by the SerperDevTool.
4. Google Gemini API: https://ai.google.dev/gemini-api — the LLM backbone.
5. langchain_google_genai: integration package for Gemini in LangChain ecosystem.
6. Streamlit: https://streamlit.io/ — UI framework.
7. Adjacent canon: [02-subagent-delegation.md](02-subagent-delegation.md), [04-skills.md](04-skills.md), [20-metagpt-role-based-workflows.md](20-metagpt-role-based-workflows.md), [25-agentic-rag.md](25-agentic-rag.md), [42-langchain-deep-agents.md](42-langchain-deep-agents.md), [155-feynman-multi-agent-research-harness.md](155-feynman-multi-agent-research-harness.md).
