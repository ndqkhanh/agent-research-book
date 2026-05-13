# 112 — BrowseComp: A Benchmark for Browsing Agents (and Deep Research)

**Paper.** Jason Wei, Zhiqing Sun, Spencer Papay, Scott McKinney, Jeffrey Han, Isa Fulford, Hyung Won Chung, Alex Tachard Passos, William Fedus, Amelia Glaese — *BrowseComp: A Simple Yet Challenging Benchmark for Browsing Agents* — arXiv:2504.12516 — OpenAI — April 16, 2025. Companion artifacts and dataset linked from OpenAI's announcement page.

**One-line definition.** **BrowseComp** is OpenAI's open benchmark of **1,266 fact-seeking questions** designed so each answer is short and verifiable but only findable via **persistent multi-step web browsing** — a measure of an agent's ability to do *deep research*, not just one-shot search.

## Why this paper matters

Earlier web benchmarks (Mind2Web, WebArena, [34-clawbench-live-web-tasks](34-clawbench-live-web-tasks.md)) primarily test **task automation** — fill the form, book the flight. BrowseComp tests something different: **find the answer humans can't easily find**. The answers are designed to be unreachable via the first page of five Google searches and to require chains of reasoning across multiple sources. It's the canonical evaluation for **Deep Research**-class agents (OpenAI's Deep Research, Anthropic's research mode, Perplexity Pro Research) and the benchmark on which the new wave of *research agents* are scored.

## Problem it solves

1. **Factuality benchmarks (TruthfulQA, SimpleQA)** test recall, not browsing.
2. **Web-task benchmarks** (WebArena, ClawBench) test side-effecting actions, not knowledge synthesis.
3. **Long-tail facts** — tucked-away references on niche pages — were never centrally evaluated.
4. **Easy verification, hard discovery.** A benchmark wants short, unambiguous answers (so grading is cheap) but hard problems. BrowseComp threads that needle.

## Core idea in one paragraph

Human trainers crafted 1,266 questions, each with a single short factual answer (a name, a year, a number) that is **indisputable** and **stable over time**. They verified that strong baselines without browsing — GPT-4o, GPT-4.5, OpenAI o1 (medium) — fail on the question. They also verified the answer is **not on the first page** of five different Google searches. The questions are organized to require chains of reasoning ("the album whose third track was covered by the band that opened for X in 1987"). Evaluation is open-ended generation, scored against the reference by exact-match-with-aliasing.

## Mechanism (step by step)

### (a) Question construction protocol

```text
For each candidate question:
  1. Trainer drafts a factual question with a short, single answer.
  2. Verify GPT-4o (no browsing) cannot answer.
  3. Verify GPT-4.5 (no browsing) cannot answer.
  4. Verify o1-medium (no browsing) cannot answer.
  5. Search Google with five different rephrasings; verify answer not on page 1.
  6. Reviewer #2 confirms answer is unambiguous.
  7. Add to dataset.
```

This anti-trivia process is what makes BrowseComp hard.

### (b) Evaluation protocol

For an agent under test, run on each of 1,266 questions; output a short answer string; score with normalized exact-match (lowercase, trimmed, alias map). Report top-1 accuracy. Also reported: **calibration** — when the agent says "I don't know," does it actually not know?

### (c) Question style

A taste (paraphrased): "Identify the only film both directed and scored by [composer], released between 1968 and 1972, that was distributed by an indie studio." The answer is a single film title — but reaching it requires composer biography, filmography, distribution-history lookups.

### (d) Anti-leak design

Questions are shuffled to break thematic patterns; releases include only the dataset, not the trainer rationale, to avoid revealing search strategies. The eval set is fixed; not a moving target.

## Empirical results

Headline (paper-reported, top-1 accuracy):

| System | BrowseComp |
|--------|----------:|
| GPT-4o (no browsing) | ~0.6% |
| GPT-4.5 (no browsing) | ~1% |
| o1-medium (no browsing) | ~5% |
| GPT-4o + browsing tool | ~2% |
| **OpenAI Deep Research** | **~50%** (paper claim) |

The 50% figure is the breakthrough: **Deep Research** — an agent specifically trained for persistent browsing — solves about half of BrowseComp, while plain GPT-4o + a `browse` tool solves nearly nothing. The gap is the value of **agent training**, not just **agent prompting**.

