# 226 — GPT-Researcher in May 2026: The OSS Deep-Research Reference Platform

> **Disambiguation.** This file is a **May-2026 paper-grounded deep-dive** on [github.com/assafelovic/gpt-researcher](https://github.com/assafelovic/gpt-researcher), the most-starred OSS deep-research agent. Read alongside [166-research-agent-frameworks-synthesis](166-research-agent-frameworks-synthesis.md), [158-deep-research-agents-survey-huang-et-al](158-deep-research-agents-survey-huang-et-al.md), [159-deep-research-survey-zhang-et-al](159-deep-research-survey-zhang-et-al.md), [160-deep-researcher-agent-24x7](160-deep-researcher-agent-24x7.md), [165-ralph-autonomous-loop](165-ralph-autonomous-loop.md), and [172-polaris-2026-deep-research-roadmap](172-polaris-2026-deep-research-roadmap.md). The technique substrate sits in [199-multi-hop-reasoning-techniques-arc](199-multi-hop-reasoning-techniques-arc.md), [200-graph-grounded-multi-hop-retrieval](200-graph-grounded-multi-hop-retrieval.md), and [206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md). The cross-project apply mapping is [207-cross-project-collaborative-multi-hop-apply-plan](207-cross-project-collaborative-multi-hop-apply-plan.md).

## One-line definition

GPT-Researcher is the **OSS deep-research reference platform** (~26.8k★, MIT, v3.4.4 April 2026) whose 2024–2026 evolution from "single-agent web summariser" to "Chief-Editor-orchestrated multi-agent + recursive Deep-Research mode + bidirectional MCP host-and-server" makes it the canonical pattern bin for any team building a deep-research agent — eight discrete patterns (planner-runner-curator-publisher, LangGraph editorial team, recursive tree exploration, hybrid web+local, pluggable retrievers, MCP both directions, NextJS streaming UI, DeepResearchGym-style eval) compose into a 26-source autonomous research pipeline that **CMU's DeepResearchGym ranked top of class on citation quality and coverage** vs. Perplexity / OpenAI / OpenDeepSearch / HuggingFace.

## Why this paper-bin matters

Deep research is the agentic application that **pays the highest dollar value per query** (each report saves an analyst hours), the **highest reputation cost when wrong** (a hallucinated citation in a published report is unrecoverable), and the **most demanding multi-hop substrate** (cross-source synthesis, citation faithfulness, coverage breadth, contradictory-evidence handling all required simultaneously). GPT-Researcher is the OSS reference because it ships every pattern an in-tree project (Polaris, Atlas-Research, Lyra, Helix-Bio) needs to consider, and ships them under MIT with a clean separation between the framework, the MCP server (separate `gptr-mcp` repo), and the front-end.

Three things make GPT-Researcher distinctive in May 2026: (1) **Chief-Editor multi-agent orchestration** built on LangGraph and AG2 — the editorial-team pattern (Researcher → Reviewer → Revisor → Writer → Publisher) maps onto a real workflow rather than a contrived role catalog. (2) **Bidirectional MCP** — GPT-Researcher acts as both an MCP *host* (consuming external MCP retrievers) *and* an MCP *server* (exposing `deep_research`, `quick_search`, `write_report` tools to external clients like Claude Desktop). (3) **Recursive Deep Research mode** (v3.2.2+) — tree-like exploration with configurable breadth + depth knobs and budget-aware termination, conceptually similar to dzhng's design but with GPT-Researcher's curation/citation pipeline.

Take this seriously and three things change. (1) Any in-tree deep-research project (Polaris, Atlas, Helix) should benchmark itself against GPT-Researcher on DeepResearchGym before claiming improvements. (2) The Chief-Editor LangGraph pattern is the lowest-cost multi-agent shape that survives the [202](202-multi-agent-multi-hop-reckoning-2026.md) equal-budget critique because the editorial team has *information-asymmetric roles* (Researcher sees web; Writer sees only Researcher's notes), not redundant-prompt cosplay. (3) Bidirectional MCP becomes the default contract for any in-tree project that wants to be a *citizen* of the broader agent ecosystem rather than a closed island.

## Problem GPT-Researcher solves

- Single-shot web summarization fails on multi-source synthesis.
- Closed deep-research products (OpenAI, Perplexity, Gemini) trap research output in proprietary silos.
- Hand-built deep-research stacks reinvent retriever interfaces, citation systems, and publisher pipelines.
- Hybrid (web + local) research is poorly supported in most deep-research tools.
- Citation drift and fabrication remain at 20–30% in raw GPT-class generation; retrieval grounding alone is insufficient.
- Token cost on naive recursive expansion explodes; budget control is required.

## §1 — Project state (May 2026)

| Property | Value |
|---|---|
| Canonical repo | [github.com/assafelovic/gpt-researcher](https://github.com/assafelovic/gpt-researcher) |
| Stars | ~26.8k |
| License | MIT |
| Latest version | v3.4.4 (16 April 2026) |
| Releases | 69+ tagged |
| Language mix | Python 55.2%, TypeScript 27.6%, JavaScript 7.0% |
| Open issues / PRs | ~173 / ~43 |
| MCP server (separate repo) | [github.com/assafelovic/gptr-mcp](https://github.com/assafelovic/gptr-mcp) |
| Primary maintainer | Assaf Elovic (also CTO/founder context at Tavily, the default search backend) |
| Documentation | [docs.gptr.dev](https://docs.gptr.dev) |
| Hosted offering | [gptr.dev](https://gptr.dev/) |
| Discord | active developer community |

**Notable commit cadence.** Multiple merges per week; dependabot churn plus feature PRs across the v3.x line. Active community contributors visible in releases.

## §2 — Architecture: planner-runner-curator-publisher

Single-agent mode is the load-bearing pattern:

```
┌────────────────────────────────────────────────────────┐
│           Query (user prompt + config)                 │
└────────────────┬───────────────────────────────────────┘
                 ▼
        ┌────────────────┐
        │   Planner LLM  │   expand → research sub-questions
        └────────┬───────┘
                 ▼
        ┌────────────────┐
        │   Researchers  │   per-sub-question
        │ (parallel fan- │   ─── retriever (Tavily / SerpAPI /
        │     out)       │       Bing / Searx / MCP / local)
        └────────┬───────┘
                 ▼
        ┌────────────────┐
        │ Context Curator│   dedupe + rank + chunk
        └────────┬───────┘
                 ▼
        ┌────────────────┐
        │     Writer     │   synthesise + cite
        └────────┬───────┘
                 ▼
        ┌────────────────┐
        │    Publisher   │   PDF / DOCX / Markdown / HTML
        └────────────────┘
```

**Key properties.**

- **Pluggable retrievers.** Tavily is default; SerpAPI, Bing, Searx, MCP retrievers, local-doc retriever, custom retrievers are all callable behind the same interface. This is the single most-copied pattern in the ecosystem.
- **Information asymmetry between roles.** Researcher sees raw web; Writer sees only Researcher's curated notes. Reduces hallucination by structuring what each LM call has access to.
- **Multi-format publisher.** PDF/DOCX/Markdown/HTML emitted by a terminal node; templates preserve citations.

## §3 — Multi-agent (LangGraph + AG2): the Chief-Editor pattern

The multi-agent mode layers an editorial team on top of the single-agent pipeline:

```
                ┌─────────────────────┐
                │   Chief Editor      │  orchestrator
                └──────────┬──────────┘
                           ▼
       ┌──────────┬────────┴────────┬──────────┬──────────┐
       ▼          ▼                 ▼          ▼          ▼
┌──────────┐┌──────────┐    ┌──────────┐┌──────────┐┌──────────┐
│Researcher││Reviewer  │    │Revisor   ││Writer    ││Publisher │
│(per-     ││(reads    │ ←─ │(applies  ││(synthe-  ││(emits)   │
│ subtopic)││draft +   │    │ feedback)││ sises)   ││          │
│          ││source)   │    │          ││          ││          │
└──────────┘└──────────┘    └──────────┘└──────────┘└──────────┘
```

Implemented on **LangGraph state graphs** (with optional AG2 / AutoGen v0.3 path). Each subtopic gets its own Researcher branch; drafts cycle through Reviewer/Revisor before the Writer assembles.

**Why this survives the [202](202-multi-agent-multi-hop-reckoning-2026.md) equal-budget critique.** Most multi-agent setups Tran & Kiela criticise are *redundantly compositional* (two agents doing the same task with different prompts). The Chief-Editor pattern has **information-asymmetric roles** — Researcher sees web; Reviewer sees draft + source; Revisor sees draft + Reviewer feedback; Writer sees only the Revisor's approved notes. The asymmetry is the structural source of the win, not parallel sampling.

## §4 — Deep Research (recursive tree mode, v3.2.2+)

Recursive workflow employing **tree-like exploration** with configurable depth and breadth:

- Each node spawns a sub-research run with sub-questions.
- Breadth = how many sub-questions per node.
- Depth = how many levels deep the tree goes.
- Budget-aware termination: stops when token / time / cost cap is reached.
- ~5 minutes per cycle, ~$0.40/query using o3-mini reasoning model (per release notes).

Conceptually similar to [dzhng/deep-research](https://github.com/dzhng/deep-research) but composes with GPT-Researcher's curation + citation pipeline.

## §5 — MCP integration: bidirectional

GPT-Researcher is the canonical OSS example of **MCP both directions**:

### Direction 1 — GPT-Researcher as MCP host

Configurable retrievers include MCP servers. Documented at [docs.gptr.dev/docs/gpt-researcher/retrievers/mcp-configs](https://docs.gptr.dev/docs/gpt-researcher/retrievers/mcp-configs). External MCP servers (GitHub, databases, custom APIs) appear as first-class retrievers behind the same pluggable interface as Tavily / SerpAPI / Bing.

### Direction 2 — GPT-Researcher as MCP server

Separate repo [gptr-mcp](https://github.com/assafelovic/gptr-mcp). Exposes three tools:

- `deep_research` — full GPT-Researcher pipeline as a callable.
- `quick_search` — single-shot search + summarise.
- `write_report` — synthesise existing research notes into a report.

Defaults: port 8000, supports stdio + SSE transports (`MCP_TRANSPORT` env), Docker deployment, documented integrations for Claude Desktop and Claude API.

**Why bidirectional matters.** A research agent that's *only a host* can't be embedded in another agent. A research agent that's *only a server* can't compose with rich retrievers. Bidirectional makes GPT-Researcher a citizen of the broader agent ecosystem.

## §6 — Recent feature drops (2024–2026)

- **Detailed Reports.** Long-form, multi-section reports replacing 5-page default; >2,000 word output common.
- **Multi-Agent (LangGraph + AG2).** Chief Editor pattern; production-ready.
- **Hybrid Research.** Web + local-doc in one run, fixing isolation bias of most RAG setups.
- **Local Documents.** PDF / DOCX / CSV / Excel / Markdown / PowerPoint / Word ingestion.
- **Pluggable retrievers / plugins.** BYO retriever; same contract for all.
- **NextJS + Tailwind frontend.** Replaced legacy Flask UI; WebSocket streaming for token-by-token research progress.
- **GPTR-MCP** (2025). Separated MCP server into its own repo for independent versioning.
- **Deep Research mode** (v3.2.2). Recursive tree exploration with breadth + depth knobs.
- **Image generation** (recent). Gemini Nano Banana for inline AI-generated illustrations.
- **DeepResearchGym top-of-class.** CMU's May 2025 evaluation ranked GPT-Researcher highest on citation quality, report quality, and coverage vs. Perplexity / OpenAI / OpenDeepSearch / HuggingFace.

## §7 — Comparison: GPT-Researcher in the OSS deep-research landscape

| Project | Stars | Distinctive shape | Wins on | Loses on |
|---|---|---|---|---|
| **GPT-Researcher** | ~26.8k | Planner+editorial team+MCP-bidirectional | Coverage, citations, openness | HLE / BrowseComp absolute scores |
| OpenAI Deep Research | (closed) | Hosted product, o3-deep-research | Raw reasoning | Local docs, self-host, cost |
| Perplexity Pro | (closed) | Consumer product | UX, breadth | Harness control |
| Gemini Deep Research | (closed) | Consumer + Workspace | Multi-modal, reach | Harness control |
| [dzhng/deep-research](https://github.com/dzhng/deep-research) | ~18.6k | Minimal TS reference | Code clarity, ease of fork | Production polish |
| [langchain-ai/open_deep_research](https://github.com/langchain-ai/open_deep_research) | ~10.5k | LangGraph reference | LangChain ecosystem | Less feature-complete than GPTR |
| [huggingface/smolagents](https://github.com/huggingface/smolagents) (`open_deep_research` example) | ~26-27k | Code-as-action agents | 55% GAIA pass@1 | Less polished output |
| [Alibaba-NLP/DeepResearch](https://github.com/Alibaba-NLP/DeepResearch) (Tongyi) | ~18.7k | Trained 30.5B MoE + agent | HLE / BrowseComp leaderboard | Bundled model+harness; less generic |
| [bytedance/deer-flow](https://github.com/bytedance/deer-flow) | ~45.6k | Long-horizon SuperAgent | Sandbox, code, video | Broader scope; less research-specialist |
| [GAIR-NLP/DeepResearcher](https://github.com/GAIR-NLP/DeepResearcher) | ~750 | RL-trained on real web | Emergent honesty | Single-recipe; less production-shape |
| [RUC-NLPIR/WebThinker](https://github.com/RUC-NLPIR/WebThinker) | ~1.4k | Reasoning-model-driven | Long-CoT integration | Narrower than GPTR |
| [PeterGriffinJin/Search-R1](https://github.com/PeterGriffinJin/Search-R1) | ~4.6k | RL training framework | RL recipe quality | Framework, not finished agent |
| [microsoft/agent-framework](https://github.com/microsoft/agent-framework) | rising | MAF + Azure Foundry o3-deep-research | Microsoft ecosystem | Closed reasoning model dependency |
| Polaris (in-tree) | n/a | Disciplined deep research | Trust tiers + 20 BL-* gates + structural provenance | Earlier-stage; ramping multi-hop substrate per [203](203-polaris-multi-hop-reasoning-apply-plan.md) |

**Read row-by-row.** GPT-Researcher's competitive position: **strongest open citation/coverage**, **mid-tier on raw reasoning depth** (loses to Tongyi / OpenAI on HLE / BrowseComp), **best documented MCP integration** in the OSS space.

## §8 — Failure modes and limitations

- **Underperforms on hard agentic benchmarks.** GPT-Researcher loses to Tongyi DeepResearch and OpenAI Deep Research on HLE and BrowseComp because GPTR is a **planner-orchestrator, not an RL-trained reasoner**. Deep multi-hop browsing is weaker than RL-trained competitors.
- **Shallow loops.** When the planner generates redundant sub-questions, the run becomes a multi-token-cost echo of one question.
- **Citation drift.** URLs cited for *adjacent* claims when the Writer paraphrases across chunks. Faithfulness ≠ source-position-correctness.
- **Tavily lock-in / cost.** Default retriever; large runs hit Tavily quotas. Mitigated by pluggable retrievers but the docs/examples lean on Tavily.
- **Token blow-up in Deep Research recursion.** Without strict breadth × depth caps, recursive expansion explodes cost.
- **Hallucinated citations.** 20–30% fabrication rate documented for raw GPT-class models; retrieval grounding mitigates but doesn't eliminate. Industry-wide problem; GPTR is no exception.
- **CVE-2026-5633.** Open security report (issue #1696). Production deployers must track.
- **No first-party trust-tier system.** All retrieved sources treated equivalently; no T1-PEER-REVIEWED / T2-PREPRINT / T3-DISCUSSION typed metadata. (Polaris's distinctive surface; cf. [172-polaris-2026-deep-research-roadmap](172-polaris-2026-deep-research-roadmap.md).)
- **No retraction-aware retrieval.** A retracted paper enters the chain like any other source. (Helix-Bio's distinctive surface; cf. [219-helix-bio-multi-hop-collaborative-apply-plan](219-helix-bio-multi-hop-collaborative-apply-plan.md).)
- **No equal-budget benchmark discipline.** Single-vs-multi-agent comparisons in the docs aren't compute-controlled.

## §9 — Eight production patterns worth lifting

For any team building a deep-research agent (Polaris, Atlas-Research, Helix-Bio, in-tree projects):

1. **Chief-Editor LangGraph pattern.** Editorial team with information-asymmetric roles (Researcher / Reviewer / Revisor / Writer / Publisher). Survives [202](202-multi-agent-multi-hop-reckoning-2026.md) equal-budget critique because asymmetry is structural.
2. **Pluggable retriever interface.** Tavily, MCP, local, BYO behind the same contract. The single most-copied pattern.
3. **Hybrid (web+local) as a first-class mode.** Not bolt-on RAG. Avoids isolation bias.
4. **MCP both directions.** Be a host *and* expose your harness as a server. Bidirectional makes the agent a citizen of the broader ecosystem.
5. **Recursive tree planner with breadth/depth knobs.** User-controllable budget; prevents token blow-up.
6. **Multi-format Publisher as terminal node.** PDF/DOCX/Markdown/HTML; templates preserve citations.
7. **NextJS frontend with WebSocket streaming.** Visible token-by-token research progress; the user-facing differentiator from a black-box agent.
8. **DeepResearchGym-style eval harness in CI.** Citation/coverage regression detection. Make this regression-gated, not aspirational.

## §10 — Implications for harness engineering

- **Adopt the editorial team pattern.** Information-asymmetric roles (Researcher/Reviewer/Revisor/Writer) generalise beyond research — code review, ops post-mortem, design doc co-author. Cf. [02-subagent-delegation](02-subagent-delegation.md), [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md).
- **MCP both directions for any agent worth keeping.** Cf. [07-model-context-protocol](07-model-context-protocol.md), [206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md) §3.
- **Pluggable retriever contract.** Standardise on a `Retriever` interface; let the user pick Tavily / SerpAPI / MCP / local at runtime. Cf. [25-agentic-rag](25-agentic-rag.md).
- **Hybrid web+local as default.** The isolation between web RAG and local-doc RAG is artificial; merge in the curator step.
- **Recursive tree mode with budget caps.** Breadth × depth is the user-visible knob; budget cap is the safety rail. Cf. [156-heavyskill-parallel-reasoning-deliberation](156-heavyskill-parallel-reasoning-deliberation.md), [199-multi-hop-reasoning-techniques-arc](199-multi-hop-reasoning-techniques-arc.md).
- **Streaming UI as default.** WebSocket → token-by-token output. Hides latency; makes the agent feel responsive. Cf. [143-ux-design-for-agentic-systems](143-ux-design-for-agentic-systems.md).
- **DeepResearchGym (CMU 2025) in CI.** Citation/coverage regression detection. Cf. [115-evaluating-llm-systems](115-evaluating-llm-systems.md), [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md).
- **Layer trust tiers + retraction gates on top.** What GPTR doesn't ship — Polaris's typed trust tiers ([172](172-polaris-2026-deep-research-roadmap.md)) and Helix-Bio's retraction gate ([219](219-helix-bio-multi-hop-collaborative-apply-plan.md)) — are the obvious next layer. Adopt the GPTR base + add the trust/retraction surface for high-stakes domains.
- **Equal-budget compare against GPTR.** Before claiming improvements over GPT-Researcher, run the comparison under [202](202-multi-agent-multi-hop-reckoning-2026.md) §3 equal-budget control. Token cost is a confounder.
- **Don't lock in to one retriever.** Tavily-first is GPTR's pragmatic default; in-tree projects should design retriever-agnostic from day one.
- **Multi-format publisher templates.** Treat publisher output as a typed object with format-specific renderers; preserve citations across formats.
- **Track CVEs and security advisories.** Deep-research agents process untrusted web content; supply-chain attack surface is real. Cf. [22-guardrails-prompt-injection](22-guardrails-prompt-injection.md), [82-poisonedrag](82-poisonedrag.md), [175-agent-skills-ecosystem-and-security](175-agent-skills-ecosystem-and-security.md).
- **Polaris / Atlas / Lyra apply plan integration.** Lift GPT-Researcher's planner+editorial pattern into Polaris's deep-research surface ([203](203-polaris-multi-hop-reasoning-apply-plan.md)) and Atlas's relaxed-gate variant ([218](218-atlas-research-multi-hop-collaborative-apply-plan.md)). Add MCP-bidirectional to argus's marketplace integration plan ([209](209-argus-multi-hop-collaborative-apply-plan.md)).

## §11 — When to use GPT-Researcher (or its patterns)

**Use GPT-Researcher directly** when (a) you want a polished open-source deep-research agent today with citation grounding and multi-format output, (b) you can self-host or accept the hosted offering's terms, (c) your domain is general research (not regulated industries needing trust tiers / retraction-aware gates), (d) Tavily-as-default-retriever is acceptable.

**Lift the patterns instead** when (a) you need typed trust tiers (Polaris), (b) you need retraction-aware retrieval (Helix-Bio), (c) you need a different gate surface entirely, (d) you need different licensing or compliance posture. Most in-tree projects (Polaris, Atlas, Helix, Lyra) are in this category.

**The two approaches are not exclusive** — a common pattern is "GPT-Researcher base + lift one or two distinctive surfaces from in-tree projects."

## §12 — Cross-references

- [166-research-agent-frameworks-synthesis](166-research-agent-frameworks-synthesis.md) — predecessor synthesis on research-agent frameworks (where GPT-Researcher is one entry).
- [158-deep-research-agents-survey-huang-et-al](158-deep-research-agents-survey-huang-et-al.md), [159-deep-research-survey-zhang-et-al](159-deep-research-survey-zhang-et-al.md), [160-deep-researcher-agent-24x7](160-deep-researcher-agent-24x7.md), [161-paper-researcher-ai-agent](161-paper-researcher-ai-agent.md), [162-paper2agent-reimagining-papers-as-agents](162-paper2agent-reimagining-papers-as-agents.md), [163-deer-flow-revisited-may-2026](163-deer-flow-revisited-may-2026.md), [164-crewai-multi-agent-framework](164-crewai-multi-agent-framework.md), [165-ralph-autonomous-loop](165-ralph-autonomous-loop.md) — adjacent deep-research / framework deep-dives.
- [172-polaris-2026-deep-research-roadmap](172-polaris-2026-deep-research-roadmap.md) — Polaris roadmap (the in-tree disciplined-research alternative).
- [203-polaris-multi-hop-reasoning-apply-plan](203-polaris-multi-hop-reasoning-apply-plan.md), [218-atlas-research-multi-hop-collaborative-apply-plan](218-atlas-research-multi-hop-collaborative-apply-plan.md), [219-helix-bio-multi-hop-collaborative-apply-plan](219-helix-bio-multi-hop-collaborative-apply-plan.md) — apply plans for in-tree research projects.
- [199-multi-hop-reasoning-techniques-arc](199-multi-hop-reasoning-techniques-arc.md), [200-graph-grounded-multi-hop-retrieval](200-graph-grounded-multi-hop-retrieval.md) — multi-hop substrate context.
- [206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md) §2 — multi-agent collaboration patterns.
- [202-multi-agent-multi-hop-reckoning-2026](202-multi-agent-multi-hop-reckoning-2026.md) — equal-budget critique that GPT-Researcher's editorial-team pattern survives.
- [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md), [187-multi-agent-shared-memory-landscape](187-multi-agent-shared-memory-landscape.md), [98-diversity-collapse-mas](98-diversity-collapse-mas.md) — coordination + diversity context.
- [205-lobehub-collaborative-teammate-platform](205-lobehub-collaborative-teammate-platform.md) — the consumer-shell counterpart (LobeHub is the chat UI; GPT-Researcher is the research backend).
- [42-langchain-deep-agents](42-langchain-deep-agents.md), [126-frameworks-comparison](126-frameworks-comparison.md), [144-build-your-own-harness](144-build-your-own-harness.md) — framework / harness context.

## §13 — One-paragraph summary

GPT-Researcher (~26.8k★, MIT, v3.4.4 April 2026) is the **OSS deep-research reference platform** — a Python+TypeScript framework with a planner-runner-curator-publisher single-agent pipeline, a Chief-Editor LangGraph multi-agent pattern (Researcher → Reviewer → Revisor → Writer → Publisher with information-asymmetric roles that survive the [202](202-multi-agent-multi-hop-reckoning-2026.md) equal-budget critique), a recursive Deep-Research tree mode with breadth+depth knobs, hybrid web+local document research, and bidirectional MCP integration (it's both an MCP host consuming external retrievers and an MCP server exposing `deep_research` / `quick_search` / `write_report` tools to Claude Desktop and others). CMU's DeepResearchGym (May 2025) ranked it top of class on citation quality and coverage vs. Perplexity / OpenAI / OpenDeepSearch / HuggingFace. It loses to Tongyi DeepResearch and OpenAI Deep Research on HLE / BrowseComp because GPTR is a planner-orchestrator rather than an RL-trained reasoner; it lacks Polaris's typed trust tiers and Helix's retraction-aware gates. Eight production patterns worth lifting: editorial-team multi-agent, pluggable retrievers, hybrid web+local, MCP both directions, recursive tree with budget caps, multi-format publisher, NextJS streaming UI, DeepResearchGym in CI. Polaris / Atlas / Helix should benchmark against GPT-Researcher on DeepResearchGym before claiming improvements; Polaris and Helix should layer their distinctive trust/retraction gate surfaces on top of GPT-Researcher's pattern bin rather than rebuilding from scratch.

**The one-line takeaway for harness designers:** Treat GPT-Researcher as the **OSS pattern bin for deep research** — adopt the planner+editorial-team+MCP-bidirectional+recursive-tree+pluggable-retriever stack verbatim, layer your domain's distinctive gate surface on top (typed trust tiers for Polaris, retraction-aware retrieval for Helix-Bio, dual-use intent classifier for biomed, audit trail for ops), and benchmark against DeepResearchGym before claiming you beat it.
