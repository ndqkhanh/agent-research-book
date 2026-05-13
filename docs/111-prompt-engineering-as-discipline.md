# 111 — Prompt Engineering as a Discipline: Beyond Ad-Hoc Prompts to Templates, Few-Shot Selection, and Optimization Loops

**Sources.** Nagasubramanian, *Agentic AI for Engineers*, Chapter 6 (The Art of Prompting); Lakshmanan & Hapke, *Generative AI Design Patterns*, Chapter 6 (Pattern 20: Prompt Optimization); Ozdemir, *Building Agentic AI*, Chapter 5 (Enhancing Agents with Prompting); plus the academic literature on chain-of-thought (Wei et al. 2022), few-shot learning (Brown et al. 2020), and prompt optimization (Pryzant et al. 2023; DSPy / Khattab et al. 2023).

**One-line definition.** Prompt engineering as practiced today is a discipline with four named subproblems — *template design*, *few-shot example selection*, *reasoning elicitation*, and *prompt optimization* — each with battle-tested patterns; the difference between an amateur and a professional prompt is not creativity, it is disciplined application of these four moves combined with measurement and a feedback loop.

## Why this chapter matters

The phrase "prompt engineering" has become diluted. Some people mean "I write a sentence and the model does the thing"; others mean "I run a 200-trial Bayesian optimization over prompt templates with held-out evaluation sets." Those are different jobs. The first is consumption; the second is engineering.

For agent builders, the second is what matters. An agent's planner prompt is invoked thousands of times a day. A 5% accuracy lift on the planner cascades to 50% fewer downstream tool failures. That lift comes not from cleverness but from disciplined practice: the same prompt across runs gives the same baseline, the same ablations always reveal the same delta, and improvements are reproducible.

This chapter is the minimal practitioner's guide: the four subproblems, the patterns that solve each, the measurement loop that keeps you honest, and the explicit list of things that do *not* work.

## Problem it solves

Five common failure modes in undisciplined prompt work:

1. **No baseline.** "I changed the prompt and it seems better" — without an evaluation set, this is wishful thinking.
2. **One-shot to few-shot regression.** Adding more examples often *hurts* performance once you exceed the model's effective utilization region.
3. **Template instability.** Small whitespace, punctuation, or wording changes shift accuracy by 10+ points; you cannot tell which change caused which delta.
4. **Reasoning over-elicitation.** "Let's think step by step" added to every prompt; sometimes it helps, sometimes it adds noise. No principled rule.
5. **Optimization theater.** Manually tweaking phrasing vs. running a structured optimizer like DSPy / OPRO; one of these compounds, the other doesn't.

Each is solved by treating prompts as code: versioned, tested, optimized.

## Core idea in one paragraph

