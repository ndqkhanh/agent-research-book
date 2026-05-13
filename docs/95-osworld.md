# 95 — OSWorld: Benchmarking Computer-Use Agents in Real OS Environments

**Paper.** Tianbao Xie, Danyang Zhang, Jixuan Chen, Xiaochuan Li, Siheng Zhao, Ruisheng Cao, Toh Jing Hua, Zhoujun Cheng, Dongchan Shin, Fangyu Lei, Yitao Liu, Yiheng Xu, Shuyan Zhou, Silvio Savarese, Caiming Xiong, Victor Zhong, Tao Yu — *OSWorld: Benchmarking Multimodal Agents for Open-Ended Tasks in Real Computer Environments* — NeurIPS 2024 — arXiv:2404.07972 — 2024.

**One-line definition.** OSWorld is a 369-task (Ubuntu) benchmark with VM-backed real application execution and **execution-based** scoring; the paper reports a **~12.2%** ceiling for the strongest published baselines versus **~72.4%** human success on the same script-evaluated tasks—spanning real Office, web, code, file, and **multi-app** workflows across Ubuntu, Windows (supplementary 43), and a macOS-capable environment stack.

## Why this paper matters (the canonical benchmark for computer-use agents; massive headroom 12% vs 72%)

Static demos, single-domain simulators, and **non-executable** web sets mis-score alternative correct plans and under-measure **cross-app** reality. OSWorld became the default referent for **end-to-end OS GUI/CLI** control: reproducible inits, **134** evaluator function families, and humans at **72.36%** overall—not saturated, so models have room to grow, yet still **~6×** above the strongest baselines (**12.24%** GPT-4 on a11y; **12.17%** GPT-4V with screenshot+a11y). That gap encodes the research agenda: grounding, software literacy, and long-horizon control; it also recalibrates product claims—code- or web-bench strength does not transfer here.

## Problem it solves (agentic OS automation evaluation gap)

Prior work either (i) **lacked executable environments** and scored single predicted actions against fixed gold sequences, (ii) **narrowed the action space** to website-specific clicks/typing, or (iii) covered **one application domain** (coding terminals, a browser silo, mobile). None captured the real failure modes of “use my computer as I do”: **intermediate** initial states (spreadsheets already open, email half drafted), **multi-app** workflows, **infeasible** requests, and the need to reward **diverse** correct finals (edited files, cookies, a11y structure). OSWorld reframes the problem as a POMDP with explicit execution-based reward and a complete mouse/keyboard program API.

## Core idea in one paragraph

**OSWorld** is an environment (VM coordinator + simulators + task-manager pipeline) and a **curated task suite** where each item bundles a natural-language instruction, a JSON **setup** block (download assets, open windows, size UI), a **post-config** for stabilizing the post-agent state, **getter** hooks that pull VM files / browser state / a11y trees / cloud “gold” references, and a **bespoke evaluator** (often composing library functions) that returns 0/1 (or partial credit) only after real execution. Agents observe **multimodal** state (at minimum full-screen **screenshots** at 1920×1080; optionally **simplified a11y trees**; optionally **Set-of-Mark** overlays) and emit **Pythonic pyautogui** programs plus **WAIT / DONE / FAIL**—then the harness rolls forward until a cap (15 steps in the main experiments) or terminal action.

## Mechanism (step by step)

(a) **Task corpus — 369 Ubuntu tasks (43 Windows adaptations).** **369** tasks: **30** infeasible (**8.1%**), **84** **integrated** (Mind2Web, GAIA, NL2Bash, others), **101** **multi-app** vs **268** **single-app**, **302** **initial states**. App clusters: **OS**, **LibreOffice**, **Chrome/VLC/Thunderbird**, **VS Code/GIMP**, **multi-app** workflow (e.g. **27.4%** of instructions by bucket in Fig. 3).

(b) **VM-based execution (Ubuntu, Windows, macOS-class stack).** A **host Coordinator** materializes a VM from **snapshots**, runs scripted setup, streams observations from the **Simulator**, and after the agent halts, executes **postconfig** to activate windows, export artifacts, or stage evaluation inputs. The design is **parallelizable** (multiple VMs per host) and can run **headless**. **Windows (43 tasks)** reuses 43 **adapted** Ubuntu cases for **cross-OS** analysis; macOS is supported in the **environment** description, with the bulk of the released benchmark on **Ubuntu** for open-source app APIs.

(c) **Execution-based scoring (not next-action matching).** **134** unique **evaluator** families compare **retrieved** state—cookies, spreadsheet cells with fuzzy rules, **check_a11y_tree** constraints on Thunderbird chrome, file hashes against cloud “expected” workbooks, etc.—sometimes after **scaffolding** (e.g. Chrome remote debugging, VS Code extension hooks). Some tasks add **dynamic** getters (crawlers) for live web facts at evaluation time. This is the paper’s key methodological advance: the reward is a **program** over post-state, not a unique action string.

