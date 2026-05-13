# 92 — ChatDev: Communicative Agents for Software Development

**Paper.** Chen Qian, Wei Liu, Hongzhang Liu, Nuo Chen, Yufan Dang, Jiahao Li, Cheng Yang, Weize Chen, Yusheng Su, Xin Cong, Juyuan Xu, Dahai Li, Zhiyuan Liu, Maosong Sun — *ChatDev: Communicative Agents for Software Development* — arXiv:2307.07924v5 — Tsinghua University; The University of Sydney; BUPT; Modelbest Inc. — 2023 (revised 5 June 2024).

**One-line definition.** **Multi-agent virtual software company** organized as **waterfall SDLC**—**design, code, test**, plus **documentation-style deliverables** in the paper’s virtual-company framing—with **chat-based collaboration** along a **chat chain** (instructor/assistant dyads per subtask), **communicative dehallucination**, and **staged short-/long-term memory** to curb coding hallucinations versus one-shot or static-SOP agents.

## Why this paper matters (the role-driven multi-agent framework that predates and complements MetaGPT)

ChatDev is an early, fully specified **communicative** paradigm: isolated waterfall phases become one **language-mediated** chain of typed multi-turn dialogues. It **complements** MetaGPT by showing that **what** to communicate (chat chain) and **how** (communicative dehallucination) can lift **executability** and holistic **quality** on a shared benchmark. For harnesses, it is a portable template: **dyad per subtask**, **sequential** chain, explicit **termination**, **staged memory**, and an **anti-hallucination** dialogue pattern.

## Problem it solves

Prior work often improved **individual** SDLC phases with **different** deep-learning machinery, producing **technical inconsistency** across phases and a **fragmented** pipeline. Single-step or one-shot LLM coding also suffers **coding hallucinations**: incomplete, non-executable, or requirement-inconsistent code arising from vague instructions and unconstrained generation. ChatDev targets (1) **cross-phase integration** via a single communication substrate, (2) **task decomposition** without hand-built per-phase models, and (3) **reduced hallucination** through structured multi-turn interaction rather than only post-hoc filtering.

## Core idea in one paragraph

Instantiate multiple **software agents** with **social roles** (e.g., CEO, CTO, programmer, reviewer, tester). Organize work as a **chat chain** `C = ⟨P₁,…,P_{|C|}⟩` where each phase `Pᵢ` is a sequence of subtasks `Tⱼ`, and each subtask is a **bounded dialogue** `C(I,A) = ⟨I → A, A ; I⟩` looping until consensus and a typed **solution** `τ` is extracted (text or code). **Inception prompts** (`P_I`, `P_A`) lock role behavior at subtask start; **short-term memory** carries the current phase dialogue while **long-term memory** injects only **prior subtask solutions** into the next phase to stay within context limits. **Communicative dehallucination** changes the vanilla turn pattern so the assistant can **interrogate** the instructor for specifics before committing, shrinking bad code before it propagates.

## Mechanism (step by step)

### (a) Chat-chain architecture

Formally (paper Eq. 1): `C = ⟨P₁,…,P_{|C|}⟩`, `Pᵢ = ⟨T₁,…,T_{|Pᵢ|}⟩`, and `Tⱼ = τ(C(I,A))` with `C(I,A) = ⟨I → A, A ; I⟩` looping. The chain is **sequential**: each subtask emits a solution `τ` that becomes the **bridge** into the next subtask or phase. The design uses **two agents per subtask**—an **instructor** `I` and **assistant** `A`—avoiding complex multi-agent broadcast topologies and simplifying consensus. Visually (Figure 2), a task like “develop a Gomoku game” flows CEO/CTO-led **design**, then CTO/programmer **coding** and **completion**, then reviewer/programmer **review**, then tester/programmer **testing**.

Pseudocode sketch:

```python
def chatdev(task, phases):
    long_term = []
    for phase in phases:
        short_term = []
        for subtask in phase.subtasks:
            I, A = instantiate_roles(P_I[subtask], P_A[subtask], long_term)
            # Inner loop: instructor leads, assistant responds; optional CDH branch
            while not done(subtask, short_term):
                instr = I.next(short_term)
                reply = A.respond(instr, short_term, communicative_dehallucination=True)
                short_term.append((instr, reply))
            tau = extract_solution(short_term)  # text or <SOLUTION> code
            long_term = propagate(long_term, tau)
    return long_term  # final codebase + metadata
```

### (b) SDLC phases (design, code, test, doc)

