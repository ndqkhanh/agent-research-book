# 101 — AutoResearchBench: Benchmarking AI Agents on Complex Scientific Literature Discovery

**Paper.** Lei Xiong, Kun Luo, Ziyi Xia, Wenbo Zhang, Jin-Ge Yao, Zheng Liu, Jingying Shao, Jianlyu Chen, Hongjin Qian, Xi Yang, Qian Yu, Hao Li, Chen Yue, Xiaan Du, Yuyang Wang, Yesheng Liu, Haiyu Xu, Zhicheng Dou — *AutoResearchBench: Benchmarking AI Agents on Complex Scientific Literature Discovery* — arXiv:2604.25256v1 — Renmin University of China — April 28, 2026. Code: https://github.com/CherYou/AutoResearchBench.

**One-line definition.** AutoResearchBench is a 1,000-task benchmark over 3M arXiv papers covering two complementary modes — **Deep Research** (precise single-paper identification under multi-clue conjunctive constraints, 600 tasks; 10% no-answer) and **Wide Research** (exhaustive set discovery scored by IoU, 400 tasks; mean 9.23 valid papers per query) — where top frontier models score 9.39% / 9.31%, an order-of-magnitude collapse from their 80%+ scores on general web-browsing benchmarks like BrowseComp.

## Why this paper matters

For two years the agent community has used GAIA and BrowseComp as the live-task ceiling, watched leaders climb past 80%, and concluded that long-horizon web reasoning is increasingly *solved*. AutoResearchBench shows the celebration is premature: those benchmarks reward shallow matching on common-sense queries with salient answers in titles and abstracts, and that pattern is structurally different from what scientific researchers actually do. Researchers don't search for *any* paper on a topic; they search for papers satisfying a **hidden conjunction of technical constraints** — methods, ablation results, reference relations, statistical artefacts, author trajectories — almost always buried in method sections, ablation tables, figure captions, appendices, and citation chains.

The empirical drop between BrowseComp and AutoResearchBench is the headline. Claude-Opus-4.6 and Gemini-3.1-Pro-Preview clear 80% on BrowseComp and land at 7–9% here. The drop is not an artefact of harder questions; it is an artefact of harder *evidence access*. The benchmark forces full-text reasoning, multi-hop scientific verification, and disciplined judgement about set completeness — all dimensions GAIA-style benchmarks systematically under-test. Take this paper seriously and three downstream things change. (1) Claims about deployed AI scientists, research assistants, and lab-automation agents need new evidence — BrowseComp scores no longer license them. (2) Search-tool design becomes a first-class harness lever: a controlled full-text index (DeepXiv) outperforms generic web search by 1.5–3 pp consistently. (3) "Think harder" — extended chain-of-thought, longer trajectories — *does not transfer* to literature-discovery accuracy; it just burns latency.

The benchmark also closes a methodological gap. RealScholarQuery and SPAR are too small and domain-narrow. WideSearch and DeepWideSearch lack a controlled corpus, so fine-grained verification is impossible. SAGE has scale but no interactive evaluation. AutoResearchBench is the first to combine 1,000 expert-curated tasks, a contamination-resistant 3M-paper arXiv index, dynamic agent interaction, and *both* precise-identification and exhaustive-collection paradigms in one suite.

## Problem it solves

Three structural reasons general web-browsing benchmarks miss the scientific frontier:

1. **Research-orientation.** Scientific tasks demand in-depth understanding of methods, statistics, and proof structure. Knowledge is technical, domain-specific, and fast-moving. The benchmark deliberately constructs queries from *non-headline* clues — methodological choices, proof details, ablation observations — so keyword matching on titles or abstracts is insufficient.
2. **Literature-focus.** Decisive evidence sits in tables, figures, reference lists, appendices, proof details, result distributions — *full-text* artefacts. A benchmark that stops at metadata or abstracts trains agents to be wrong in a particular flat way.
3. **Open-endedness.** The number of qualifying papers is unknown and can be zero. Agents must decide *when to stop* — Deep Research with 10% no-answer tasks tests rigorous abstention; Wide Research scored by IoU forces precision/recall trade-offs against an unknown set boundary.

## Core idea in one paragraph

Build a benchmark whose construction pipeline is itself the artefact: take real, technically substantive arXiv papers as targets, mine *non-headline* clues from full text and citation graphs, fuzzify them so lexical shortcuts disappear, prune them iteratively against the corpus until only materially discriminative constraints remain, then verify with a separate human-and-LLM dual-track check that no shallow reformulation makes the task trivial. Run agents in a controlled environment over a 3M-paper arXiv index with a custom DeepXiv search tool that exposes full text, score them with strict exact-match for Deep Research and IoU for Wide Research, and report task accuracy plus average turns. The outcome is a benchmark that exposes whether agents *reason* about papers or merely *retrieve* them.

