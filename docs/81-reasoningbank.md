# 81 — ReasoningBank + MaTTS: Self-Evolving Agents via Reasoning Memory

**Paper.** Siru Ouyang, Jun Yan, I-Hung Hsu, Yanfei Chen, Ke Jiang, Zifeng Wang, Rujun Han, Long T. Le, Samira Daruki, Xiangru Tang, Vishy Tirumalashetty, George Lee, Mahsan Rofouei, Hangfei Lin, Jiawei Han, Chen-Yu Lee, Tomas Pfister — *ReasoningBank: Scaling Agent Self-Evolving with Reasoning Memory* — arXiv:2509.25140v2 — University of Illinois Urbana-Champaign; Google Cloud AI Research; Yale — 2025 (revised 2026). Code: https://github.com/google-research/reasoning-bank

**One-line definition.** **ReasoningBank** is a test-time memory framework that distills each trajectory into structured, reusable *reasoning* items (title, description, content) from both LLM-judged successes and failures; **Memory-aware Test-Time Scaling (MaTTS)** spends extra inference to generate multiple trajectories (parallel self-contrast) or iterative self-refinement (sequential) and *curates* memory from that diversity so scaling and memory form a positive feedback loop without weight updates.

## Why this paper matters

Long-running agents see **task streams** without IID resets; they repeat errors if each query is stateless, while finetuning is often infeasible. The open deployment question is **self-improvement without weight updates**—from representation, retrieval, and extra inference.

ReasoningBank isolates *what* is distilled: **Synapse** (trajectory memory) and **AWM** (workflows) share the same **retrieval + append** harness in the ablations; only extraction changes. **MaTTS** is not vanilla multi-sample: parallel mode uses **self-contrast**; sequential mode uses **self-refinement**—both feed higher-signal items than `MaTTS w/o aggregation`. The headline claim is **synergy**: memory quality steers scaling; scaled rollouts improve the bank.

## Problem it solves

1. **Raw trajectories** are long, noisy, and weakly transferable vs distilled strategies.
2. **Success-only / workflow memory** skips failure signal; Synapse/AWM barely gain or worsen when failures are added naïvely (Fig. 7); ReasoningBank uses both.
3. **No test labels** → need LLM-judge and robustness to label noise (Fig. 8).
4. **Vanilla multi-rollout TTS** underuses contrast (Fig. 3a); needs **MaTTS** aggregation.
5. **SR vs cost:** aim for higher SR *and* fewer steps (Tables 1–2, 4); naive RAG can add tokens without gains (Table 5).

## Core idea in one paragraph

Policy \(\pi_L(\cdot\mid M,A)\) augments ReAct (web or bash) with **ReasoningBank** \(M\). Per task: **embed-query retrieval** of prior items; execute; **LLM-judge** success/fail (no test labels); **extract** 1–3 Markdown items (success: why it worked; failure: avoid recurrence), forbidding site-specific strings. **MaTTS** uses \(k\) parallel rollouts **or** sequential refinements, then a richer extractor (up to 5 items) that performs **contrast** or mines refinement checks—so extra compute becomes **contrastive memory**, not a pile of redundant traces.

## Mechanism (step by step)

### (a) ReasoningBank schema — what is stored

Each experience contributes **memory items**, not the raw trace as the retrievable unit (though the implementation JSON also stores the originating query, trajectory, and precomputed query embedding for search).

| Field | Role |
| --- | --- |
| **Title** | Concise name of the strategy or pitfall. |
| **Description** | One-sentence abstract. |
| **Content** | 1–3 (or 1–5 under MaTTS) sentences of operational reasoning: decision rationales, checklists, and failure-prevention patterns. |

Success prompts ask the extractor to first explain *why* the run succeeded, then emit up to **three** non-overlapping items. Failure prompts require reflection, then *preventive* strategies. Items must avoid citing specific sites, queries, or literal strings to force transfer.

### (b) MaTTS algorithm

**Vanilla TTS (MaTTS w/o aggregation, Figure 3a):** run \(k\) independent trajectories with retrieved memory, convert each to items—adds items but little contrastive curation.

**Parallel MaTTS (Figure 3b, Appendix A.3, Figure 11 left):** under guidance of *current* retrieved memory, sample \(k\) trajectories for the *same* query, then a dedicated **self-contrast** extraction pass ingests the bundle, compares successes vs failures, and emits up to **five** items capturing stable patterns and mistakes. BoN (best-of-N) for reporting uses an LLM judge with explicit rubrics (progress, efficiency, loop detection, error severity) over all \(N\) trajectories (Figure 12).

**Sequential MaTTS (Figure 3c, Figure 11 right):** one trajectory is iteratively re-checked with follow-up *refinement* instructions; intermediate critique passes become extra supervisory text for memory.

Scaling factor \(k\) counts **parallel trajectories** or **refinement steps** depending on the mode. Empirically on WebArena-Shopping (Gemini-2.5-Flash), parallel SR rises from 49.7% at \(k{=}1\) to **55.1%** at \(k{=}5\); sequential reaches **54.5%** at \(k{=}5\). At large \(k\), parallel wins because diversity keeps yielding new contrast; sequential can saturate after a decisive success/fail.

