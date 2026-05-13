# 82 — PoisonedRAG: Knowledge Corruption Attacks on RAG Systems

**Paper.** Wei Zou, Runpeng Geng, Binghui Wang, Jinyuan Jia — *PoisonedRAG: Knowledge Corruption Attacks to Retrieval-Augmented Generation of Large Language Models* — USENIX Security 2025 — arXiv:2402.07867 — 2024 (code: [PoisonedRAG](https://github.com/sleeepeer/PoisonedRAG)).

**One-line definition.** A *knowledge corruption* attack in which an adversary injects a small number of crafted documents into a RAG knowledge base so that, for chosen user queries, retrieval surfaces those documents and the generator LLM is steered to an attacker-chosen *target answer*—treating the corpus as writable policy, not read-only truth.

## Why this paper matters (RAG = critical attack surface; this is THE foundational paper on RAG attacks)

Most RAG work optimizes quality or cost; this paper shows the **knowledge store** is a security boundary: document-level write (edits, crawls, insider uploads) changes **inference-time** answers without weight updates—unlike training-data poisoning, which the authors explicitly do *not* require. The contribution is a clean optimization + **black- vs white-box** retriever analysis + **million-token-scale** numbers—i.e. a portable **RAG threat model** for red teams and harness design.

## Problem it solves — actually FRAMES (this is an attack paper, so frame as "what attack capability it characterizes")

The paper does not “solve” safety; it **characterizes** a capability: *targeted answer control via corpus writes*. The adversary picks a set of **target questions** \(Q_i\) and arbitrary **target answers** \(R_i\) (often false or biased). The attack succeeds if, after injecting a budget of malicious texts \(\Gamma\) into database \(D\), the RAG pipeline yields \(R_i\) for \(Q_i\) when users ask \(Q_i\). Success is measured by **Attack Success Rate (ASR)**: fraction of target questions for which the LLM’s output matches the attacker’s desired answer (evaluated with **substring matching** for close-ended QA, validated against human evaluation). The framed question is not “how to build RAG” but **how little write access and how little side information** are needed to corrupt RAG outputs at scale.

## Core idea in one paragraph (the white-box vs black-box attack constructions; budgeted poison documents)

Knowledge corruption is cast as maximizing the fraction of questions for which `LLM(Q, E(Q; D∪Γ))` equals the target answer, subject to retrieval returning the poison from the corrupted store. A single malicious document \(P\) must satisfy **two necessary conditions**: (1) **retrieval condition**—\(P\) lands in the top-\(k\) results for \(Q\); (2) **generation condition**—when \(P\) (or \(P\) among others) is in context, the LM outputs the target answer. Because naively maximizing embedding similarity to \(Q\) can break faithfulness, the authors **decompose** \(P = S \oplus I\): sub-text **\(I\)** is generated (e.g., by GPT-4) so that *alone* it causes the victim LM to emit the target answer for \(Q\); sub-text **\(S\)** is chosen to align the full string with \(Q\) in embedding space **without** destroying \(I\)’s effect. In **black-box** mode the attacker cannot query or differentiate the retriever: they set **\(S = Q\)** (the target question itself), so \(P = Q \oplus I\)—brutally simple yet highly effective. In **white-box** mode they **optimize \(S\)** (e.g., via HotFlip or synonym-preserving TextFooler) to maximize `Sim(f_Q(Q), f_T(S⊕I))` for the victim retriever’s encoders, improving rank when the question is *not* literally prepended. Attackers inject **\(N\)** poison documents per target question (default **\(N=5\)**) into databases of **2.68M–8.84M** clean passages.

## Mechanism (step by step)

**(a) Attacker capabilities / threat model.** The victim **LLM is a black box** (no parameter access, no queries for attack optimization in the formal model). The victim **database contents are not readable** to the attacker except for write/inject. The attacker **may or may not know the retriever**: **black-box** = no retriever parameters and no retriever queries; **white-box** = retriever weights available (e.g., public Contriever on Hugging Face). The attacker can inject up to **\(N\)** malicious texts **per target question** (e.g., malicious Wikipedia edits, SEO pages ingested into a crawl, or insider uploads). Ground-truth answers in the corpus are **not** assumed visible.

**(b) White-box vs black-box optimization.** **Generation leg:** both settings use an **attacker-chosen LM** (experiments use **GPT-4**) to synthesize **\(I\)** with a template: given \((Q, R)\), produce a short “corpus” such that answering \(Q\) with \(I\) as context yields \(R\); repeat up to **\(L\)** trials (default **\(L \le 50\)**, with **~1.6–2.7 average GPT-4 queries** per poison on NQ/MS-MARCO). **Retrieval leg:** **black-box** sets **\(S=Q\)** so the passage is semantically tethered to the query. **White-box** solves Equation (5): gradient-based or adversarial-word search on **\(S\)** to maximize retriever similarity between **\(Q\)** and **\(S \oplus I\)**, optionally co-optimizing **\(S\)** and **\(I\)** for semantics (TextFooler path).

**(c) Corpus injection.** Poisons are added to \(D\); evaluation uses standard top-\(k\) retrieval (default **\(k=5\)**) and a fixed system prompt (Appendix B in the paper). ASR is measured on **100** held-out target questions (10 per trial × 10 repeats) with GPT-4–sampled **false** target answers where needed.

**(d) Crafted text properties.** Poisons are **not** random adversarial Unicode: **\(I\)** is fluent, GPT-4–authored narrative that *supports* the false fact (example in the paper: Tim Cook as OpenAI CEO). That makes the string **low perplexity** and “normal” when concatenated with a natural-language **\(S\)**, unlike classic prompt-injection strings that are instruction-heavy and comparatively detectable. The paper argues such text is simultaneously **retrievable** (high embedding match to \(Q\) via **\(S\)**) and **persuasive in context** (generation via **\(I\)**), unlike Zhong et al.’s corpus-poisoning passages that optimize retrieval with nonsensical tokens but fail the generation leg.

**(e) How few documents flip the answer.** With **\(N=5\)** and **\(k=5\)**, poison–retrieval F1 often exceeds **0.9**; when **>3** of the top-\(k\) results are malicious, ASR nears ceiling. **~90%+** ASR is reported in the abstract with **five** poisons; main results give **97%** ASR on NQ (PaLM 2) by default.

## Empirical results (table with attack success rate across LMs / retrievers / corpora; how N poisoned docs scales)

**ASR (Table 1, \(N=5\), million-scale \(D\))** — NQ: PaLM2 **0.97**, GPT-4 **0.97**, GPT-3.5 **0.92** (black-box); HotpotQA up to **0.99**; MS-MARCO **~0.89–0.92** (weakest LM row: Vicuna-7B **0.74** on MS-MARCO). White-box NQ: GPT-4 **0.99**. **Retrievers** (Contriever / Contriever-ms / ANCE): NQ/Hotpot **0.95–0.99**; MS-MARCO **0.83–0.91** (Table 5). **Baselines** missing one condition: Naive **0.03**; Zhong-style corpus poison **~0.01**; GCG **0.02**; prompt-injection **0.62** (NQ) / **0.93** (Hotpot); **P=I** disinformation **0.69** (NQ). **\(N\)** vs **\(k\)** (Fig. 4): ASR rises for **\(N \le k\)**, then flattens; **\(k>N\)** dilutes with clean text unless the attacker grows **\(N\)**. **Self-RAG / CRAG** **0.70–0.87** ASR; **21M**-article Wikipedia chatbot **0.95–1.0**; **ReAct** **0.72 / 0.58 / 0.52** (NQ / Hotpot / MS-MARCO).

## Variants and defenses (perplexity filter, corpus integrity, etc.)

- **Input paraphrase before retrieve:** GPT-4 paraphrases **\(Q\)** (5 variants); attacker poisons for *original* \(Q\), so match may break. ASR **drops** (e.g., NQ black-box **0.97 → 0.87**) but **remains high**; not a full defense (Table 12).
- **Perplexity filtering:** PPL of clean vs malicious text overlaps heavily because **\(I\)** is fluent and **\(S\)** is natural; ROC **AUC ≈ 0.25 (black-box)** and **0.30 (white-box)** on NQ (Figure 6)—unusable as a stand-alone filter.
- **Duplicate / near-duplicate hashing:** each poison uses a distinct **\(I\)**, so dedup does nothing (ASR **unchanged**, Table 13).
- **Knowledge expansion (retrieve more):** increase **\(k\)** so *clean* context dilutes poisons. At **\(k=50\)** with **\(N=5\)**, HotpotQA ASR is still **41% (black-box)** and **43% (white-box)**; attacker can **inject larger \(N\)** to push ASR back up (Figures 24–26), and large **\(k\)** taxes latency and cost.
- **Manual error inspection** when outputs look wrong: noted as **impractical** at scale.
- **Training-set poisoning defenses** from ML literature are **mostly inapplicable** because weights are not poisoned.

## Failure modes and limitations (of the attack, and of proposed defenses)

**Attack limitations:** ASR is **not 100%**; authors give qualitative failure cases (Appendix I). **Open-ended** QA is **out of scope** in the main evaluation (deferred to discussion). **Joint optimization across many target questions** is left future work—current poisons are **independent** per question (suboptimal for a global campaign). **Non-target** queries are largely unaffected: only **0.3% / 0.9%** of random non-target questions retrieve poison (black/white), and **0% / 0.4%** of answers change (NQ)—stealthy against broad QA, dangerous for the chosen queries. **Defenses’ limitations:** paraphrase and larger **\(k\)** are **costly** and **partial**; PPL and dedup are **largely ineffective** against this threat; knowledge expansion **fails** under adaptive **\(N\)** and increases **compute**.

## When to use defenses (since this is an attack paper, the "use" guidance is: when defending RAG)

Treat corpus writes as **authorization-sensitive**: anyone who can add documents to an index used by a production LLM has a **content-supply attack** comparable to `sudo` on facts. **Use** (combine) **provenance** (signed corpora, allowlisted sources), **separate** user queries from **trusted** knowledge channels where possible, **retrieval gates** (cross-encoder reranking with tamper evidence), **k consistency / voting** over disjoint corpora, **output constraints** and **structured citation verification**, and **integrity monitoring** on high-risk collections—not single-feature perplexity or naive dedup. Re-run **red-team** with PoisonedRAG-style dual objectives whenever you add **untrusted** ingestion paths.

## Implications for harness engineering

A harness that wires an agent to RAG, tools, and external memory must assume **knowledge is adversarial by default** whenever ingestion is not cryptographically or procedurally bound. [22-guardrails-prompt-injection](22-guardrails-prompt-injection.md) covers **instruction** injection; PoisonedRAG shows **data-plane** attacks need not be instructions at all—**semantic poison** in the retriever’s index is enough. [25-agentic-rag](25-agentic-rag.md) and multi-step **agent** loops expand retrieval **surface** and **re-query**—each hop is a new chance to pull poisoned chunks (see ReAct results). [35-malicious-intermediary-attacks](35-malicious-intermediary-attacks.md) reframes *who* you trust; PoisonedRAG reframes *what* you trust: **any intermediary that can host or modify corpus entries** is in the TCB for factual outputs. [49-agents-of-chaos-red-teaming](49-agents-of-chaos-red-teaming.md) should include **knowledge-base corruption** scenarios as **first-class**—not only jailbreaks on the system prompt. Position **PoisonedRAG** as the **canonical** joint threat model: **(retrieval rank) × (LM faithfulness to context)**.

## Connections to other work in this corpus

Isolates the **index** (not just prompts/tools) as a policy channel. “Grounding” to a **poisoned** store is anti-grounding. Multi-hop RAG and **agent memory** compound exposure unless each hop re-verifies. **N vs k** tradeoffs: static **larger k** may help until the adversary injects more **N**; **provenance** matters more than a single feature (e.g. PPL).

## Key takeaways

1. RAG’s **knowledge base** is a **security boundary**: small, semantic injections can **flip answers** for chosen queries.  
2. Viability rests on two **necessary** conditions: **be retrieved** and **steer the generator**—defenses must address **both**; neither retrieval-only nor LM-only prior art suffices.  
3. A **strong** attacker model (**no** LLM or DB read, **no** retriever in black-box) still reaches **~97%** ASR on NQ with **5** poisons in **2.68M** passages.  
4. **“Advanced RAG”** (Self-RAG, CRAG) and **ReAct** agents remain **vulnerable**; scale to **20M+** Wikipedia chunks still yields **0.9+** ASR in the paper’s chatbot case study.  
5. **Paraphrase** and **larger k** are **partial** mitigations; **PPL** and **dedup** are **largely ineffective** against this attack.  
6. For harness engineering, **untrusted corpora** demand **provenance, isolation, and cross-checks**—treating retrieval as **policy-bearing input**, not oracle truth.

## References

- Zou, W., Geng, R., Wang, B., Jia, J. *PoisonedRAG: Knowledge Corruption Attacks to Retrieval-Augmented Generation of Large Language Models.* USENIX Security 2025. arXiv:2402.07867, 2024. [https://arxiv.org/abs/2402.07867](https://arxiv.org/abs/2402.07867)  
- Related (baselines / context): Liu et al., formalizing prompt injection; Zhong et al., poisoning retrieval corpora; Zhong *vs.* this work on **generation** requirement; RAG original (Lewis et al.); ReAct (Yao et al.) for agent setting.