## Mechanism (step by step)

### Task formulation

Given query q and corpus D, an agent interacts with the search environment and outputs a predicted answer set Ŷ(q) approximating Y*(q) = {d ∈ D : d ⊨ q} — the papers in D satisfying every constraint in q. State at step t is s_t = (q, h_t, D_t), where h_t is the interaction history and D_t ⊆ D the documents observed so far.

Two paradigms with different |Y*(q)| structure:
- **Deep Research** — 600 tasks, 90% with |Y*(q)| = 1, 10% with |Y*(q)| = 0. The agent must identify the unique qualifying paper or rigorously establish that none exists.
- **Wide Research** — 400 tasks, mean 9.23 valid papers (range 2–34). The agent must trace the boundary of a target concept through the corpus while preserving precision.

### Construction pipelines

**Deep Research (4 stages).**
1. Sample a technically substantive CS paper (typically 10–100 citations; surveys/technical reports excluded — scope weakens identifiability).
2. Mine non-headline clues from full text and citation graph: methodological choices, proof details, local empirical observations, author trajectory. Citation relations are promoted into explicit subproblems (multi-hop expansion).
3. Fuzzify constraints to suppress lexical shortcuts (topic-level and detail-level paraphrase). Iteratively prune against the corpus, keeping only materially discriminative ones (minimal-sufficiency principle).
4. Verify: instances solved by frontier LLMs and humans in the same DeepXiv environment. Retain only if (i) gold answer is supported by explicit textual evidence (or no valid paper exists for no-answer cases), (ii) plausible alternatives are systematically ruled out, (iii) no shallow reformulation makes the task trivial.

**Wide Research (4 stages, entity-graph-based).**
1. Define a high-level CS theme; retrieve a candidate pool; filter and summarize with an LLM.
2. Extract shared multidimensional attributes (method, datasets, results) from candidates to construct an entity graph; translate into a query encoding a strict conjunction of constraints.
3. Rewrite the query as natural scientific intent while preserving the logical constraints. Human annotators verify alignment and manually augment with missing valid papers.
4. Iteratively expand the set via search tools. Each newly retrieved document undergoes full-text analysis; admission requires unanimous consensus of three advanced LLMs. Expansion halts when no new valid candidates emerge. Human experts audit the final set, purging marginal violators.

### Evaluation infrastructure

Controlled environment over **3M+ arXiv papers** with full-text extraction. The agent has access to **DeepXiv search**, a custom tool exposing full text rather than simplified metadata. The benchmark also evaluates against open-web search (jina) for comparison, and supports a 50-query subset for end-to-end commercial systems (GPT Deep Research, AI Studio Gemini, Alphaxiv).

### Metrics

- **Deep Research — Accuracy.** Acc = (1/|Q|) Σ_q 𝟙[Ŷ(q) = Y*(q)]. Strict exact match; partial correctness scores zero.
- **Wide Research — IoU.** IoU = (1/|Q|) Σ_q |Ŷ(q) ∩ Y*(q)| / |Ŷ(q) ∪ Y*(q)|. Holistic; rewards systematic enumeration without out-of-scope inclusions. Unlike top-k ranking, it makes false positives and missed entries equally painful.

## Empirical results

### Headline (Table 2)

| Model | Deep (Acc %) | Turns | Wide (IoU %) | Turns |
|---|---:|---:|---:|---:|
| Claude-Opus-4.6 | **9.39** | 28.1 | 6.56 | 27.11 |
| Gemini-3.1-Pro-Preview | 7.93 | 24.4 | **9.31** | 4.55 |
| GPT-5.4 | 7.44 | 6.1 | 8.12 | 3.69 |
| Seed-2.0-Pro | 6.80 | 22.9 | 7.87 | 4.15 |
| Qwen3.5-397B-A17B | 6.97 | 27.4 | 3.83 | 7.11 |
| Claude-Sonnet-4.6 | 6.96 | 27.5 | 5.83 | 19.0 |
| Deepseek-V3.2 | 4.21 | 28.8 | 7.70 | 6.25 |
| Kimi-K2.5 | 4.69 | 27.0 | 6.23 | 8.35 |
| Qwen3.5-122B-A10B | 3.88 | 26.2 | 2.76 | 5.39 |