The paper’s **operational** decomposition is **three core phases** with **five subtasks** (see §4 implementation details): **design**; **coding** (including **code writing** and **code completion**); **testing** split into **code review (static)** and **system testing (dynamic)**. That maps cleanly to a **waterfall** reading: requirements/design discourse → implementation → V&V. **Documentation** is not a separate named phase in the same way, but the **framing and figures** present **Codes** and **Docs** as first-class **artifacts** of the “virtual company,” and natural-language design phases produce much of the narrative specification from which code is derived—so in harness terms you should still budget **doc/readme** as an explicit **downstream or parallel** subtask if your product needs user-facing text.

### (c) Inception prompts and role specialization

At **subtask start only**, the framework applies **inception prompting**: symmetric(ish) **system prompts** `P_I` and `P_A` that encode subtask **overview, objectives, specialized roles, tools, communication protocol, stop conditions, and anti-spam/anti-fake constraints**. Agents are `I = ρ(LLM, P_I)`, `A = ρ(LLM, P_A)` (role customization via system messages; paper Eq. 2). **Role text is not cosmetic**: the ablation removing roles (**⧹Roles**) collapses **executability** to **0.58** and **quality** to **0.2212** (Table 4)—e.g., “prefer GUI” programmers stop producing CLI-only defaults; “careful reviewer” roles yield concrete defect findings instead of vague praise.

### (d) Thought instructions for agent communication

The instructor is tasked with **steering** multi-turn flow toward subtask completion; the assistant must **comply** without **role-flipping**, **repeated non-progress instructions**, or **facile acknowledgments** that do not change artifacts. The **instructor uses prior short-term memory** to generate the next instruction; the **assistant** conditions on memory plus latest instruction. Turn updates follow paper Eqs. 3–4: short-term `Mᵢᵗ` accumulates `(I,A)` pairs until a **max dialogue budget** or **consensus/termination** (implementation: subtask ends after **10** communication rounds or **two successive rounds with unchanged code** during code-heavy subtasks, with **communicative dehallucination** active in code completion, review, and testing).

### (e) Communicative dehallucination

**Vanilla** alternation: `⟨I → A, A ; I⟩` (paper Eq. 6). **Communicative dehallucination** inserts a **deliberate micro role-reversal** so the assistant first **asks the instructor** for **missing concrete detail** (dependency names, class surface, invariants) before a final code answer—pattern in paper Eq. 7. Empirically, **⧹CDH** drops **completeness** from **0.56 → 0.47**, **quality** from **0.3953 → 0.3094** (Table 4). The mechanism targets **vague** instructions that would otherwise elicit **placeholder** code and **unfinished** methods (common reviewer flag: “Method Not Implemented” at **34.85%** of review topics in one analysis, Figure 4).

### (f) Memory stream

**Short-term memory** (Eq. 3) stores all `(Iₖ, Aₖ)` in the **current** phase to support coherent dialogue. **Long-term** carries **only solutions** from prior subtasks (Eq. 5): at phase `i+1`, the first instructor prompt **merges** the injected memory `M̃` with the phase entry prompt—**not** the full prior logs—mitigating context overflow and focusing attention. This is directly relevant to **agent harnesses** that must interleave long chat trees with **deterministic** tool traces.

## Empirical results (tables — execution success rate, comprehensiveness, executability; cost; comparison vs GPT-Engineer, MetaGPT)

**Table 1 (holistic software metrics, SRDD / 1,200 prompts, shared hyperparameters: ChatGPT-3.5, temperature **0.2**, Python **3.11.4**).** ChatDev achieves **completeness** **0.5600** (no placeholder snippets), **executability** **0.8800** (compiles and runs), **consistency** (requirement↔code embedding cosine) **0.8021**, and **quality** (product of the three) **0.3953**. **GPT-Engineer** scores **0.5022 / 0.3583 / 0.7887 / 0.1419**; **MetaGPT** **0.4834 / 0.4145 / 0.7601 / 0.1523**. Differences vs ChatDev are marked **†** as statistically significant at **p ≤ 0.05** where shown.

**Pairwise preferences (Table 2).** Versus **GPT-Engineer**, ChatDev wins **77.08%** (GPT-4 judge) and **90.16%** (human). Versus **MetaGPT**, **57.08%** / **88.00%** with **5.42%** / **4.08%** ties—so ChatDev is preferred, but the GPT-4 margin over MetaGPT is **not** a blowout, highlighting **evaluation sensitivity**.

