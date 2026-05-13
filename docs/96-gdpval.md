# 96 — GDPval: Evaluating AI on Real-World Economically Valuable Tasks

**Paper.** Tejal Patwardhan, Rachel Dias, Elizabeth Proehl, Grace Kim, Michele Wang, Olivia Watkins, Simón Posada Fishman, Marwan Aljubeh, Phoebe Thacker, and colleagues — *GDPval: Evaluating AI Model Performance on Real-World Economically Valuable Tasks* — OpenAI — arXiv:2510.04374v1 [cs.LG] — 5 October 2025.

**One-line definition.** A benchmark of 1,320 tasks across 44 occupations spanning the nine U.S. sectors with the largest GDP share, each defined by an expert request plus reference files and a professional deliverable, evaluated primarily with blinded pairwise human judgments of model output versus human expert work (and an open 220-task gold subset with an experimental automated grader).

## Why this paper matters (the new bar — economic value rather than reasoning benchmark abstraction)

GDPval is positioned as a complement to “exam-style” capability tests and narrow vertical coding benchmarks: the measured quantity is not abstract reasoning score but the frequency with which a model’s deliverable is ranked at least tied with a human expert’s on realistic, O*NET-anchored professional work. The preprint is explicit that leading models sit far below saturated academic leaderboards: on the gold subset, Claude Opus 4.1 reaches 47.6% of cases where the model is strictly preferred or tied with the human reference, and the narrative characterizes aggregate frontier behavior as competitive on just over half of tasks under that protocol. For OpenAI’s own line, Table 2 reports strict win rate *w* (model strictly better, not tied) of 39.0% for GPT-5, 35.2% for o3, 29.1% for o4-mini, and 12.5% for GPT-4o on the same gold slice. Those mid-band percentages are a feature, not a bug: they show that when tasks require multi-hour, multi-attachment, multi-modal artifacts whose quality is assessed by people who do the job for a living, “frontier” systems are closer to partial substitution than to mastery—aligning eval with economic consequence rather than with reasoning-benchmark abstraction.

## Problem it solves (gap between benchmark scores and real economic productivity)

Macro studies of adoption, usage, and AI-attributed growth are necessary but often lag and confound organization-level frictions. Academic benchmarks that reward correctness on closed-form items can overstate the mapping to professional productivity, which is jointly determined by format fidelity, client-appropriate style, and integration of scattered inputs. GDPval compresses the chain from model behavior to a labor-market-relevant object: a deliverable with imputed cost from OEWS wages and self-reported (reviewed) time, compared head-to-head to an expert on identical specifications.

## Core idea in one paragraph

For each of 1,320 full-set (and 220 public gold) items, the unit is a request plus a target deliverable, often with many reference files (mean near two on gold with heavy tails, up to 38 in the full set; the abstract also mentions 17 in one summary line). Industry experts (mean ~14 years experience) author tasks; each passes model-assisted screening and multiple human review rounds (full set: average ~5 human reviews, minimum 3). Models are run with tool and environment choices documented in the paper (e.g. OpenAI with web search and code interpreter; Claude via a UI path to exercise file-creation features). The primary score is win rate from pairwise comparison; the paper frames win rate as a non-saturating metric because the baseline can evolve from “human” to a stronger model over time. An automated GPT-5-based grader is released for the gold set but measured at ~66% agreement with humans versus ~71% human–human, i.e. within ~5 points of the inter-rater ceiling—not a full replacement for occupational experts.

## Mechanism (step by step)

**(a) Curation: experts and real deliverables.** Sectors: those contributing more than 5% of U.S. GDP in Q2 2024 (FRED value-added by industry). Within each sector, select five high wage-bill occupations classified as “predominantly digital” if at least 60% of O*NET tasks (weighted by relevance, importance, and frequency) are labeled digital using GPT-4o; validate that digital share correlates with non-routine cognitive and anti-manual patterns in the Acemoglu–Autor (2011) task-content decomposition (Appendix A.7.1). Experts must have ≥4 years in-role, pass interview and checks, and map each task to O*NET for coverage. Task dollar value = estimated hours × OEWS median wage for the occupation. Full set: 1,320 tasks = 30 × 44 occupations; public gold: 220 = 5 × 44, prompts and files released, expert-identifying material scrubbed.