```text
mems := topk_similar_experiences(embed(query), bank, k=1)     # read
trajs := [rollout(query, mems) for _ in 1..k]   # MaTTS parallel
items := extract_self_contrast(query, trajs)    # ≤5 items; sequential path uses refine_check_loops
bank := bank ++ items                           # write (append-only; no decay in paper)
return BoN_rubric_pick(trajs)                   # eval for reported SR
```

### (c) Write / read / “decay” policies

- **Read:** `gemini-embedding-001` on the query; cosine match to **stored query vectors**; inject items from top-\(k\) *experiences* (default **\(k=1\)**) into system text; model must *debate* each item’s relevance. Ablation: 1 experience **49.7%** vs 4 → **44.4%** SR (Fig. 13).
- **Write:** judge (temp 0) → extract (temp 1) → **append**; same backbone as agent.
- **Decay:** **none** in-baseline (append-only, App. A.2). App. D lists decay/refresh as *future* layered stacks—today, implicit pressure is short items + low-\(k\) retrieval, not eviction policy.

### (d) Integration with the agent loop

Memory enters as **system-instruction augmentation**, not as hidden state in the tool API. The ReAct loop remains standard (BrowserGym for web, bash-only mini-SWE-Agent for code). The closed loop is: **retrieve → act → LLM-judge label → extract → append**—continuous test-time learning across the task stream \(q_1,\ldots,q_N\) with no peeking at future tasks.

## Empirical results

Primary metrics: **SR** (task success) and **AS** (average steps). WebArena overall across five domains and **684** instances; **MaTTS numbers** in the paper’s main table use **parallel MaTTS with \(k=5\)** in the `+MaTTS` row.

### WebArena (Table 1; 684 tasks, `+MaTTS` = parallel, \(k{=}5\), pass@1 selection)

| Model | No Mem | +Bank | +MaTTS | Steps: No → Bank → MaTTS |
| --- | --- | --- | --- | --- |
| Gemini-2.5-Flash | 40.5 | 48.8 | 51.8 | 9.7 → 8.3 → 7.9 |
| Gemini-2.5-Pro | 46.7 | 53.9 | 56.3 | 8.8 → 7.4 → 7.1 |
| Claude-3.7-Sonnet | 41.7 | 46.3 | 48.8 | 8.0 → 7.3 → 7.2 |

**Multi** subset (cross-site transfer, Pro): 6.9% → 13.8% → **20.7%** SR; AWM **3.4%**—workflow-style memory breaks when transfer is hardest.

### SWE-Bench-Verified (Table 2, 500 tasks; no AWM—bash too open-ended)

| Model | Resolve % (no → bank) | AS (no → bank) |
| --- | --- | --- |
| Gemini-2.5-Flash | 34.2 → **38.8** | 30.3 → **27.5** |
| Gemini-2.5-Pro | 54.0 → **57.4** | 21.1 → **19.8** |

### Mind2Web (Table 3) — Cross-Domain task SR example: Flash **1.0 → 1.6%**; Pro **1.4 → 1.7%** (hard benchmark; relative ordering still favors Bank).

### MaTTS / cost (Figs. 4–5, Table 5, Shopping, Flash)

Pass@1 **49.7 → 53.0** with bank+scaling; BoN@5 **39 → 55.1** (bank) vs **39 → 42.2** (no memory). MaTTS w/o memory ~39–42% (flat). Tokens: **~53,054** vs **50,847** baseline per task (+**4.3%**), vs **~59,373** for AWM with worse SR.

## Variants and ablations

- **Success-only vs +failure items (Figure 7, WebArena-Shopping, Gemini-2.5-Flash):** ReasoningBank **46.5% → 49.7%** when failures are included; Synapse and AWM barely move or *drop* when failed trajectories are added—validating the extraction objective, not just “store another trace.”
- **Number of retrieved experiences (Figure 13):** one experience **49.7%**; two–four experiences **46.0, 45.5, 44.4%**—*quality and relevance beat recall breadth* in this setup.
- **LLM-judge quality simulation (Figure 8):** success rates stay in a **tight band** for simulated judge accuracies 50%–100% (Shopping)—method is **not brittle** to moderate judge noise, with best SR at 100% as expected.
- **Gemma-3-12B (Table 6, Shopping):** 17.1% → **24.1%** SR; **11.8** steps vs 13.7+ baselines.
- **Table 4:** step cuts concentrate on **successful** trajectories (e.g. Shopping **6.8 → 4.7** steps; up to **2.1** fewer steps, ~**27%** relative vs no memory in reported cells).

## Failure modes and limitations

