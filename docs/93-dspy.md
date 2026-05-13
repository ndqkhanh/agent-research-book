# 93 — DSPy: Compiling Declarative LM Pipelines

**Paper.** Omar Khattab, Arnav Singhvi, Paridhi Maheshwari, Zhiyuan Zhang, Keshav Santhanam, Sri Vardhamanan, Saiful Haq, Ashutosh Sharma, Thomas T. Joshi, Hanna Moazam, Heather Miller, Matei Zaharia, Christopher Potts — *DSPy: Compiling Declarative Language Model Calls into Self-Improving Pipelines* — ICLR 2024 — arXiv:2310.03714 — Stanford / UC Berkeley / CMU / industry collaborators — 2023/2024.

**One-line definition.** A **programming model for LMs** that pairs **declarative signatures** and **composable modules** (Predict, Chain-of-Thought, ReAct, Retrieve, etc.) with **teleprompters**—optimizers that **compile** a pipeline from data: they search over bootstrapped demonstrations, instructions, and (optionally) finetune small models so that **you write programs, not hand-tuned prompt strings**.

## Why this paper matters (the prompt engineering compilation thesis)

LM pipelines often hard-code **prompt templates** per stage; that is brittle across models and domains. Khattab et al. treat the system as **compilation**: declare *what* each stage does (signature + graph) and a **metric**, and a **teleprompter** derives few-shot prompts or finetunes from **data**—**write programs, not prompts**; **learn** the invocation details by search and supervision, like optimizers over parameters instead of hand-tuned weights.

## Problem it solves

1. **Prompt brittleness.** A long template that works for one model, retriever, or domain often fails on the next; multi-stage systems multiply that sensitivity.  
2. **Manual prompt engineering that does not compose.** Chains-of-thought, ReAct, RAG, and reflection are published as *techniques* but in practice are copy-pasted as huge strings, not as reusable, optimizable components.  
3. **Unscalable “stacking.”** Pipelines in contemporary agent frameworks (LangChain-class) still often encode behavior through ad hoc templates, making it hard to **systematically** improve end-to-end quality when multiple LM calls must cooperate.

## Core idea in one paragraph

DSPy is a **text transformation graph**: **signatures** declare I/O roles; **modules** (CoT, ReAct, Retrieve, user `Module`s) replace ad hoc strings and hold **learnable** demos/LM choices. A **teleprompter** runs the program on a **trainset**, **bootstraps** traces per predictor when the **metric** passes, optionally **searches** over demo subsets (random/Optuna) or **finetunes** small LMs. Output: a **compiled** pipeline whose prompts match the metric, not a one-off template.

## Mechanism (step by step)

### (a) Signatures: typed LM I/O, not a wall of text

A **signature** is a tuple of **input** and **output** fields, optionally with an instruction; field names (e.g. `question`, `answer`, `context`, `search_query`) steer both formatting and the compiler’s later bootstrapping. Shorthand is allowed:

```python
qa = dspy.Predict("question -> answer")
qa(question="Where is Guaraní spoken?")
# Prediction(answer='...South America...')
```

The paper stresses signatures as the stable **interface**; concrete prompts are **emitted** by the compiler, not hand-authored for each task.

### (b) Modules: `Predict`, `ChainOfThought`, `ReAct`, `Retrieve`, compositions

- **`Predict`**: the basic wrapper—stores signature, optional LM override, and a **list of demonstrations**; in compile mode it records **traces** for the teleprompter.  
- **`ChainOfThought`**: implements CoT by **expanding the signature** with a rationale field and delegating to `Predict` on the extended signature (paper Figure-style sketch).  
- **`ReAct`**: a built-in **agent loop** over tools (e.g. retrieval) with a max-iteration cap—same conceptual substrate as [13-react](13-react.md) but as a **module** with optimizable demonstration sets.  
- **`Retrieve`**: non-LM “tool” module (ColBERTv2, Pyserini, Pinecone integrations in the paper’s experiments).  
- **User `Module` subclasses** compose these in `forward` (e.g. RAG: retrieve then `ChainOfThought("context, question -> answer")`).

Example RAG from the paper:

```python
class RAG(dspy.Module):
    def __init__(self, num_passages=3):
        self.retrieve = dspy.Retrieve(k=num_passages)
        self.generate_answer = dspy.ChainOfThought("context, question -> answer")
    def forward(self, question):
        context = self.retrieve(question).passages
        return self.generate_answer(context=context, question=question)
```

### (c) Teleprompters: `BootstrapFewShot`, `BootstrapFinetune`, and search-based variants (note on MIPRO)

The **ICLR 2024 paper** (arXiv:2310.03714) names and studies:

