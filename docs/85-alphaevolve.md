# 85 — AlphaEvolve: An Evolutionary Coding Agent for Scientific and Algorithmic Discovery

**Paper.** Alexander Novikov, Ngân Vũ, Marvin Eisenberger, Emilien Dupont, Po-Sen Huang, Adam Zsolt Wagner, Sergey Shirobokov, Borislav Kozlovskii, Francisco J. R. Ruiz, Abbas Mehrabian, M. Pawan Kumar, Abigail See, Swarat Chaudhuri, George Holland, Alex Davies, Sebastian Nowozin, Pushmeet Kohli, Matej Balog — *AlphaEvolve: A coding agent for scientific and algorithmic discovery* — arXiv:2506.13131 — Google DeepMind — June 2025.

**One-line definition.** Gemini-orchestrated evolutionary search over full programs: an LLM proposes SEARCH/REPLACE diffs on code marked for evolution, a pool of evaluators returns scalar (often multi-metric) scores, and a MAP-elites–inspired program database with island structure drives selection—producing provable algorithmic improvements (e.g. 4×4 complex matrix multiply in 48 multiplications vs Strassen’s 49) and production stack wins (Borg scheduling, Pallas matmul heuristics, TPU RTL, XLA/FlashAttention IR).

## Why this paper matters (DeepMind's evolutionary coding agent demonstrating real scientific discoveries)

Frontier LLM + **`evaluate()`-grounded evolution** yields **published algorithmic records** (48 mults for 4×4 complex, 14 improved matmul tensors) and **shipped systems** (Borg +0.7% capacity, Gemini kernels, TPU RTL, XLA)—not benchmark-only. The 56-year characteristic-0 gap below Strassen’s 49 is checkable classical CS; infrastructure wins show the same harness generalizes.

## Problem it solves (numbered list — algorithm discovery beyond human heuristics, scientific search w/ verifiable evals)

1. **Open algorithm design under expensive, non-obvious search spaces** — e.g. low-rank tensor decompositions for matrix multiplication, where the minimum rank for 3×3 is still unknown but improvements on many ⟨m,n,p⟩ upper bounds are achievable.
2. **Constructive mathematics and combinatorial geometry** — discover objects (packings, graphs, point sets) that beat published records when an automatic scoring function can judge quality.
3. **Infrastructure superoptimization** — improve interpretable heuristics and code paths (schedulers, kernels, hardware RTL, compiler output) where correctness can be checked by simulators, hardware regression, or numerical comparison to references.
4. **Reliability of long-horizon LLM “discovery”** — replace pure language-only iteration with a closed loop: every candidate is a program, and **feedback is machine-graded** so wrong suggestions do not compound across generations.
5. **Scaling test-time compute without hand-written mutation operators** — classical genetic programming needs crafted crossover/mutation; here the LLM plays the role of a domain-informed mutation operator over **entire files** in multiple languages.

## Core idea in one paragraph

**Seed** code in `# EVOLVE-BLOCK` regions + user **`evaluate` → {metric: float}** (maximize). **Database** samples parent + inspirations; **prompt sampler** packs scores, code, and optional domain text; **Gemini 2.0 Flash + Pro** output SEARCH/REPLACE **diffs**; **evaluators** execute (cascade, parallel, up to **~100** compute-hrs/solution in parallel settings). **MAP-Elites + islands** retain diversity; iterate until budget. Scale-up vs FunSearch: whole files, any language, multi-metric, SOTA LMs, thousands of LLM calls.

## Mechanism (step by step). Cover: (a) program representation; (b) prompt sampling for diversity; (c) ensemble of evaluators; (d) the evolutionary loop; (e) the discovered algorithms (matrix multiplication, datacenter scheduling, hardware design, etc.)

**(a) Program representation.** Solutions are **full source trees**, not a single Python function. Evolvable blocks sit inside a fixed skeleton; the rest provides wiring so `evaluate` can import and run the candidate. Short programs can be rewritten wholesale; long ones use a prescribed diff format:

```text
<<<<<<< SEARCH
  # exact span to find
=======
  # replacement
>>>>>>> REPLACE
```

This is how targeted mutations scale to hundreds of lines and multi-file changes. Table 1 in the paper contrasts prior FunSearch: AlphaEvolve evolves **entire files**, supports **any language**, and can use **thousands** (not millions) of LLM samples because SOTA models + rich prompts are more sample-efficient.

