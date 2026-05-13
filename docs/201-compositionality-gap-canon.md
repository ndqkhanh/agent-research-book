# 201 — The Compositionality-Gap Canon: Why Latent Multi-Hop Reasoning Is Fragile

> **Disambiguation.** This file is the **formal-theoretical** half of a four-part block (198–201) on multi-hop reasoning. The dataset canon is in [198-multi-hop-qa-datasets-canon](198-multi-hop-qa-datasets-canon.md), the technique arc in [199-multi-hop-reasoning-techniques-arc](199-multi-hop-reasoning-techniques-arc.md), graph-grounded retrieval in [200-graph-grounded-multi-hop-retrieval](200-graph-grounded-multi-hop-retrieval.md). This file connects to the architectural lineage in [32-recurrent-depth-implicit-reasoning](32-recurrent-depth-implicit-reasoning.md) and [110-transformer-llm-architecture-for-agent-builders](110-transformer-llm-architecture-for-agent-builders.md).

## One-line definition

The **compositionality gap** is the empirically measured fraction of cases where a language model knows both sub-facts of a two-hop question but **fails to compose them latently** (i.e. without explicit chain-of-thought) — and the 2022–2024 canon (Press → Wang → Yang → Biran → Balesni) jointly establishes that this gap is real, layer-allocation-bound, scaling-resistant, and the formal hook beneath every prompted / declarative / RL-trained multi-hop technique catalogued in [199-multi-hop-reasoning-techniques-arc](199-multi-hop-reasoning-techniques-arc.md).

## Why this canon matters

Most multi-hop-reasoning papers are *empirical* — they propose a technique and benchmark it. The five papers in this chapter are *theoretical* — they ask **why** multi-hop is hard for transformers, **where** in the architecture it breaks, and **whether** scaling alone fixes it. The answers are surprisingly clean, and they justify the entire prompted/agentic line: explicit chain-of-thought isn't a "prompting trick," it's a structural workaround for an architectural limitation.

Press et al. 2022 named the **compositionality gap** and showed it doesn't shrink with scale on GPT-3 sizes — single-hop accuracy improves, but the *fraction* of "knew the parts, missed the whole" stays constant. Yang et al. 2024 (DeepMind) probed inside the model and found **strong evidence for first-hop latent traversal but weak evidence for second-hop latent traversal**, with no scaling on the second hop. Biran et al. 2024 ("Hopping Too Late") localised the failure: **the bridge entity resolves in early layers; the second hop has to commence in later layers; the architecture sometimes runs out of depth**, and *back-patching* the later-layer hidden state to an earlier layer fixes 66% of failures. Balesni et al. 2024 (Apollo / UK AISI) showed via synthetic-fact controls that latent two-hop **fails when both facts are synthetic** and **succeeds when one is synthetic and one natural** — the bridge entity needs representational alignment that purely-synthetic data doesn't provide. Wang et al. 2024 ("Grokked Transformers") added an optimistic counterpoint: transformers *can* learn implicit multi-hop, but only via grokking — extended training far past overfitting — and even then they generalise OOD on comparison but **fail systematically on composition**.

Take this canon seriously and three things change. (1) You stop treating CoT as optional — it materialises the bridge entity as a token, side-stepping the layer-allocation bottleneck. (2) You stop expecting scale to fix multi-hop — the second-hop curse doesn't scale away on standard pretraining schedules; explicit reasoning steps or grokking-regime training are the only known fixes. (3) You start budgeting *depth* in your harness — long-horizon multi-hop tasks need *either* very deep models *or* explicit token-emitted intermediate steps; conflating "more parameters" with "more reasoning depth" is the trap.

## Problem this canon characterises

- Why does GPT-4 reliably state "*the spouse of the director of Casablanca is X*" if asked, yet sometimes fail when asked the question and required to compose silently?
- Where in the transformer does the second hop happen — and can it always happen at all?
- Does scaling the model fix latent multi-hop, or does the gap persist?
- Why does CoT — which doesn't change parameters — close the gap so consistently?
- What can be proven about multi-hop *capacity* via mechanistic interpretability and grokking experiments?

## §1 — The compositionality gap (Press et al. 2022)

### Citation and setup

- *Measuring and Narrowing the Compositionality Gap in Language Models*. Ofir Press, Muru Zhang, Sewon Min, Ludwig Schmidt, Noah A. Smith, Mike Lewis. arXiv 2210.03350; Findings of EMNLP 2023.
- Data: 2WikiMultiHopQA-style two-hop questions where the model is shown to know both sub-facts when asked separately.

### Mechanism

For each two-hop question, ask the LM:

1. The question directly (no CoT). Did it answer correctly?
2. The first-hop sub-question. Did it answer correctly?
3. The second-hop sub-question (with the bridge entity provided). Did it answer correctly?