- **`BootstrapFewShot`**: run a **teacher** program (default: zero-shot student) on training inputs, collect full **traces** per `Predict` when the pipeline satisfies the **metric**, and attach those as demonstrations.  
- **`BootstrapFewShotWithRandomSearch`**: after candidate demos exist, **random search** selects which demonstration subsets to keep, validated against a **dev metric** (GSM8K code uses `gsm8k_accuracy`).  
- **`BootstrapFewShotWithOptuna`**: same spirit with **Tree-structured Parzen / Optuna**-style search over candidates (Appendix pseudocode in the paper).  
- **`BootstrapFinetune`**: turn bootstrapped traces into **supervision for finetuning** a smaller LM per predictor (e.g. T5-Large 770M), optionally with a **teacher** program supplying labels even when end labels are missing—metrics like `answer_passage_match` gate ungrounded answers.  
- **`LabeledFewShot`**: simple baseline—sample *k* labeled fields (e.g. 8) without trace bootstrapping.  
- **`Ensemble`**: run multiple **compiled** program variants and **reduce** (e.g. majority), used in both case studies.

**MIPRO** (multi-objective / instruction+example search used in the mature open-source `dspy` library) is **not defined** in the PDF text reviewed here. Treat MIPRO as a **lineage extension**: same “teleprompter = optimizer over prompt parameters” idea, with richer search than the paper’s random-search/Optuna bootstraps. For this corpus’s **paper-grounded** account, the canonical optimizers are **bootstrap + search (random / Optuna) + finetune + ensemble**.

### (d) The `compile()` loop: metric, bootstrapping, and search

Typical **compiler stages** in the paper:

1. **Candidate generation** — find all `Predict` instances, propose **demonstration** sets (and other discrete parameters) by executing teachers / students on train inputs with metric filtering.  
2. **Parameter optimization** — discrete search (random, Optuna) to maximize **validation** performance; or **finetune** from accepted traces.  
3. **Higher-order transforms** — e.g. **ensembles** over multiple good compiled candidates.

RAG example with EM:

```python
teleprompter = dspy.BootstrapFewShot(metric=dspy.evaluate.answer_exact_match)
compiled_rag = teleprompter.compile(RAG(), trainset=qa_trainset)
```

Custom metrics can be full Python (including another LM judge—conceptually aligned with [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md) and verifier-style checks in [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md)); the paper’s `answer_and_context_match` example combines EM with “answer substring of a retrieved passage” for **faithfulness**.

### (e) Self-improvement via demonstration generation

**Self-improvement** in DSPy is not weight update by default; it is **self-supervision of prompts**: successful multi-stage **trajectories** on train inputs become few-shot **demonstrations** for every module, including intermediate fields (e.g. rationale, queries) the developer never hand-labeled. **Nested bootstrapping** uses an already-compiled program as **teacher** for a second `compile` pass (`bootstrap×2` in results). **Teacher–student** composition lets a **large, expensive** compiled pipeline supervise a **smaller** one via `teacher=` in `BootstrapFinetune`, aligning with distillation thinking.

## Empirical results (concrete numbers)

**GSM8K** (200 train / 300 dev / 1.3k test; final-number accuracy). Table 1 (abstract: vs few-shot and vs expert prompt stacks):

- **Vanilla** `Predict`: zero-shot **25.2%** test; `LabeledFewShot` **33.1%** dev; first **bootstrap** **44.0%** dev; **bootstrap×2** **64.7%** dev, **61.7%** test; vanilla **+ensemble** **61.9%** test.  
- **CoT** `ChainOfThought("question -> answer")`: few-shot w/ **+human CoT** in train **72.4%** test; **bootstrap** w/out human chains **72.9%** test; **+ensemble** **88.3%** dev, **81.6%** test.  
- **Reflection** (`ThoughtReflection`, five chains + `MultiChainComparison`): **bootstrap** **83.0%** dev, **76.0%** test; **+ensemble** **86.7%** dev (GPT-3.5, test not tabulated) and **49.0%** dev / **46.9%** test (llama2-13b-chat).

**llama2-13b-chat** moves from **9.4%** test (vanilla none) to **46.9%** test (reflection + ensemble) in the table—i.e. the paper’s “**4–20% to 49–88%**” (Table 1 caption) range across program/LM/column choices.

**HotPotQA** (ColBERTv2, hard “bridge”-style training slice; Table 2): **BasicMultiHop** + **bootstrap** **48.7%** / **47.0%** Ans/Psg (GPT-3.5 dev), **39.6%** / **43.8%** test; **+ensemble** **54.7%** Ans dev, **45.6%** Ans test (half the test set, ∗). **Llama2-13b multihop** **36.4%** / **43.5%** test. **T5-Large** finetune: **39.3%** Ans, **46.0%** Psg dev; **200** labeled + **800** unlabeled. Yao et al. **ReAct** at **27.4%** EM (PaLM-540B) is a cited point of comparison; align indices carefully across papers.