Subsequent published numbers (third-party retrievals via parallel.ai, BrowseComp-Plus / texttron, Galileo's BrowseComp tracking) show **closed deep-research agents** continuing to climb past 50% through 2025–2026, while OSS agents lag.

## Variants and ablations

- **Anchor model effects.** Replacing GPT-4o with a cheaper backbone in Deep Research roughly halves the score; reasoning matters as much as browsing.
- **Tool ablation.** Removing the search tool (only allowing direct URL fetch) drops scores >40 points — search is the high-leverage primitive, not raw browsing.
- **Tree-search vs sequential.** Beam-search-style multi-thread browsing improves over linear ReAct on BrowseComp, but at large compute cost.

## Failure modes and limitations

1. **Western/English bias.** Questions skew toward English-language sources; performance on non-English-language fact-seeking is not measured.
2. **Answer staleness.** Even "stable over time" answers can change (a band reunites, a record is broken). The dataset needs maintenance.
3. **Memorization confounds.** A model that has seen the BrowseComp dataset in pretraining gets credit without browsing. Evaluators should test contamination.
4. **No process credit.** A wrong answer with great research process scores 0; this hides agents that are *almost* correct.
5. **Sample bias.** The trainer pool's interests bias question topics; representativeness is approximate.

## When to use, when not

**Use BrowseComp when** evaluating a **deep research / browsing** agent and the question is "can it find hard-to-find facts?" Pair with [38-claw-eval](38-claw-eval.md) for trajectory-aware evaluation if you also want process credit.

**Don't use it** as a measure of **task automation** capability (use WebArena / ClawBench), of **short-form QA** (use SimpleQA), or of **multimodal browsing** (use MM-BrowseComp instead).

## Implications for harness engineering

- **Browsing agents need search-tool quality**, not just the ability to fetch URLs. Search engine access is the primary bottleneck.
- **Persistent state across hops** ([12-todo-scratchpad-state](12-todo-scratchpad-state.md)) — the ledger / scratchpad pattern — is what lets agents track what they've checked and what's still open.
- **Verification step is critical.** Before answering, agents should re-check the chain that produced the answer; otherwise spurious confident-wrongs.
- **Calibration matters.** Penalize confident hallucinations by training agents to emit "uncertain" when verification fails.
- **Cost per task is high.** Deep Research-class agents make hundreds of fetches per question; plan budgets accordingly.

## Connections to other work in this corpus

- **[34-clawbench-live-web-tasks](34-clawbench-live-web-tasks.md):** sister benchmark for *task* execution; complementary axis.
- **[107-browser-use](107-browser-use.md), [106-computer-use-agents](106-computer-use-agents.md):** browsing tools that an agent *uses* on BrowseComp.
- **[111-magentic-one](111-magentic-one.md):** multi-agent generalist with WebSurfer specialist — could be a reasonable architecture for BrowseComp-style tasks.
- **[25-agentic-rag](25-agentic-rag.md):** RAG with self-critique is the spiritual ancestor; BrowseComp is the live-web extreme.
- **[38-claw-eval](38-claw-eval.md):** trajectory-aware eval; pair with BrowseComp's outcome-only scoring for fuller picture.

## Key takeaways

1. **BrowseComp is the deep-research benchmark of the era** — short verifiable answers, hard browsing-required questions, 1,266 examples.
2. **Plain LLMs + browse tool ≪ Deep Research** — 1% vs 50% — *agent training* is what unlocks the capability.
3. **Search tool quality** is the dominant tool-side factor.
4. **Calibration and verification are agent-side competitive levers** for closing the gap.
5. **Process credit is missing** — combine with trajectory-aware evals where it matters.

## References

- Wei, J. et al. (2025). *BrowseComp: A Simple Yet Challenging Benchmark for Browsing Agents.* arXiv:2504.12516. https://arxiv.org/abs/2504.12516
- OpenAI announcement and dataset: https://openai.com/index/browsecomp/
- BrowseComp-Plus (a more transparent re-evaluation, ACL 2026): https://github.com/texttron/BrowseComp-Plus
- MM-BrowseComp (multimodal extension): OpenReview 2025.
- Galileo coverage: *What Is BrowseComp? OpenAI's Agent Benchmark Reveals 2026 Gaps.*
- Parallel.ai retrieval-frontier analysis: parallel.ai/blog/deep-research-benchmarks.