**(b) Nine sectors / 44 occupations.** Table 1 reports GDP shares and top compensated roles, including (among others) Real Estate and Leasing ~13.8%, Government ~11.3%, Manufacturing ~10.0%, Professional/Scientific/Technical Services ~8.1%, Health ~7.6%, Finance ~7.4%, Retail ~6.3%, Wholesale ~5.8%, Information ~5.4%, with 44 five-occupation slices totaling roughly $3T/yr in compensation in the paper’s wage construction. Work spans CAD, A/V, spreadsheets, office documents, social copy, and similar digital artifacts—explicitly not manual labor in v1.

**(c) Pairwise human-vs-AI protocol.** Grading on gold uses additional occupational experts who compare unlabeled human and model deliverables given the same request and references. Each gold comparison is expensive: the authors state over one hour on average. Comparisons are intended to be blind, but the paper documents leakage risks (em dash frequency, Claude first-person, Grok self-reference), so the procedure is not information-theoretically perfect blinding. Expert justifications support qualitative failure clustering.

**(d) Win / tie / loss.** Appendix A.6.1 formalizes {0, 0.5, 1} for human, tie, and model. Section 3.1’s 47.6% for Claude counts wins and ties; Table 2’s *w* is the probability the model is strictly better—do not conflate the two in cross-benchmarking.

**(e) Cost and price per task (gold).** Reported means on the 220-gold analysis: human completion ~404 minutes and ~$361; expert review of a model output ~109 minutes and ~$86; model time and cost from API telemetry for OpenAI systems. Table 2 contrasts naive human/model time ratios (e.g. GPT-5 ~90×) with “try once then complete yourself” and “try n times” expectations that land near 1.1–1.4× realized speedup once review and fallbacks are included at measured w. Cost estimates for Claude, Gemini, and Grok are not in that table. Full-set per-task value averages ~$391 with a long right tail (max in thousands to tens of thousands in appendix tables), underscoring that catastrophic tail risk is real even when mean ratios look modest.

## Empirical results (table — frontier models on GDPval; specific win rates for Claude Opus 4.1, GPT-5, Gemini 2.5 Pro)

| Model (paper naming, Oct 2025 preprint) | Reported number | Type |
| ---------------------------------------- | --------------- | ---- |
| Claude Opus 4.1 | 47.6% | share of tasks with model win or tie vs expert (§3.1) |
| GPT-5 | 39.0% | strict win rate w (Table 2) |
| o3 | 35.2% | strict w (Table 2) |
| o4-mini | 29.1% | strict w (Table 2) |
| GPT-4o | 12.5% | strict w (Table 2) |
| Gemini 2.5 Pro | not tabulated in prose | figures: overall below Claude; frequent instruction/reference failures in clustering |
| Grok 4 | not tabulated in prose | figures: similar IF failure pattern; not worst on accuracy in expert clustering |

Sector breakdown (Appendix, Fig. 10): Government, Retail Trade, and Wholesale Trade are named where the strongest systems approach parity; other sectors can remain low for all models. By deliverable (Fig. 12, A.2.4): Claude leads on most file types; GPT-5 (high) leads on pure text though absolute win rates on text stay low. By duration (Fig. 13): win rates are highest for 0–2 hour expert-time tasks and decline with longer tasks. The request referred to “Gemini-3.1” and “Claude-4.5” naming: this arXiv version evaluates Gemini 2.5 Pro and Claude Opus 4.1.

## Variants and ablations

Reasoning: for o3 and GPT-5, higher reasoning effort monotonically improves results (Figure 9a). Prompting and environment: a general “rigorous formatting and LibreOffice-based PDF, render-to-PNG self-check, reduce unicode hazards” style prompt, plus best-of-4 with a GPT-5 judge and allowing GET in the container, largely removes “black square” PDF artifacts, cuts severe PowerPoint issues from 86% to 64%, raises agents actually inspecting deliverables as images from 15% to 97%, and lifts human-preference win rates by about 5 percentage points (Figure 9b). Context: the “under-contextualized” GDPval experiment (~42% token length) worsens GPT-5 by forcing discovery of what to do, isolating long-horizon scoping that full prompts hide (A.2.7). Automated grader: 65.7% vs 70.8% inter-rater; grader is least aligned for strong OpenAI outputs, consistent with known self-favoritism. Failure relabeling on GPT-5 losses: most losses are “acceptable but subpar”; ~29% bad or catastrophic with ~3% catastrophic (A.2.6 narrative).