## Variants and ablations

- **Program choice matters more than a single template tweak**: vanilla vs CoT vs **ThoughtReflection** traces different best points; **ensembles** add several points (e.g. CoT 72.9% → 81.6% test for GPT-3.5).  
- **Compilation strategy**: `LabeledFewShot` < `BootstrapFewShot` < **bootstrap×2** on weaker programs.  
- **Search**: random search and **Optuna** (paper) over demonstration subsets.  
- **Signature reuse**: the same `question -> answer` signature is reused for GSM8K and HotPotQA—**task adaptation** is pushed into **compiled** demos, not new string templates.  
- **Finetune vs ICL**: T5-Large finetune trades some accuracy for **massive** inference cost reduction vs proprietary chat models.

## Failure modes and limitations

- **Compile cost**: the paper states compilation is **minutes to tens of minutes** and **O(thousands)** of program evaluations (e.g. 10–20 trials on 150–300 validation items)—unlike a one-shot hand prompt.  
- **Metric specification is load-bearing**: garbage metric ⇒ bootstrapped demos optimize the wrong thing (faithfulness, safety, and tool use all need **task metrics**, echoing [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md)).  
- **Signature and decomposition design** still require judgment; the framework removes *string* anguish, not *problem factorization* anguish.  
- **Reproducibility across retrievers / APIs**: HotPot results depend on ColBERT+index; GSM8K numbers track **value extraction** heuristics.  
- **Scope of the 2023 text**: field-level typing is mostly strings; richer typed outputs are noted as future work.  
- **Ecosystem note**: post-paper teleprompters (e.g. MIPRO-style optimizers) may outperform early random/Optuna search but add new hyperparameters and compute.

## When to use, when not to

**Use** when you have (or can cheaply get) a **hundred-ish** training examples, a defensible **metric**, a **stable** pipeline graph, and you expect to **iterate** on models or retrievers—DSPy amortizes re-prompting across those changes. **Use** to **distill** large LMs or teacher pipelines into small finetuned predictors. **Avoid** (or be cautious) when labels/metrics are **scarce or misleading**, the task is **one-shot** with no room for search budget, or **latency** is dominated by compile-time exploration in production. **Avoid** expecting miracles if the program’s **control flow** cannot represent the true algorithm.

## Implications for harness engineering

**[42-langchain-deep-agents](42-langchain-deep-agents.md)**-style harnesses own **planning, tools, async**; DSPy **compiles** the **LM subgraph** (prompts / small weights). They **stack** (compiled prompts inside a Deep Agent). A DSPy **`ReAct`** is the same **tool-loop** family as [13-react](13-react.md), but **metric-optimizable**—not a full harness. **Metrics** from [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md) and [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md) map to `metric=` so bootstrapping aligns with **judge/verifier** targets. **Skill libraries** carry **knowledge and specs**; DSPy fills **demos** for *your* data. **Position:** DSPy = **declarative pipeline compiler**; free-form **agent** scaffolds = **runtime orchestration**—often combined.

## Connections to other work in this corpus

**ReAct** ([13-react](13-react.md)) as a module; **RAG/retrieval** (ColBERT family). **Metrics-as-programs** links judges/verifiers ([21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md)). App. B contrasts **chain-library hand prompts** vs DSPy’s **compile** move. **Ensemble** teleprompter ≈ program-level self-consistency.

## Key takeaways

1. **Signatures + modules** replace ad hoc templates with a **reusable, optimizable** interface.  
2. **Teleprompters** (bootstrap, random/Optuna search, finetune, ensemble) implement **actual compilation** to **few-shot prompts** or **weights**.  
3. Empirically, **compiled** small programs can match or beat **human CoT** demonstrations on **GSM8K** and beat **expert ReAct** prompts in the **HotPot** setup, with strong **T5** results for cost.  
4. The research agenda is **reproducible** comparisons: “LM X on task Y with **program P** and **compiler S**,” not inscrutable prompt blobs.

## References

- Khattab, O., et al. (2024). *DSPy: Compiling Declarative Language Model Calls into Self-Improving Pipelines.* ICLR 2024. arXiv:2310.03714.  
- Cobbe, et al. GSM8K.  
- Yang, et al. HotPotQA.  
- Yao, et al. ReAct (tool-using agents).  
- Wei, et al.; Yoran, et al. (building blocks for CoT / multi-chain comparison in modules).  
- Project code: `https://github.com/stanfordnlp/dspy` (paper footnote; framework evolved post-ICLR).