1. **No hierarchical or episodic stack**—the paper *focuses on content*; layered memory (working vs long-term) is listed as future work, not compared empirically.
2. **Append-only bank** can eventually grow large; *without* merge/decay, retrieval quality depends on embedding geometry and the cap implied by only surfacing *top* experience items—*not* a principled solution to unbounded growth.
3. **LLM-as-judge** can mislabel ambiguous tasks; the framework mitigates statistically but is not a verifier (Appendix E).
4. **SWE** excludes AWM; workflow-centric competitors are underrepresented in code domain numbers.
5. **Mind2Web** absolute SRs remain small—gains are statistically meaningful in relative terms but the benchmark is *hard*; do not over-interpret single-digit point wins as “solved navigation.”
6. **Over-retrieval** (too many experience bundles) and **over-long** item lists add conflict noise—contra naive “store everything in RAG.”

## When to use, when not

**Use** when: you have **streaming** tasks, can pay for **judge + extractor** calls, need **transferable strategies** across near-duplicate intents, and can benefit from **parallel rollouts** or **refinement** for critical queries. Fits web agents and ReAct-style SWE with bash. **Do not** assume this replaces **ground-truth evaluation** in training pipelines; it is a **test-time** construct. If your domain has **cheap, exact verifiers** (compilers, unit tests), you may prefer verifier-gated distillation *instead of* or *in addition to* the LLM judge. If memory *must* stay tiny on-device, the JSON bank + repeated embedding search may need engineering (none of that is the paper’s focus).

## Implications for harness engineering

- **[09-memory-files](09-memory-files.md):** industry “memory file” systems often conflate *logs* and *knowledge*; ReasoningBank is an explicit **schema** for the latter: title/description/content, generalization rules, and failure-aware extraction—what ad-hoc memory files are *groping toward*.
- **[14-reflexion](14-reflexion.md):** like Reflexion, ReasoningBank uses post-hoc language reflection; it **structurally differs** by banking **many compressed items per trajectory** with external retrieval, and by a **contrastive MaTTS** path that Reflexion’s single-trajectory verbal feedback does not cover.
- **[19-voyager-skill-libraries](19-voyager-skill-libraries.md):** Voyager’s skill library is **procedural**; ReasoningBank is closer to a **strategic** library—less about code subroutines, more about *reasoning heuristics* applicable across tasks.
- **[72-claude-mem-persistent-memory-compression](72-claude-mem-persistent-memory-compression.md):** production persistent-memory stacks compress and chunk conversation into stores; the paper’s evidence suggests **convergence** on *structured, self-judged distillates* with explicit success/fail branches—ReasoningBank is the *research-side articulation* of that convergence (cleaner ablations than product stacks allow).
- **[55-hermes-agent-self-improving](55-hermes-agent-self-improving.md):** Hermes-style self-improvement and ReasoningBank both pursue **in-loop adaptation**; here the **capability** comes from *memory content ↔ test-time compute synergy*, not from policy-gradient updates.

**Read-across:** ReasoningBank is the **principled schema**—typed items, self-judged outcomes, contrast-aware writes—that ad-hoc stacks (compressed timelines, multi-agent context handoffs) **converge toward** in product; the paper holds retrieval and consolidation still enough to *prove* the content design.

## Connections to other work in this corpus

- **[77-meta-tts-agentic-coding](77-meta-tts-agentic-coding.md):** MaTTS scales **trajectory diversity** to curate *memory*, whereas Meta TTS scales **summaries** for coding RTV/PDR—both reject raw-rollout comparison, different substrates.
- **Reflexion / Voyager (see § harness links):** shared emphasis on post-hoc text signals; ReasoningBank adds **retrieval + contrastive multi-rollout** extraction as first-class.
- **SWE-Exp** (cited in the paper) aligns with “coding experience”; ReasoningBank’s SWE numbers are **bash-only** mini-agent—contrast to tool-rich harnesses.

## Key takeaways

1. **Distill strategy, not traces**—with baselines held equal on retrieval and append-only consolidation, structured reasoning items from **success + failure** dominate trajectory/workflow memory on WebArena, Mind2Web, and mini-SWE.
2. **MaTTS** makes \(k>1\) useful *only* when the bank can **aggregate** contrast; vanilla multi-rollout TTS underuses shared structure.
3. **Synergy is measurable** via pass@1 and BoN curves: good memory *amplifies* scaling; weak memory *wastes* it.
4. **Efficiency** coexists: fewer steps, often lower tokens than other memory baselines, when counting judge+extraction.
5. **Retrieval** obeys a **less-is-more** rule in their ablation—governor logic for the harness should prefer **tight** recall over saturating K.

## References

- Ouyang, S. et al. (2025). *ReasoningBank: Scaling Agent Self-Evolving with Reasoning Memory.* arXiv:2509.25140. https://arxiv.org/abs/2509.25140 — official paper and benchmarks (WebArena, Mind2Web, SWE-Bench-Verified), Appendices A–E for prompts and limitations.
- Project repository: https://github.com/google-research/reasoning-bank
- Key cited memory baselines in the paper: **Synapse** (trajectory memory; Zheng et al., 2024) and **AWM** (workflow memory; Wang et al., 2025d) — read those originals for the precise baseline extraction recipes ReasoningBank displaces.
- **Self-contrast** (Chen et al., 2020) and **self-refinement** (Madaan et al., 2023) are the two conceptual pillars for the parallel and sequential MaTTS instantiations.