**Cost and runtime (Table 3; no $ figures in the paper).** **ChatDev**: **148.21 s**, **22,949** tokens, **4.39** files, **144.3** lines. **GPT-Engineer**: **15.60 s**, **7,183** tokens. **MetaGPT**: **154.00 s**, **29,279** tokens. ChatDev is **~9.5×** slower than GPT-Engineer, **~0.96×** MetaGPT in time; **~22% fewer** tokens than MetaGPT, **~3.2×** more than GPT-Engineer. Billable **USD** is whatever your provider charges × those token counts; wall time **~2.5 min** mean for ChatDev in this table.

**SRDD** supplies **1,200** prompts (5 areas × 40 subcategories × 30 tasks; Ubuntu and major app stores, §4).

## Variants and ablations

**Truncated chains (Table 4).** **≤Coding** quality **0.2512**; **≤Complete** **0.3690**; **≤Review** **0.3717**; **≤Testing** matches full on **completeness/executability** (**0.56 / 0.88**)—so some aggregate heads are unchanged without test, but **completion+review** still matter for **completeness/quality** overall. **⧹CDH** and **⧹Roles** hurt most (**roles** drives executability **0.58**). **Figure 3:** **57.20%** NL utterances (mostly **design**); later work concentrates in **code review**.

## Failure modes and limitations

§6: **autonomy** is easy to overread—**shallow** logic without dense requirements. Metrics are **not** full functional, **security**, or **UX** tests. **Token/time** vs single-agent increases footprint. Runtime-error traces: **ModuleNotFound**-class **45.76%**, **NameError/ImportError** **15.25%** each (Figure 5 analysis)—**imports/wiring** still fail often.

## When to use, when not

**Use** when you need a **reproducible, inspectable** multi-agent **dialogue log** for **small/medium** Python utilities or prototypes; when **decomposing** a vague brief into design → code → test is more reliable than a **single-shot** repo synthesis; when you can pay **higher token and latency** than a single agent; when you can provide **instructor-grade** system prompts and role cards. **Avoid** (or wrap heavily) when you need **provable** correctness, **subtle security**, large-scale refactors, non-Python stacks without retooling, or **tight latency/cost** envelopes where **~23k+ tokens** per app is uneconomical.

## Implications for harness engineering

**ChatDev** = **chat-driven, waterfall, dialogue-first**; **MetaGPT** = **SOP/artifact-driven** ([20-metagpt-role-based-workflows](20-metagpt-role-based-workflows.md), [91-metagpt-deep](91-metagpt-deep.md))—*conversation graph* vs *artifact graph* templates for modern harnesses. Both need **delegation and boundaries** ([02-subagent-delegation](02-subagent-delegation.md)). **Dyads** can limit broadcast echo but still **collapse** onto bad shared plans; see [98-diversity-collapse-mas](98-diversity-collapse-mas.md). **Hybrid** pattern: **MetaGPT**-style staged artifacts + **ChatDev**-style **CDH** debug loops.

## Connections to other work in this corpus

The paper directly compares to **GPT-Engineer** (single-agent) and **MetaGPT** (multi-agent with **standard operating procedures**). It cites **CAMEL** for communicative “inception” ideas, **CodeGen**-era code models for programming-language subtasks, and **AutoGPT**-style **agent** momentum. It aligns with broader themes: **waterfall** segmentation with **unified** LLM backbones, **self-review** in review phases, and **test-time** interaction scaling via multi-turn **dialogue** rather than **single** completions.

## Key takeaways

- **Chat chain** = explicit **SDLC** serialization **plus** per-subtask **instructor/assistant** consensus loop; solutions `τ` bridge phases.
- **Inception prompts** + **roles** are **load-bearing**; removing roles devastates **executability** and **quality** (Table 4).
- **Communicative dehallucination** is a **pattern-level** fix (assistant asks; instructor specifies) and measurably raises **completeness/quality** vs ablated CDH.
- **Memory** is **staged**: full dialogue in-phase; **cross-phase** = **solutions only** to **fit context** and reduce noise.
- On SRDD, ChatDev leads **Table 1** (notably **executability 0.88**), wins **Table 2** head-to-heads, and sits between GPT-Engineer and MetaGPT on **time/tokens**—**higher** than single-agent, **slightly** leaner on tokens than MetaGPT in **Table 3**.

## References

- Qian, C. et al. *ChatDev: Communicative Agents for Software Development.* arXiv:2307.07924v5. [https://arxiv.org/abs/2307.07924](https://arxiv.org/abs/2307.07924)  
- Code and data: [https://github.com/OpenBMB/ChatDev](https://github.com/OpenBMB/ChatDev)  
- Baselines cited: Osika, *GPT-Engineer*; Hong et al., *MetaGPT* (ICLR 2023).  
- Communicative / inception precedent: Li et al., *CAMEL* (NeurIPS 2023).