## Failure modes and limitations (judge bias, sector coverage, real economic value vs deliverable proxy)

Blinding is imperfect; subjective dimensions (aesthetics, taste, audience fit) dominate many professions. Inter-rater agreement near 71% caps what any automated grader can achieve; the released grader also omits 12/220 open tasks and struggles with internet-only, some polyglot runtimes, font rendering, and speech. Coverage is 44 digital-heavy occupations, not the full economy; v1 is self-contained, one-shot, and largely well-specified (~89% well-specified tasks), so it under-weights the real job of eliciting requirements. A polished in-benchmark artifact is only a proxy for liability, client trust, and organizational workflow. Table 2’s “naive” speedups are pedagogically there to be discounted once human review and rework enter—catastrophic mistakes are called out as under-modeled in ratio analyses.

## When to use, when not

Use when evaluating whether a harness produces executive-grade files across heterogeneous formats, and you can pay expert time or use gold+auto-grader for trends. Do not use as licensure, safety certification, or full macro forecast; not a substitute for OS desktop control tests ([95-osworld](95-osworld.md)); not globally representative beyond the U.S. BLS-based construction.

## Implications for harness engineering

Treat GDPval as a third family of target alongside [95-osworld](95-osworld.md) (computer-use trajectories) and SWE-bench–style code patches: *operate the machine*, *ship the code*, *produce the client-facing artifact*. [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md) maps directly: expert pairwise grading is the validity anchor; any LM grader in the loop inherits bias patterns like those discussed with Panickssery et al. [38-claw-eval](38-claw-eval.md) is adjacent in spirit—tool-rich containers matter—but GDPval’s evidence is that “deep” scientific stacks and format pipelines (the paper’s long Docker dep list) plus sampling and judge scaffolds move metrics as much as base-model version bumps. [27-horizon-long-horizon-degradation](27-horizon-long-horizon-degradation.md) is reinforced by their duration and under-context ablations: harnesses for production agents need staged information discovery, self-render QA, and explicit handling of long expert-time tasks, not just more tokens. Net: product harnesses that optimize only OSWorld or only SWE miss the GDPval surface—file craft, client polish, and oversight economics.

## Connections to other work in this corpus

- Labor-market adoption literatures (Clio, NBER “How people use ChatGPT,” Anthropic economic index) are complementary leading indicators; GDPval is a controlled outcome measure.  
- LLM-judge and trajectory-judge works ([21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md)) reappear in the automated grader analysis.  
- [95-osworld](95-osworld.md) and GDPval are orthogonal axes (environment interaction vs. professional deliverable quality).  
- Meta TTS and similar inference-time structure ([77-meta-tts-agentic-coding](77-meta-tts-agentic-coding.md)) differ in domain (coding trajectories) but share the high-level point that test-time organization—here formatting prompts, best-of-N, and visual self-checks—lifts long outputs without retraining.

## Key takeaways

1. 1,320 / 44 / 9 GDP sectors; gold 220; expert-authored, O*NET-mapped, wage-imputed.  
2. Mid-40% win∪tie (Claude 4.1) vs ~39% strict-win w (GPT-5) on their reported slices—expert-graded, not saturated leaderboards.  
3. “Naive” model speed is misleading; review + fallback collapses speedups in Table 2.  
4. File-type and duration effects are large; prompting + rendering checks yield multi–percentage-point lifts.  
5. Open data + public grader with documented limitations; human grading remains the recommendation.

## References

1. Patwardhan, T., et al. (2025). *GDPval: Evaluating AI Model Performance on Real-World Economically Valuable Tasks.* arXiv:2510.04374v1.  
2. U.S. Bureau of Labor Statistics, O*NET, OEWS, National Employment Matrix (sources as cited in paper).  
3. Federal Reserve Bank of St. Louis, FRED—value added by industry (GDP share), accessed as cited.  
4. Panickssery, A., Bowman, S. R., & Feng, S. (2024). *LLM evaluators recognize and favor their own generations.* arXiv:2404.13076.  
5. Acemoglu, D., & Autor, D. H. (2011). Skills, tasks and technologies. *Handbook of Labor Economics*, 4.  
6. OpenAI. Automated grading interface: `https://evals.openai.com` (as stated in the paper; verify for your run).