**End-to-end systems on 50-query subset.** GPT Deep Research solves 22/50 Deep tasks (44%) at IoU 4.06 on Wide; AI Studio Gemini-3.1-Pro 14/50 (28%) at IoU 4.84; Alphaxiv 0/50 at IoU 4.31. The specialized end-to-end products do better than ReAct frameworks on Deep but plateau well below 50%, and never exceed 5% IoU on Wide.

Four observations:
- **Order-of-magnitude collapse vs. general benchmarks.** Top frontier scores under 10% on both metrics, despite the same models clearing 80%+ on BrowseComp.
- **Longer trajectories don't help.** GPT-5.4 hits 7.44% with 6.1 turns; Deepseek-V3.2 hits 4.21% with 28.8 turns. Extended reasoning turns into redundant queries and invalid branches.
- **Reasoning is the bottleneck, not retrieval.** Failures are not "agent omitted relevant papers." Agents identify plausible candidates but cannot precisely verify constraints, eliminate boundary cases, or integrate fragmented evidence.
- **Comprehensiveness gap on Wide.** Claude-Opus-4.6 expands aggressively (27.11 turns) but fails to filter violators (IoU 6.56). Seed-2.0-Pro and GPT-5.4 terminate prematurely (4.15 / 3.69 turns), missing valid set members.

### Search-tool ablation (Table 3)

| Model | Tool | Deep (%) | Wide IoU (%) | Tokens |
|---|---|---:|---:|---:|
| Gemini-3-flash | WebSearch | 2.01 | 3.99 | 9,574 |
| Gemini-3-flash | DeepXiv | 2.75 | 6.61 | 14,052 |
| Gemini-3.1-pro | WebSearch | 6.82 | 7.37 | 6,744 |
| Gemini-3.1-pro | DeepXiv | 7.93 | 9.31 | 12,206 |
| Seed-2.0-pro | WebSearch | 3.96 | 4.18 | 14,915 |
| Seed-2.0-pro | DeepXiv | 6.80 | 7.87 | 14,444 |
| Deepseek-V3.2 | WebSearch | 3.09 | 4.78 | 14,959 |
| Deepseek-V3.2 | DeepXiv | 4.21 | 7.70 | 21,575 |

Averaged over four models, Deep accuracy drops 5.42% → 3.97% when DeepXiv is replaced by web search; Wide IoU drops similarly. Academic constraints live inside paper internals; open-web search is noisier and fragmented.

### Thinking-mode ablation (Table 4)

| Model | Mode | Deep (%) | Time (s) | Wide IoU (%) | Time (s) |
|---|---|---:|---:|---:|---:|
| Gemini-3-flash | THINK | 1.83 | 433.6 | 2.53 | 225.4 |
| Gemini-3-flash | NOT THINK | 2.75 | 236.9 | 6.61 | 64.7 |
| Qwen3-max | THINK | 2.33 | 170.9 | 4.18 | 217.6 |
| Qwen3-max | NOT THINK | 3.24 | 166.0 | 6.89 | 183.4 |
| Deepseek-V3.2 | THINK | 5.67 | 583.7 | 4.28 | 511.5 |
| Deepseek-V3.2 | NOT THINK | 4.21 | 405.7 | 5.96 | 206.7 |

THINK mode is **not consistently helpful** and is generally harmful for Wide Research, while always increasing latency and tool calls. The takeaway: in literature search, deliberation only helps when it improves *evidence acquisition* — which extended chain-of-thought rarely does on its own.

### Test-time scaling (Figure 5)

Pass@k on Deep climbs steeply through k=8 (correct paper is reachable; single-run paths are unlucky). Best@k IoU on Wide climbs slowly (repeat runs reproduce similar omissions; the underlying recall ceiling barely shifts). This split — **Deep failures are trajectory-level; Wide failures are recall-level** — is the cleanest diagnostic the benchmark offers for harness designers.

## Variants and ablations

- **Open-source vs. frontier closed-source** — closed leads by 2–5 pp consistently.
- **Specialized end-to-end vs. ReAct** — end-to-end systems beat generic ReAct frameworks on Deep but still cap at <50% on the 50-query subset, and never break 5% Wide IoU.
- **Single-turn vs. multi-turn** — 6 turns can match 28 turns; iteration count alone is not progress.
- **Implicit ablations** — running pass@k separately for Deep and Wide reveals different failure structures (trajectory vs. recall), which different harness levers can target.

## Failure modes and limitations

The paper's error analysis (Appendix F) groups failures into five categories that are themselves a useful checklist:
1. **Weak scientific reasoning** — agents cannot apply domain-specific constraints rigorously.
2. **Incomplete use of paper-level information** — agents read titles/abstracts and miss method/ablation/appendix evidence.
3. **Difficulty handling long conjunctive queries** — multiple constraints don't get integrated into one search hypothesis.
4. **Insufficient comprehensiveness** — Wide Research agents either over-expand (false positives) or under-explore (false negatives).
5. **Limited iterative reflection** — agents repeat similar searches instead of reformulating after negative evidence.