**(b) Prompt sampling for diversity.** The prompt may include: fixed human instructions and equations; **stochastic template** instantiations from a config; **rendered** prior programs with their `evaluate` scores and outputs; and optional **meta–prompt evolution** (co-evolved instructions in a separate database). Multi-metric training is used deliberately: high performers under *different* metrics are injected as inspirations to elicit structurally different children.

**(c) Ensemble of evaluators / evaluation stack.** The core is user `evaluate` → dict of floats. **Evaluation cascade** runs cheap or easy instances first, promoting survivors to harder cases—reducing spend on broken code. For qualitative criteria (e.g. simplicity) the paper allows **LLM-graded** auxiliary scores or filters, but the primary ground truth remains execution. **Parallel evaluation** (multiple random seeds, cluster jobs) is first-class: serial slow evals would starve the evolutionary loop. The paper reports that spending on the order of **100 compute-hours** per new solution is feasible when evaluation is parallelized. **Multi-score optimization** is supported throughout; the authors note multi-objective pressure often **helps the primary metric** by maintaining diversity in the prompt set.

**(d) The evolutionary loop.** The paper’s distributed controller pattern:

```text
parent_program, inspirations = database.sample()
prompt = prompt_sampler.build(parent_program, inspirations)
diff = llm.generate(prompt)                    # Flash + Pro ensemble
child_program = apply_diff(parent_program, diff)
results = evaluator.execute(child_program)
database.add(child_program, results)
```

The database balances exploration and exploitation using ideas from **MAP-Elites** and **island** models. Ablations (Section 4) show that removing **evolution** (repeatedly mutating only the *initial* program), removing **context**, disabling **meta-prompt evolution**, restricting to **loss-only** evolution, or using a **small base LLM only** all degrade the tensor-decomposition and kissing-number tasks at matched compute budgets—each component contributes materially.

**(e) Discovered and deployed artifacts — summary.** *Matrix multiplication (§3.1, Table 2):* for m,n,p ≤ 5 AlphaEvolve **matched or surpassed** prior ranks across the board; **14** shapes strictly improved (e.g. ⟨2,4,5⟩ 33→32, ⟨3,4,7⟩ 66→63, ⟨4,4,8⟩ 98→96, ⟨4,5,6⟩ 93→90; full table in paper/Colab). *Mathematics (§3.2):* 50+ problems — ~**75%** match prior best, **~20%** new SOTA (**Erdős** minimum overlap; **11D kissing number** 592→**593**). *Google (§3.3):* **Borg** heuristic via datacenter simulator → **0.7%** fleet capacity recovery, production rollout. **Pallas** tiling for Gemini matmul → **~23%** kernel speedup, **~1%** total training time, **months→days** engineering. **TPU** Verilog (matmul unit): area/power; expert-verified. **XLA** IR (FlashAttention stack): **+32%** kernel region, **+15%** pre/post; numeric checks vs reference.

## Empirical results (concrete discoveries: 4x4 complex matrix multiplication using 48 multiplications instead of 49 — Strassen — etc.)

- **⟨4,4,4⟩ / Strassen.** Recursive Strassen yields rank **49** (any field). Specialized **F₂** work hit **47** (AlphaTensor line) but does not carry as a general recursive scheme. For **56 years**, rank less than **49** over **characteristic 0** inside the standard **matmul-tensor decomposition** (the kind that composes to larger blocks) was open. AlphaEvolve finds rank **48** over **complex** numbers—first in that regime—artifacts in paper-linked **Colab**. (Non-tensor decompositions can use fewer than 49 mults but are not recursively composable in the same way; see paper footnote.)
- **Trace complexity:** Figure 4 shows **~15** accepted mutations for one matmul evolution (optimizer, loss, hyperparameter sweep)—not a one-line tweak.
- **Aggregate:** **14** improved matmul targets (Table 2); 50+ math tasks (~**75%** tie, **~20%** beat); production: **0.7%** Borg, **23%/1%** kernel/training, **32%/15%** IR, kissing **593**, Erdős improved.
- **LLM quality:** Ablations + Sec. 2.3: stronger base models improve outcomes under fixed harness budget.

## Variants and ablations