(d) **Multimodal observation space.** Full **desktop screenshots** preserve clutter, z-order, pointer shape, and **hidden-in-plain-sight** UI noise. **A11y trees** (AT-SPI on Ubuntu, PyWinAuto-class hooks on Windows) are **filtered** to a compact TSV of tag/name/text/pos/size; raw XML can reach **“millions of tokens”** in pathological UIs, so the benchmark explicitly studies **simplified** trees. Custom streams (terminal scrollback) are possible.

(e) **Unified action space (pyautogui + control primitives).** Actions are **syntactically valid Python** calling **pyautogui** (`moveTo`, `click`, `write`, `hotkey`, `scroll`, `dragTo`, `keyDown`/`keyUp`, etc.), optionally wrapped in loops, plus **`WAIT`**, **`DONE`**, **`FAIL`**. The paper contrasts this to web-only action sets that **omit** right-click-with-modifier patterns and other **general HCI** affordances, which caps expressivity of older benchmarks.

**Agent loop (experiments).** **Chat-style** context with the **last 3** `(observation, action)` turns (few-shot (obs,act) was **2.79%** and abandoned). **T=1.0**, **top_p=0.9**, max **15** steps, **1920×1080** by default, truncation from the **start** of context if over token limits.

## Empirical results (table — GPT-4V, Claude, etc. with success rates; vs 72% human baseline)

**Human baseline (annotators, CS majors, *not* pre-exposed to each app):** **72.36%** overall, with per-category **75.00% (OS)**, **71.79% (Office)**, **70.51% (Daily)**, **73.47% (Professional)**, **73.27% (Workflow)**—tight **~70–75%** band vs models’ spiky, category-dependent curves. **Median** human time **111.9 s** (vs **~35 s** on a WebArena 100-example sample in their plot), with a long **900s+** tail, underscoring difficulty.

| Setting / model | OS | Office | Daily | Prof. | Workflow | **Overall** |
|-----------------|----|--------|-------|------|-------------|--------|
| **Human** | 75.00% | 71.79% | 70.51% | 73.47% | 73.27% | **72.36%** |
| A11y — **GPT-4** | 20.83% | 3.58% | 25.64% | 26.53% | 2.97% | **12.24%** (best) |
| A11y — Qwen-Max | 29.17% | 3.58% | 8.36% | 10.20% | 2.61% | 6.87% |
| A11y — Gemini-Pro-1.5 | 12.50% | 2.56% | 7.83% | 4.08% | 3.60% | 4.81% |
| Screenshot — **GPT-4V** | 12.50% | 1.86% | 7.58% | 4.08% | 6.04% | 5.26% |
| Screenshot — **GPT-4o** | 8.33% | 3.58% | 6.07% | 4.08% | 5.58% | 5.03% |
| Screenshot — Gemini-Pro-1.5 | 12.50% | 6.99% | 2.71% | 6.12% | 3.60% | 5.40% |
| Screenshot + A11y — **GPT-4V** | 16.66% | 6.99% | 24.50% | 18.37% | 4.64% | **12.17%** |
| Screenshot + A11y — **GPT-4o** | 41.67% | 6.16% | 12.33% | 14.29% | 7.46% | 11.21% |
| Set-of-Mark — **GPT-4V** | 8.33% | 8.55% | 22.84% | 14.28% | 6.57% | 11.77% |
| Set-of-Mark — **Claude-3-Opus** | 12.50% | 2.72% | 14.24% | 6.12% | 4.49% | 6.72% |
| Set-of-Mark — **CogAgent** (open) | 4.17% | 0.00% | 2.71% | 0.00% | 0.53% | **0.99%** (floor) |

**Readout:** best overall **A11y GPT-4 12.24%** vs **GPT-4V+a11y 12.17%**; **screenshot-only** peak **5.80%** (Gemini-Pro-vision) / **5.26% (GPT-4V)**; **workflow** column tops out **6.57% (GPT-4V SoM)**. **Claude-3-Opus** lags on OSWorld (**2.42%** screenshot, **4.41%** +a11y) vs chat leaderboards.

## Variants and ablations

- **Input modality:** a11y-only vs **screenshot** vs **a11y+screenshot** vs **Set-of-Mark**; SoM can **hurt GPT-4V** vs a11y+shot (11.77% vs 12.17%) because dense OS UIs and **sub-element** control need **sub-box** pixel precision, not just indexed regions.
- **Resolution sweep (Fig. 5, 10% subset):** for **screenshot-only**, higher **native** input ratio monotonically helps; for **SoM**, **downsampling to 0.4** can **improve** vs full-res—suggesting **noise/scale** tradeoffs.
- **History length (Fig. 7, 10% subset, GPT-4V SoM):** more **a11y text history** helps; more **image history** does not—**VLMs still weak** at visual trajectory reasoning.
- **Perturbation study (28 “easy” tasks, 50.79% SoM success):** moving windows, min-sizing targets, or **clutter** drops success to **~15–25%** (and **>60% relative** loss)—**fragile** policies.
- **Task subsets (Table 6, GPT-4V SoM):** success vs human-time buckets **16.78% (Easy) / 13.12% (Medium) / 4.59% (Hard)**; **multi-app 6.57%** vs **13.74%** single-app; **infeasible** tasks show agents can get **lucky FAIL** behavior (**16.67%**).