A production prompt has four orthogonal axes you can optimise independently. **Template design** decides the structure: role / task / context / examples / constraints / output format. **Few-shot example selection** decides which demonstrations to include — *which* matters more than *how many*, and the optimal K is task-dependent (often 2–4, never as high as you'd guess). **Reasoning elicitation** decides whether and how to invoke chain-of-thought, self-consistency, plan-and-solve, or tree-of-thoughts. **Prompt optimization** runs an outer loop — manual A/B, automated search (OPRO), or compiler-style (DSPy) — that improves the template against an evaluation set. Each axis is independent: improving template design does not change which examples are best; improving examples does not change whether CoT helps. Treating them as four separate jobs is what turns prompt engineering from craft into discipline.

## Mechanism (step by step)

### 1. Template design — the six slots

A robust agent prompt fills six slots in a stable order:

```text
[ROLE]        You are a senior software engineer specialising in...
[TASK]        Given the following code, identify and fix the bug.
[CONTEXT]     The code below is run in production with PostgreSQL...
[EXAMPLES]    Example 1: <buggy code> → <fix>
              Example 2: ...
[CONSTRAINTS] - Output only the fixed code.
              - Do not modify lines outside the bug location.
              - Match the existing indentation.
[OUTPUT]      Return JSON with keys: fix, explanation, confidence.
```

Three rules:
- **Slot order matters.** Role → Task → Context → Examples → Constraints → Output is the canonical order for instruction-tuned models. Reversing slots costs accuracy.
- **One responsibility per slot.** Don't put constraints inside the role description; don't put output format in the task statement.
- **Stable text, dynamic content.** Static slots (role, output format) belong at the top for prompt caching; dynamic content (current task, retrieved context) at the bottom.

### 2. Few-shot selection — which examples, not how many

Few-shot prompting works because the model picks up the input/output pattern from examples and applies it to the new input. But:
- **More examples ≠ more accuracy.** Past 4–8 examples, the model's accuracy plateaus or drops (per the K=4 sweet spot in [107-memento-cbr-memory](107-memento-cbr-memory.md)).
- **Diversity beats density.** Two examples covering different sub-cases beat ten examples of the same sub-case.
- **Hardest-to-easiest order.** Order examples from most-similar-to-current-task to least-similar; the closest example anchors the pattern.
- **Selection should be retrieval-based, not random.** A fixed example bank with embedding-based selection per query outperforms a static list.

Three concrete patterns:
- **Static k-shot.** Hand-picked examples baked into the prompt. Cheap, good for narrow tasks.
- **Retrieved k-shot.** Embedding-based retrieval of K examples from a bank per query. Better generalisation; this is the [25-agentic-rag](25-agentic-rag.md) and [107-memento-cbr-memory](107-memento-cbr-memory.md) pattern applied to in-context demonstrations.
- **Adaptive k-shot.** Start with K=0; add examples only when the model's confidence is below a threshold. Cheaper at inference.

### 3. Reasoning elicitation — when "think step by step" helps

The literature on reasoning elicitation is messy. The robust generalizations:

- **Chain-of-Thought (CoT)** ("Let's think step by step") helps on multi-step reasoning tasks: math, logic puzzles, multi-hop QA. It hurts on tasks that don't decompose into steps (sentiment classification, pure recall).
- **Self-consistency.** Sample multiple CoT traces, take majority vote. Wins on tasks with verifiable correct answers; loses on creative tasks.
- **Plan-and-Solve** (Wang et al. 2023, [16-plan-and-solve](16-plan-and-solve.md)). "First make a plan, then execute it." Helps when the task has multiple sub-goals.
- **Tree-of-Thoughts** (Yao et al. 2023, [15-tree-of-thoughts-lats](15-tree-of-thoughts-lats.md)). Search over reasoning branches with a value function. Wins on combinatorial search; expensive elsewhere.
- **ReAct** (Yao et al. 2022, [13-react](13-react.md)). Interleave reasoning and tool calls. The right pattern for almost all agent loops.

Rule of thumb: **default to CoT for reasoning-heavy tasks, ReAct for tool-using tasks, plain instruction for classification/recall**.

### 4. Prompt optimization — the outer loop

Manual prompt tweaking is unreliable. Three structured approaches:

- **Manual A/B with eval sets.** Define an evaluation set of 50–500 examples with ground-truth answers. Run baseline prompt + variant; compare metrics. The minimum bar.
- **OPRO (Optimization by PROmpting).** Use an LLM to propose new prompts, evaluate them, feed scores back. Yang et al. 2023.
- **DSPy / declarative compilation.** Specify the *task signature* and *desired metric*; the compiler optimises the prompt automatically using teleprompters (BootstrapFewShot, MIPROv2). [93-dspy](93-dspy.md). Removes manual prompt tweaking entirely for narrow tasks.

For agent builders: **use DSPy for narrow, well-specified subtasks (classifier, extractor, scorer); use manual A/B for the planner prompt** where the search space is too large for current optimizers.

### 5. Format anchoring — the underrated lever

Output format affects the model's internal "mode" of generation. Four observations:

- **JSON output mode** ("Return JSON with keys X, Y, Z") forces the model into a more constrained generation style. Pair with grammar enforcement ([112-constrained-decoding](112-constrained-decoding.md)) for reliability.
- **Markdown output** is the default "reasonable assistant" mode; verbose, explanatory, polished.
- **Code-only output** ("Output only Python code") suppresses prose; useful for code-gen agents.
- **Tagged sections** (`<plan>...</plan><action>...</action>`) are the most stable for downstream parsing; XML-like tags survive tokenization and are ignored by the model's natural tendency to add prose.

### 6. Robustness patterns

Three patterns that each prevent a class of failure:

- **System-message persistence.** Repeat the most important constraint at the start *and* end of the prompt. Counters mid-context attention dispersion ([110-transformer-llm-architecture-for-agent-builders](110-transformer-llm-architecture-for-agent-builders.md)).
- **Negative examples.** Include "what NOT to do" examples. Teaches the boundary of the desired behavior more sharply than positives alone.
- **Refusal hooks.** Explicit instructions like "If unsure, output `{"answer": null, "reason": "..."}`" prevent the model from hallucinating when uncertain.

## Empirical anchors

- **K=4 sweet spot for few-shot.** Recurring across [107-memento-cbr-memory](107-memento-cbr-memory.md) and many practical RAG/CBR systems.
- **CoT helps math, hurts classification.** Wei et al. 2022 + many follow-ups.
- **Format swap delta.** Same content rendered as JSON vs Markdown vs tagged sections produces 5–15% accuracy differences on downstream parsing reliability.
- **DSPy gains.** Reported 10–30% accuracy improvements on narrow tasks vs manually-engineered prompts.
- **Position bias.** First and last positions in a context are 2–3× more likely to be recalled accurately than middle positions (Liu et al. 2023).

## Variants and counter-arguments addressed

- **"Just write a clear prompt."** Works for one-shot demos; fails for production. The variance across "clearly written" prompts is huge; only measurement makes the difference visible.
- **"Larger models don't need prompt engineering."** Larger models are more *robust* to prompt variation but still benefit from disciplined templates. The gain from optimisation shrinks but doesn't vanish.
- **"Prompt engineering will be obsolete with reasoning models."** Reasoning models (o1, o3) shift the work from CoT-elicitation to task-decomposition. The discipline survives, just with different weights on the four axes.
- **"DSPy makes manual work obsolete."** DSPy is excellent for narrow well-specified subtasks; the planner-of-an-agent prompt remains a manual job because the search space is too open-ended.

## Failure modes and limitations

1. **No evaluation set.** The first and biggest failure. "Looks better" is not data.
2. **Overfitting to small eval sets.** 20-example eval sets can be hill-climbed in 5 minutes to 100% with no real generalisation.
3. **Tokenizer-sensitive prompts.** A prompt tuned on GPT-4 may degrade on Llama because tokenization shifts attention weights subtly.
4. **Stale baselines.** Models change frequently; a prompt optimal for GPT-4-2024 may be suboptimal for GPT-4-2026. Re-baseline periodically.
5. **Format drift.** Telling the model to output JSON works most of the time; sometimes it adds prose. Without grammar enforcement, downstream parsers will fail.
6. **Reasoning over-elicitation cost.** CoT adds tokens; tokens cost. On easy tasks, the extra tokens are pure overhead.
7. **Few-shot poisoning.** A bad example in the bank corrupts retrieval-based few-shot. See [82-poisonedrag](82-poisonedrag.md) for the analogous threat in retrieval.

## When to use, when not

**Use the full discipline (template + selection + elicitation + optimization) when** the prompt is in production, invoked at high volume, and a measurable accuracy lift would compound. Agent planner prompts, classification prompts, extraction prompts.

**Use lightweight prompt design (template + maybe k-shot) when** the prompt is a one-off or exploratory.

**Skip elaborate optimization when** the model and the task are both small and well-known, or when iteration speed is the priority and the prompt is going to be rewritten weekly.

## Implications for harness engineering

- **Treat prompts as versioned artifacts.** Every prompt in the harness should be in a file, in git, with a name and a version. See [29-dive-into-claude-code](29-dive-into-claude-code.md) for the prompts-as-code convention.
- **Build an evaluation harness alongside the agent harness.** Without an eval loop, prompt changes are speculation. See [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md), [115-evaluating-llm-systems](115-evaluating-llm-systems.md).
- **Externalize examples to a bank.** Retrieve K=4 per call. This is exactly the [107-memento-cbr-memory](107-memento-cbr-memory.md) pattern applied to in-context demonstrations.
- **Use DSPy for narrow components.** Classifier-style subagents are a good DSPy fit; the planner is not. See [93-dspy](93-dspy.md).
- **Format anchor with tags or JSON + grammar.** Tagged sections are robust; JSON without grammar enforcement is not.
- **Repeat critical constraints at the prompt boundary** (start *and* end). Counters mid-context attention loss.
- **Audit prompt cache hit rate.** Static prefix at the top, dynamic content at the bottom; this is a 5–10× cost lever per [110-transformer-llm-architecture-for-agent-builders](110-transformer-llm-architecture-for-agent-builders.md).

The one-sentence takeaway: **prompt engineering is four jobs (template, examples, reasoning, optimization) with measurement under all of them — without measurement it is theatre, with measurement it compounds.**

## See also

- [13-react](13-react.md), [14-reflexion](14-reflexion.md), [15-tree-of-thoughts-lats](15-tree-of-thoughts-lats.md), [16-plan-and-solve](16-plan-and-solve.md), [17-rewoo](17-rewoo.md), [18-chain-of-verification-self-refine](18-chain-of-verification-self-refine.md) — the reasoning-elicitation patterns.
- [93-dspy](93-dspy.md) — declarative prompt compilation.
- [107-memento-cbr-memory](107-memento-cbr-memory.md) — retrieval-based example selection generalised.
- [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md), [115-evaluating-llm-systems](115-evaluating-llm-systems.md) — the evaluation loop that makes optimization possible.
- [112-constrained-decoding](112-constrained-decoding.md) — when format anchoring needs grammar enforcement to be reliable.