The **compositionality gap** is the fraction:

`gap = P(answer_2hop_wrong | answer_hop1_correct ∧ answer_hop2_correct)`

i.e. *given* the model knew both pieces, how often did it fail to compose?

### Headline empirical result

Across GPT-3 sizes (350M → 175B), **single-hop accuracy improves with scale, but the compositionality gap stays roughly constant** at ~40–50%. **Pure scaling does not fix composition.** Self-Ask (the prompting fix introduced in the same paper) narrows the gap by externalising the bridge entity as an explicit follow-up question.

### Why it matters

This is the most-cited *negative* result for multi-hop scaling. Every prompted / declarative / RL-trained multi-hop technique is, in part, a response to the question "if scale doesn't fix this, what does?"

## §2 — Latent multi-hop reasoning (Yang et al. 2024, DeepMind)

### Citation and setup

- *Do Large Language Models Latently Perform Multi-Hop Reasoning?* Sohee Yang, Elena Gribovskaya, Nora Kassner, Mor Geva, Sebastian Riedel. ACL 2024, arXiv 2402.16837.

### Mechanism — probing for latent traversal

For each two-hop query of the form *"the [hop2-relation] of the [hop1-relation] of [entity-A]"*:

1. Run the model and extract internal representations.
2. Probe whether the bridge entity (hop-1 answer) is encoded in the residual stream at any layer.
3. Probe whether the final answer is computed *from* that bridge representation (vs. from raw input or surface co-occurrence).

### Headline numbers

- **>80% latent first-hop usage** for some relation types (the model *does* compute the bridge entity internally).
- **Moderate** evidence for second-hop / full traversal — much weaker than first-hop signal.
- **Clear scaling for first-hop** (bigger models compute bridge more reliably) but **no scaling for the second hop** — the second hop is the binding constraint.

### Why it matters

Pinpoints the **second-hop curse**: the bridge resolves fine, but chaining from it is the failure mode. The first-hop probing methodology is the de-facto standard for any subsequent work claiming or measuring latent reasoning.

