# 35 — Malicious Intermediary Attacks on the LLM Supply Chain

**Definition.** "Your Agent Is Mine" (arXiv:2604.08407) documents how compromised or malicious **third-party API routers** — application-layer proxies between an agent and its upstream model — can intercept, modify, or exfiltrate every in-flight JSON payload. Four attack classes are catalogued, and evaluation across 428 routers finds evidence of active injection attacks, credential theft, and autonomous session exploitation in the wild.

## Problem it solves

Most LLM security discussion focuses on the prompt (jailbreaks) and the model (alignment). The supply chain between the agent and the model is a much bigger, less-watched attack surface. Teams routinely adopt "OpenAI-compatible" routers for cost, failover, and multi-model routing — often without considering that the router has plaintext access to every prompt (including tool schemas, injected RAG content, API keys embedded in system prompts) and every response (including generated tool calls that can be modified before delivery to the agent).

This paper formalizes the threat, introduces attack classes, and proposes a defensive proxy (Mine) with pragmatic mitigations.

## Mechanism — four attack classes

1. **AC-1: Payload Injection.** The router rewrites the in-flight prompt to inject malicious instructions the user never sent. Because the agent cannot distinguish router-inserted text from its own system prompt, the injection inherits the agent's authority.
2. **AC-1.a: Dependency-Targeted Injection.** A selective variant — only inject when the request matches certain triggers (target user, target tool, target dependency). Evades broad monitoring.
3. **AC-1.b: Conditional Delivery.** Inject only under runtime conditions (time of day, conversation state, presence of a specific token). Same evasion intent, different trigger.
4. **AC-2: Secret Exfiltration.** Harvest credentials, API keys, PII, and proprietary prompts from the plaintext stream. Stored to router-side logs or siphoned to external endpoints.

The threat is possible because routers commonly lack cryptographic integrity or confidentiality between client and upstream model. Even TLS to the router does nothing if the router itself is the adversary.

## Concrete pattern — the defense stack (Mine-style)

The paper's proposed research proxy suggests a layered defense any production team can adopt:

```
1. Fail-closed policy gates
   - Whitelist of allowed model endpoints.
   - Deny unknown / upgraded routers.
   - Strict request schema validation; refuse unusual fields.

2. Response-side anomaly screening
   - Detect unexpected tool_calls (unknown tool name, anomalous args).
   - Detect content-type drift in structured outputs.
   - Rate-based anomaly alerts (spike in tool calls, unusual arg length).

3. Append-only transparency log
   - Every outbound request + inbound response hashed and logged.
   - Periodically reconcile hashes with expected upstream model behavior.

4. Upstream attestation (where available)
   - Prefer direct model-vendor endpoints over routers.
   - Prefer vendors that offer request signing / attested responses.
```

For teams that must use routers (cost, vendor lock-in concerns), adopt all four layers plus narrow scope — do not send secrets in prompts, do not embed auth tokens, rotate often.

## Variants & related techniques

- **Prompt-injection defense** ([22-guardrails-prompt-injection.md](22-guardrails-prompt-injection.md)) — the endpoint-side analogue; does not help against in-transit routers.
- **Permission modes** ([06-permission-modes.md](06-permission-modes.md)) — even if a router injects a tool call, tight permissions can refuse it.
- **Observability / tracing** ([24-observability-tracing.md](24-observability-tracing.md)) — the append-only log is an extension of trace logging with cryptographic integrity.
- **Human-in-the-loop** ([23-human-in-the-loop.md](23-human-in-the-loop.md)) — HITL on high-stakes tool calls stops injection outcomes even if injections succeed.
- **Agents of Chaos** ([49-agents-of-chaos-red-teaming.md](49-agents-of-chaos-red-teaming.md)) — demonstrates agent damage under non-router adversaries; complementary threat model.

## Failure modes & anti-patterns

- **"TLS is enough."** TLS protects against third parties on the wire, not the endpoint you deliberately send to. The router *is* the endpoint.
- **Secrets in prompts.** The most common enabler of AC-2. Move secrets out of prompts into tool binding, inject per-call with short-lived tokens.
- **Unvetted router upgrades.** A benign router is acquired, or pushes a compromised update. Fix: pin versions; alert on behavior drift.
- **Trust-by-reputation.** "Big-name" routers have been compromised in the past. Fix: verify via transparency logs and anomaly screening regardless of reputation.
- **Blind retries.** Retry on failure through another router without correlating responses — a silent channel for AC-1.b attacks to succeed intermittently.
- **Ignoring tool-call schema drift.** An agent accepting any `tool_call` the router sends back will execute injected tool calls. Fix: validate every tool call against your local registry; refuse unknown.

## When to use these defenses

Apply **always** if:

- You use any third-party LLM router.
- Your agent has write-capable tools.
- You handle any user data, credentials, or proprietary prompts.

The "when not to" list is essentially empty — even for local-model-only deployments, the supply-chain reasoning applies to MCP servers, plugin marketplaces, and any binary in the tool chain.

## References

- arXiv:2604.08407 — "Your Agent Is Mine: Measuring Malicious Intermediary Attacks on the LLM Supply Chain". <https://arxiv.org/abs/2604.08407>
- OWASP, "LLM Top 10" — 2024/2025 updates. <https://owasp.org/www-project-top-10-for-large-language-model-applications/>
- Simon Willison, "Prompt injection" archive. <https://simonwillison.net/tags/prompt-injection/>
- Agents of Chaos (arXiv:2602.20021). <https://arxiv.org/abs/2602.20021>
- OpenAI, "Instruction Hierarchy" (arXiv:2404.13208). <https://arxiv.org/abs/2404.13208>
