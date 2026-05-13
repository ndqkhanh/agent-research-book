# 08 — Context Compaction

**Definition.** Compaction is the practice of summarizing a long agent transcript — typically when it approaches the context window — and continuing from the summary rather than the raw history. Done well, the agent loses little; done poorly, compaction discards the exact detail that would have let the task finish.

## Problem it solves

Two distinct problems collapse into one. **Context-window exhaustion** is the obvious one: you physically cannot fit more tokens. **Context rot** is the subtler one: Chroma's 2025 research showed that *every* frontier model degrades as input length grows, even when the window isn't full. Performance follows a U-shaped curve — strong attention to the beginning and end of context, weak in the middle. By step 50 of an agent loop, the middle is where everything important lives, and the model ignores it.

Compaction addresses both by trading raw history for a distilled, re-ordered summary that (a) fits, (b) re-anchors the most relevant items at the edges of context, and (c) preserves what the agent will still need.

## Mechanism

The harness monitors total tokens on each turn. When usage crosses a threshold (Claude Code triggers around 95% of window capacity), it runs a compaction pass:

1. **Select what to keep verbatim.** The last N turns — often 3–5 — stay raw, because their formatting shapes the model's next move. The system prompt, memory files, and the current plan also stay raw.
2. **Summarize the rest.** Older turns are summarized, usually in a structured format: goal, decisions made, files touched, findings, open questions, failed approaches.
3. **Rebuild the transcript.** System prompt → memory → plan → structured summary → recent raw turns → current state.
4. **Continue.** The model sees a clean, compact transcript and keeps going.

Summarization strategies:

- **Recursive summarization.** Each compaction summarizes the *previous summary* plus new turns. Information decays over many compactions — older details fade like a game of telephone.
- **Hierarchical summarization.** Summaries at multiple granularities (recent detail → mid-depth summary → high-level gist). Preserves access to deeper detail when the compaction trigger fires far from when the detail was generated.
- **Targeted compaction.** Keep specific known-important artifacts (plan file, failing test outputs, latest diffs) verbatim; aggressively summarize everything else.
- **External offload.** Write key findings to a file and refer to them by path rather than inlining. The model re-reads the file when needed, avoiding permanent embedding in context.

## Concrete pattern

Structured summary template used during compaction:

```markdown
## Compacted history up to step 42

**Goal.** Add dark-mode toggle to settings; wire to theme store; update components.

**Decisions.**
- Chose CSS variables over Tailwind class flips (see discussion with user, step 8).
- Theme persisted to localStorage under `ui.theme`.

**Files touched.**
- `src/settings/ThemeToggle.tsx` (new)
- `src/lib/theme.ts:1-42` (added store)
- `src/components/Header.tsx:18-24` (consumed store)

**Findings.**
- Existing `useSettings` hook already reads from localStorage — reused.
- Storybook config in `.storybook/preview.ts` needs theme decorator update (not yet done).

**Open questions.**
- Does design want system-preference default? (Asked user step 30; awaiting answer.)

**Failed approaches.**
- Tried Tailwind `dark:` variants first; abandoned because non-Tailwind components existed.
```

This structure beats free-form prose because future turns can grep sections ("what's the open question?") and because the model generates predictably-shaped output when asked to extend it.

Hook points some harnesses expose:

- `PreCompact` — fire before summarization; good for saving artifacts.
- `PostCompact` — fire after; good for re-injecting pinned context (e.g., the plan file).

## Variants & related techniques

- **Memory files** ([09-memory-files.md](09-memory-files.md)) are the persistent cousin — compaction throws history away at session boundaries, memory files survive.
- **Subagent delegation** ([02-subagent-delegation.md](02-subagent-delegation.md)) can be understood as *anticipatory* compaction: the orchestrator never let the sub-detail into its own context in the first place.
- **Agentic Context Engineering (ACE, arXiv:2510.04618)** treats context as an evolving "playbook" updated through generation/reflection/curation instead of monolithic summaries — meant to resist drift over many compactions.
- **Retrieval over summaries.** Instead of compressing history linearly, index turns into a vector store and retrieve on demand. Better for very long sessions, worse for recency.
- **Sliding window with anchors.** Keep the last K turns + a handful of pinned anchors. Very simple, surprisingly effective for coding.

## Failure modes & anti-patterns

- **Compaction destroys the thing you needed.** A small detail ("we decided to use Pydantic v2") gets summarized out, and the agent later writes v1 code. Fix: pin important decisions into a memory file or plan file that is never compacted.
- **Compaction too late.** By the time the compaction fires, the model has been misbehaving for ten steps on rotten context. Fix: trigger earlier (80% not 95%), or compact proactively when a big tool result lands.
- **Compaction too eager.** Constant summarizing shreds narrative flow. Fix: larger window thresholds + don't summarize the last N turns.
- **Prompt format drift.** After compaction, the model's output style changes because the few-shot inertia of recent raw turns is gone. Fix: always keep the *most recent* turns verbatim, even when compacting.
- **Unverifiable compaction.** The model writes a summary the user can't audit. Fix: compaction happens in a deterministic format the user can inspect; log pre/post-compaction transcripts.
- **Information-free summaries.** "The user asked questions and the agent performed tasks." Fix: force structured sections; reject summaries lacking citations or decisions.

## When to use (and when not to)

**Use** compaction when:

- Sessions routinely exceed 50% of the context window.
- Long-running tasks span many phases; older phases' details aren't needed at step-level fidelity.
- Context rot is empirically hurting output quality (model forgetting instructions).

**Don't** over-rely on compaction when:

- You could put durable facts in a memory file instead — memory survives sessions; compaction doesn't.
- The task is short — compaction overhead adds a round-trip for no gain.
- You can restructure the harness to prevent the bloat in the first place (subagent the noisy searches, offload tool outputs to files).

## References

- Anthropic Engineering, "Effective context engineering for AI agents" — <https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents>
- Chroma Research, "Context Rot" — <https://research.trychroma.com/context-rot>
- Morph, "Context Rot: Why LLMs Degrade as Context Grows" — <https://www.morphllm.com/context-rot>
- Anthropic Engineering, "Effective harnesses for long-running agents" — <https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents>
- "Agentic Context Engineering: Evolving Contexts for Self-Improving Language Models", arXiv:2510.04618 — <https://arxiv.org/abs/2510.04618>
- Google ADK, "Context compression" — <https://google.github.io/adk-docs/context/compaction/>