- **Repo.** [github.com/google-deepmind/latent-multi-hop-reasoning](https://github.com/google-deepmind/latent-multi-hop-reasoning) — 91★.

## §3 — Hopping too late (Biran et al. 2024)

### Citation and setup

- *Hopping Too Late: Exploring the Limitations of LLMs on Multi-Hop Queries*. Eden Biran, Daniela Gottesman, Sohee Yang, Mor Geva, Amir Globerson. EMNLP 2024, arXiv 2406.12775.

### Mechanism — layer-allocation analysis

Use logit-lens and patchscope-style probes to track *when* in the layer stack the bridge entity becomes encoded, and *when* the second hop begins.

### Key finding

- **Bridge entity resolves in early-to-middle layers** (layer 10–20 of a 32-layer model, typical).
- **Second hop only commences in later layers** (layer 22+), but the *necessary knowledge* for the second hop may already be gone from those layers — it lived in earlier-layer attention patterns.
- The architecture **runs out of depth** to chain — the second hop "hops too late."

### Back-patching — the diagnostic intervention

Patch a *later-layer hidden state back to an earlier layer* (i.e. give the early layer the to-be-computed bridge entity as if it had already been computed):

- **66% of previously-incorrect cases become correct.**

This is a striking causal demonstration that the limitation is **layer allocation**, not knowledge.

### Why it matters

The two-hop curse is reframed from a knowledge problem to a **depth-allocation problem**. CoT works precisely because it materialises the bridge entity as a token, restoring the full depth budget for the second hop. This justifies the "always emit the chain" stance of [13-react](13-react.md), [16-plan-and-solve](16-plan-and-solve.md), [18-chain-of-verification-self-refine](18-chain-of-verification-self-refine.md).

## §4 — Two-hop latent reasoning under controls (Balesni et al. 2024, Apollo / UK AISI)

### Citation and setup

- *Lessons from Studying Two-Hop Latent Reasoning*. arXiv 2411.16353. Apollo Research × UK AI Safety Institute.

### Mechanism — synthetic-fact controls

Construct controlled two-hop facts where:

- **Both facts synthetic** ("Alice's mentor is Bob; Bob lives in Z" — neither in the model's training).
- **One synthetic, one natural** ("Alice's mentor is Bob; Bob lives in Paris" — first synthetic, second natural).
- **Both natural** (the standard 2WikiMHQA-style setup).

Train the model to know each fact in isolation; test whether it composes silently.

### Findings

- LLMs **fail** to latently compose two **synthetic** facts.
- LLMs **succeed** when one fact is synthetic and one is natural.
- Mechanistic explanation: the bridge entity must serve **both as output (hop 1) and input (hop 2)** without CoT. Synthetic-only entities lack the representational alignment that the natural-data lookup paths provide; the bridge can be *retrieved* but not *re-used* in the same forward pass.

### Why it matters

Latent two-hop is **real but fragile and depends on training-data overlap**. Explicit CoT is robust precisely because it materialises the bridge entity as a *token* that the next forward pass can consume normally — bypassing the representational-mismatch failure.

- **Repo.** [github.com/mbalesni/synthetic-two-hop](https://github.com/mbalesni/synthetic-two-hop).

## §5 — Grokked transformers as implicit reasoners (Wang et al. 2024)

### Citation and setup

- *Grokked Transformers are Implicit Reasoners: A Mechanistic Journey to the Edge of Generalization*. Boshi Wang, Xiang Yue, Yu Su, Huan Sun. NeurIPS 2024, arXiv 2405.15071. (OSU NLP.)

### Mechanism — extreme training regimes

Train transformers on synthetic composition and comparison reasoning tasks **far past overfitting** — until *grokking* occurs (validation accuracy snaps from low to high after the train loss has long flatlined).

### Findings

- Transformers **can** learn implicit composition and comparison reasoning — but **only via grokking**.
- On out-of-distribution test sets:
  - **Comparison** generalises (the grokked circuit transfers).
  - **Composition** fails systematically (the circuit is regime-bound).
- Mechanistically, a **generalising circuit emerges and out-competes the memorising circuit** at the grokking transition.
- A fully grokked transformer beats GPT-4-Turbo / Gemini-1.5-Pro on hard reasoning tasks with large search spaces.

### Why it matters

The optimistic counterpoint to Press / Yang / Biran / Balesni: **multi-hop implicit reasoning is achievable**, but requires regimes far beyond standard pretraining schedules. This explains why standard-scale pretraining alone doesn't close the gap — and motivates the recent line of synthetic-fact-heavy SFT regimes for reasoning (DeepSeek-R1's distillation, the math-heavy continued-pretraining of MathPile-class corpora).

- **Repo.** [github.com/OSU-NLP-Group/GrokkedTransformer](https://github.com/OSU-NLP-Group/GrokkedTransformer) — 238★.

## §6 — In-context two-hop reasoning (2025 follow-ups)

- **arXiv 2502.13913** — *How Do LLMs Perform Two-Hop Reasoning in Context?* Mechanistic study of *in-context* two-hop (vs. parametric) — same layer-allocation patterns hold, with the bridge entity living in attention patterns rather than residual stream.
- The in-context regime is *better* than the parametric regime for hop 2 (the explicit context provides the alignment that purely-parametric retrieval lacks) but still subject to the same depth-allocation ceiling.

## §7 — A unifying picture

Read the five papers together and a coherent theory emerges:

1. **Latent two-hop is possible** in principle (Wang grokked transformers exist).
2. **It is not what standard pretraining produces** at GPT-3-class scale (Press's gap holds, Wang's grokking is the *exception*).
3. **The first hop works well** (Yang's probes).
4. **The second hop suffers from layer-allocation** — by the time it can start, the necessary knowledge has been overwritten in the residual stream (Biran's logit-lens analysis).
5. **The bridge entity needs representational alignment** that purely-synthetic data can't provide (Balesni's controls).
6. **CoT works** because it serialises the hops across forward passes, materialising the bridge as a token that the next pass can consume normally — restoring the full depth budget for the second hop and providing the alignment the bridge needs.

This explains, formally, why every technique in [199-multi-hop-reasoning-techniques-arc](199-multi-hop-reasoning-techniques-arc.md) emits explicit reasoning steps, why DSPy's `MultiHop` module loops `(num_hops, num_docs)` rather than asking the LM to compose silently, and why RL-trained search agents (Search-R1, R1-Searcher) emit `<search>` tokens between reasoning steps.

## Variants and adjacent work

- **Implicit reasoning vs explicit CoT trade-offs.** The recurrent-depth and looped-transformer line ([32-recurrent-depth-implicit-reasoning](32-recurrent-depth-implicit-reasoning.md)) is an architectural attempt to add *internal* depth without emitted tokens — a structural competitor to CoT.
- **Reflection-token training** (Self-RAG, Search-R1) bridges the gap by making *some* of the reasoning chain implicit (model decides when to retrieve) and *some* explicit (the search action is a real token).
- **Distilled reasoning traces** (DeepSeek-R1) compress long CoT into more efficient forms while retaining the bridge-entity materialisation property.
- **Mechanistic interpretability of multi-hop circuits** — circuit-level analyses (e.g. *Patchscopes*, *Path Patching* on multi-hop questions) confirm Biran's layer-allocation thesis and identify specific attention heads as bridge-entity carriers.
- **Graph-grounded multi-hop** ([200-graph-grounded-multi-hop-retrieval](200-graph-grounded-multi-hop-retrieval.md)) sidesteps the architectural limitation entirely by externalising the multi-hop chain into the retrieval index.

## Failure modes and limitations of the canon

- **Synthetic-fact studies don't fully transfer to natural data.** Balesni's controls are the most defensible methodology for isolating mechanism, but real-world multi-hop is messier.
- **Probing artifacts.** Probe-based "latent reasoning" claims are sensitive to probe capacity; some probes overstate latent computation. Yang et al. controls for this carefully but it remains a methodological caveat.
- **Layer-allocation is one mechanism among many.** Hopping-Too-Late doesn't claim to explain *all* multi-hop failures; long-distance attention, in-context-knowledge interference, and tokenisation effects all contribute.
- **Grokking transfer.** Wang's grokked transformers don't generalise OOD on composition — so "we just need to grokk-train at scale" is not a full solution.
- **Most studies are 2-hop.** The picture for ≥3 hops is much less developed; conjecturally, the layer-allocation problem compounds, but the mechanistic studies are still nascent.

## When this canon matters in practice

**It matters most** when you're (a) deciding whether to add explicit CoT to an agent, (b) evaluating whether a base-model upgrade should fix a multi-hop regression, (c) debugging "the model knew both facts but answered wrong" failures, (d) designing benchmarks that distinguish knowledge from composition, or (e) considering an architectural change (recurrent depth, looped transformer) as an alternative to CoT.

**It matters less** for systems that *already* externalise the chain — graph-RAG, DSPy multi-hop, RL-trained search agents have already paid for the workaround. In those regimes the bottleneck is retrieval quality and decomposition fidelity, not the composition gap per se.

## Implications for harness engineering

- **Always emit the chain on multi-hop tasks.** Don't bet on latent composition; the gap is real and scaling-resistant. Cf. [13-react](13-react.md), [14-reflexion](14-reflexion.md), [16-plan-and-solve](16-plan-and-solve.md).
- **Materialise the bridge entity as a token.** Self-Ask's follow-up sub-question, IRCoT's per-sentence retrieval, DSPy's `Hop` loop — all do this for the same architectural reason.
- **Budget depth, not just parameters.** A 70B model with shallow effective depth on the second hop won't beat a 13B model with explicit CoT. Cf. [110-transformer-llm-architecture-for-agent-builders](110-transformer-llm-architecture-for-agent-builders.md).
- **Treat scale-up as orthogonal to multi-hop fix.** Don't promise reviewers / stakeholders that a base-model upgrade will fix a multi-hop regression — Press's gap is constant in scale.
- **Externalise the chain whenever practical.** Graph-RAG, DSPy multi-hop, structured tool calls — externalised chains side-step the architectural limit. Cf. [200-graph-grounded-multi-hop-retrieval](200-graph-grounded-multi-hop-retrieval.md).
- **Synthetic data for multi-hop SFT needs natural anchors.** Balesni's controls show purely-synthetic two-hop training data breaks the bridge alignment. Mix synthetic chains with natural-data anchors. Cf. [142-trajectory-simulation-agents](142-trajectory-simulation-agents.md).
- **Mechanistic regression tests.** When debugging multi-hop failures, log the bridge entity (when known) and probe whether it's even computed. Adapt Yang/Biran's probe methodology.
- **Recurrent-depth as an architectural lever.** [32-recurrent-depth-implicit-reasoning](32-recurrent-depth-implicit-reasoning.md) is a structural alternative to CoT — useful when token-emission is too expensive.
- **Reflection-token discipline.** Self-RAG-style reflection tokens give a controlled middle ground between fully latent and fully emitted chains. Cf. [199-multi-hop-reasoning-techniques-arc](199-multi-hop-reasoning-techniques-arc.md) Phase 3.
- **Cite the right paper.** When justifying agentic CoT to a sceptical reviewer, Press → Biran → Balesni is the cleanest three-paper chain. The mechanism is now well-attested.
- **Beware "implicit reasoning" claims without probe-based evidence.** "The model can do it" benchmarks rarely distinguish memorisation from composition; demand probe + supporting-fact F1 + MuSiQue-Full evidence.
- **Don't conflate compositionality gap with multi-hop *retrieval* gap.** They're related but separable: the retrieval gap is about finding the right passages, the compositionality gap is about chaining them. Both must be closed.

**The one-line takeaway for harness designers:** Treat the compositionality gap as an *architectural* constant — externalise the chain (CoT, sub-questions, graph walks) instead of waiting for scale to fix what scale doesn't fix.