## Failure modes and limitations (of the benchmark itself + of current agents)

**Benchmark / harness limits:** despite **4 rounds of QA** and **>400h** of checking, the authors note residual **false pos/neg** risk and that **heavier red-teaming** could still tighten evaluators. **Infeasible** tasks (30) are a stress test but complicate **aggregate** scoring semantics. The **a11y** channel is **not** universally faithful (custom UIs, canvas apps, game-like surfaces). **Windows** tasks need **separate** licensing activation. Human scores (~72%) are **not** “expert” power users, biasing *up* the gap vs specialized operators.

**Agent failures (§5.4, experiments):** coordinate drift, **repetitive** actions, weak **GUI commonsense** and app-specific affordances, **popups** and **z-order**, **multi-app** handoff, a11y **token** load, premature **FAIL**, fragility to **window** moves/resize/clutter. **Vision-only** is deployment-realistic but peaked **~5.3%** in this sweep—**deployability vs score** tradeoff.

## When to use, when not

**Use OSWorld** when you need: (i) a **standardized** “full OS” number comparable across papers, (ii) to stress **real app** I/O, **infeasible** detection, and **multi-app** plans, (iii) to justify research into **a11y-assisted** vs **vision-only** scaffolds, or (iv) to calibrate product claims on **end-user** software, not a stubbed website.

**Do not** treat a **12%** score as “ready for production,” as a substitute for **domain** harnesses, or as measuring **safety** (it scores task success, not exfiltration or policy compliance). Complement with targeted web-only or IDE-only tracks when those dominate your deployment.

## Implications for harness engineering

**OSWorld is the canonical OS-level benchmark:** the *computer-use agent* line of work now largely reports against it (success rate, modality, and workflow columns) alongside narrower suites. In this corpus, pair it with:

- [34-clawbench-live-web-tasks](34-clawbench-live-web-tasks.md) for **browser-centric / live** task harness patterns where OSWorld’s desktop breadth is not the SLO;
- [38-claw-eval](38-claw-eval.md) for **eval harness design and iteration** (metrics, run orchestration) that should wrap OSWorld **VM pools** the same way as any long-horizon tool loop;
- [26-linuxarena-production-agent-safety](26-linuxarena-production-agent-safety.md) when porting any OSWorld-style **shell + filesystem + package** power to **production safety** invariants (OSWorld is not a red-team suite);
- [96-gdpval](96-gdpval.md) for **economic / knowledge-work** valence—GDPVal-style “did the deliverable help?” complements OSWorld’s **mechanical** success bit.

**Harness takeaway:** use OSWorld as the **broad** real-OS axis; add **narrow** domain harnesses (IDE, CRM) for regression—same **pyautogui layer + per-task eval** pattern.

## Connections to other work in this corpus

**GUI / CUA** work inherits OSWorld’s *screenshot ± a11y ± SoM* design; [38-claw-eval](38-claw-eval.md) captures **repro** patterns for any VM-backed suite; [26-linuxarena-production-agent-safety](26-linuxarena-production-agent-safety.md) for **safety** when OS control leaves sandboxes; [34-clawbench-live-web-tasks](34-clawbench-live-web-tasks.md) and [96-gdpval](96-gdpval.md) for **web** and **knowledge-work value** when headline OSWorld scores stay low.

## Key takeaways

1. **OSWorld = 369 real Ubuntu computer tasks, 43 Windows extras, 134 unique eval programs, 302 init states, 30 infeasible**—it measures **outcomes**, not action-match.
2. **Humans: ~72% overall** with small cross-category spread; **best models: ~12.2%** (GPT-4 a11y or GPT-4V+a11y), **screenshot-only ~5.3%**, **multi-app workflows ~6.6%** at best.
3. **A11y** can double scores vs raw screenshot but is **not** a free win (tokens, sparsity, SoM can backfire for pixel tasks).
4. The paper’s ablations (resolution, **textual** history, **window** noise) are directly actionable for **harness** designers building evaluators, observation compressors, and **reset** policies.

## References

Tianbao Xie, Danyang Zhang, Jixuan Chen, Xiaochuan Li, Siheng Zhao, Ruisheng Cao, Toh Jing Hua, Zhoujun Cheng, Dongchan Shin, Fangyu Lei, Yitao Liu, Yiheng Xu, Shuyan Zhou, Silvio Savarese, Caiming Xiong, Victor Zhong, and Tao Yu. *OSWorld: Benchmarking Multimodal Agents for Open-Ended Tasks in Real Computer Environments.* In *Advances in Neural Information Processing Systems (NeurIPS)*, 2024. arXiv:2404.07972. Project: https://os-world.github.io

Positioning: WebArena, VisualWebArena, MiniWoB++, Mind2Web, GAIA, hybrid code/web sets; **pyautogui** + **AT-SPI**.