Section 4 (tensor decomposition + kissing numbers; Figure 8): **No evolution** (always reset to initial seed) tanks performance—**the database is load-bearing**. **No context in the prompt** and **small base LLM only** each cause large gaps. **No meta-prompt evolution** (tensor task) and **no full-file evolution** (loss-only baseline in matmul) both underperform the full system. Table 1 vs **FunSearch**: single-function → full repo; one objective → multi-metric; small code LMs → Gemini 2.0 Flash+Pro; millions of samples → thousands.

## Failure modes and limitations (compute-heavy, requires good evaluator, only verifiable problems)

1. **Need automated `evaluate`.** No faithful metric → out of scope (wet-lab science). LLM auxiliary grades exist but are secondary.
2. **Cost and wall-clock** — Expensive or serial evals starve the loop; **~100** compute-hrs/solution is feasible only with parallel evaluation clusters.
3. **Bad metrics** — Simulators can be gamed; cascades can discard rare winners; Borg uses held-out test snapshots.
4. **No global optimality proof** — Upper-bound improvements, not lower-bound mathematics.
5. **Encoding sensitivity** — Constructor vs search abstraction must match problem symmetry; TPU/XLA needed expert sign-off beyond numeric spot-checks.

## When to use, when not

**Use** when: the artifact is a **program**, quality is **machine-measurable**, you can pay **many eval rounds**, the solution space benefits from **diverse inspirations** and **long-range code edits** (not local parameter sweeps), and you are comfortable treating the LLM as a **learned mutation operator** with verifiable rollouts.

**Do not use** when: success criteria are **subjective** without a proxy metric, **evaluation is more expensive** than the value of marginal gains, **regulatory** or safety context forbids automatic execution of generated code, or the bottleneck is **problem formulation** rather than search—no evaluator, no loop.

## Implications for harness engineering. Reference [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [19-voyager-skill-libraries](19-voyager-skill-libraries.md), [55-hermes-agent-self-improving](55-hermes-agent-self-improving.md), [69-agent-world-self-evolving-training-arena](69-agent-world-self-evolving-training-arena.md). Position as: AlphaEvolve is the canonical "agent does science" reference; an evolutionary harness pattern.

AlphaEvolve instantiates **harnessed scientific agency**: claims become **code** and progress is **execution-gated**—aligned with [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md). The evolving **program archive** parallels memory/skill growth in [19-voyager-skill-libraries](19-voyager-skill-libraries.md), but entries are **diffable programs plus metrics**. Gemini **training** stack improved by the same agent loop links to [55-hermes-agent-self-improving](55-hermes-agent-self-improving.md) and [69-agent-world-self-evolving-training-arena](69-agent-world-self-evolving-training-arena.md): **verified environments** turn open search into measurable improvement. Reusable bundle: **spec + seed code + `evaluate` + MAP/island store + LLM mutator**—the canonical **evolutionary harness** reference for “agent does science with receipts.”

## Connections to other work in this corpus

**FunSearch / AlphaTensor:** extends FunSearch to repos + multi-metric + frontier LMs; matmul target overlaps AlphaTensor [26] but search is **LLM–evolution** not RL-only. **TTC:** paper frames runs as **test-time compute** via eval feedback. **Co-Scientist-style** NL agents vs **code+execute** ground truth. **Superoptimization** lineage, with LLM diffs and numeric oracles.

## Key takeaways

1. **Executable loop** — Ideas are **patches**; fitness is **evaluator output**, not rhetorical plausibility.
2. **Evaluator = objective** — Cascades, parallelism, multi-metric dictionaries **are** the optimization target design.
3. **Diversity machinery** — MAP-Elites/islands, prompt stochasticity, multi-metric inspirations, Flash+Pro **ensemble** reduce local minima in program space.
4. **Evidence bar** — **48 vs 49**, **0.7%** Borg, **23%/1%** kernel/training: claim strength tracks **measured** deltas.
5. **Ablate the harness** — Sec. 4 shows each component matters at fixed budget.

## References

- Novikov et al. (2025). *AlphaEvolve: A coding agent for scientific and algorithmic discovery.* arXiv:2506.13131. Google DeepMind. [Mathematical results Colab](https://colab.research.google.com/github/google-deepmind/alphaevolve_results/blob/master/mathematical_results.ipynb).
- Strassen (1969); Fawzi et al. / AlphaTensor-style **F₂ rank 47** vs **characteristic-0 rank 48** result in AlphaEvolve.
- Romera-Paredes et al. — **FunSearch**; MAP-Elites / island GA citations as in paper; Borg, Pallas/JAX, XLA, FlashAttention per §3.3.