Benchmark-design caveats:
- Static arXiv snapshot — papers added later are out-of-corpus and would falsely score as misses.
- 10% no-answer tasks rely on annotator judgement; very obscure qualifying papers may be missed.
- Limited to CS (8 core areas); biology/physics literature has different structure.
- DeepXiv-platform dependency — reproducibility relies on this infrastructure being maintained.
- 50-query end-to-end sample is small (cost-driven).

## When to use, when not

**Use as the diagnostic** for any agentic system intended for scientific research, literature review, or knowledge discovery; for testing whether improvements on BrowseComp/GAIA transfer to specialized scientific tasks; for building or improving harnesses with multi-hop reasoning and constraint verification; for measuring real-research bottlenecks (constraint understanding, false-positive elimination, set completeness).

**Don't use it** for general-purpose web browsing, common-sense Q&A, abstract-only retrieval, or non-arXiv corpora with very different distributions.

## Implications for harness engineering

- **Tool design is a harness lever, not a model lever.** DeepXiv-style full-text indexing beats generic web search by 1.5–3 pp consistently across models. For scientific harnesses, building a domain-specific search backend is more impactful than swapping models. See [25-agentic-rag](25-agentic-rag.md) for the dynamic-retrieval pattern this should plug into.
- **Trajectory length is not progress.** GPT-5.4 hits 7.44% in 6 turns, Deepseek-V3.2 4.21% in 28.8 turns. Harnesses that reward turn-count or impose long horizons by default are wasting compute. Pair the loop in [01-agent-loop-architecture](01-agent-loop-architecture.md) with explicit *evidence-quality* termination conditions, not turn-count budgets.
- **Two failure modes need different fixes.** The pass@k decomposition shows Deep failures are *trajectory-level* (reachable; unlucky paths) and Wide failures are *recall-level* (systematic omissions). Trajectory failures want diversity, backtracking, hypothesis reset, MCTS-style search ([15-tree-of-thoughts-lats](15-tree-of-thoughts-lats.md)); recall failures want exhaustive constraint decomposition, iterative boundary exploration, and structured stopping criteria. A harness that uses one strategy for both fails.
- **Verification is a first-class harness component for Wide.** Wide Research demands "have I found *all* of them?" judgement. Existing harnesses mostly answer "is this answer good enough?" Wide Research forces agents to maintain a candidate set, score each against the full constraint conjunction, and decide when expansion has converged. See [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md) and [18-chain-of-verification-self-refine](18-chain-of-verification-self-refine.md) for the verifier patterns that should feed into a Wide-specific stopping rule.
- **THINK mode is not free.** The thinking-mode ablation shows extended deliberation hurts Wide Research and inconsistently helps Deep. Harness defaults that always enable extended thinking burn latency and money. Make THINK conditional on *verification calls returning ambiguous evidence*, not on task type.
- **Tools, not models, distinguish end-to-end systems.** The 50-query end-to-end subset shows GPT Deep Research and Gemini Deep Research beat generic ReAct on Deep but plateau on Wide — the gap looks like *better tool routing and verifier integration*, not better base models. See [42-langchain-deep-agents](42-langchain-deep-agents.md) for the open-source distillation of this pattern.
- **Construction pipeline as eval-harness blueprint.** AutoResearchBench's construction itself is reusable: full-text-first curation, citation-graph multi-hop expansion, fuzzification + iterative pruning to ensure no shallow shortcut, expert verification with explicit alternatives ruled out. Harness teams building domain-specific evals (legal, biomedical, financial) should adopt the same four-stage pipeline rather than the metadata-shallow defaults of [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md).
- **Where in the benchmark mix it sits.** AutoResearchBench is to scientific reasoning what [95-osworld](95-osworld.md) is to OS-level computer use and [96-gdpval](96-gdpval.md) is to GDP-occupation tasks. Harness eval suites running multiple benchmarks should report scientific-discovery scores separately — a single "agent score" that averages across BrowseComp and AutoResearchBench will silently hide the order-of-magnitude collapse the paper exposes. See [34-clawbench](34-clawbench-live-web-tasks.md) and [38-claw-eval](38-claw-eval.md) for related production-agent benchmarks.

The one-line takeaway for harness designers: **scientific literature discovery is not "BrowseComp but harder" — it is a different benchmark family, and your harness needs different levers.**
