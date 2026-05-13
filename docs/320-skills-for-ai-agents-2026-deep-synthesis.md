# 320 - Skills for AI Agents 2026: Deep Synthesis

**Scope.** A single dense synthesis of skill-relevant AI-agent papers and systems, grounded in the supplied OpenReview/arXiv links and `references/Skills for AI Agents - A Capstone-Successor Synthesis (R7) .md`. It connects classical RL skill discovery, LLM procedural skill libraries, SKILL.md / MCP productized skills, memory-as-procedure, and skill safety into one research map.

**Thesis.** "Skill" is now the unifying primitive of the AI-agent stack. The field's older definitions have converged:

1. **RL skill:** a latent-conditioned option/policy discovered from interaction.
2. **LLM skill:** a reusable natural-language, code, workflow, or script artifact retrieved and invoked by an agent.
3. **Product skill:** a `SKILL.md` folder, MCP server, or marketplace package with metadata, interface, and trust boundary.

The modern skill is best treated as a **callable, persistent capability with an interface, verifier, edit history, and lineage**. The next strong paper should evaluate the whole lifecycle, not just one-shot skill use.

---

## 0. Reading Map

If you only have one hour:

1. **Definition:** read the 7-tuple skill formalism below.
2. **LLM skill evidence:** ASI, PolySkill, SkillEvo, SKILLRL, SkillsBench.
3. **RL skill lineage:** DIAYN, METRA, DSR, DCSL, HMASD, HiSSD, VO-MASD.
4. **Product layer:** Anthropic Skills, MCP, AGENTS.md, skills marketplaces.
5. **Safety:** Agent Skills in the Wild, A-MemGuard, MCPoison/mcp-remote-style risks.
6. **Next-paper idea:** SkillNet-R: lifecycle + adversarial + cross-domain skill benchmark.

---

## 1. Unified Skill Definition

The minimal 2026 definition:

> A skill is a parameterizable, callable, named, persistent capability with an interface and an evaluation protocol.

The stronger research definition is a 7-tuple:

| Field | Meaning | Examples |
|---|---|---|
| **Applicability** | when the skill should be considered | SKILL.md description, option initiation set, retrieval predicate |
| **Policy** | how the skill acts | Python code, LLM prompt, neural policy, MCP handler |
| **Termination** | when the skill finishes | option termination, return value, tool response, verifier pass |
| **Interface** | invocation contract | function signature, JSON schema, MCP input schema |
| **Edit operator** | how it changes | add, refine, merge, split, prune, distill, compose, rerank |
| **Verification** | admission / success test | unit test, environment reward, LLM judge, human review |
| **Lineage** | provenance graph | source trajectory, parent skill, evolution round, author |

### Why This Matters

Earlier work measured **skill count**. Modern systems must measure **skill lifecycle quality**: admission rate, repair rate, reuse, transfer, safety, provenance, and negative-transfer frequency.

---

## 2. Timeline: From Options to SKILL.md

```text
1999  Options framework: initiation, policy, termination
2019  DIAYN: unsupervised diverse skill discovery
2020  DADS / SMERL: dynamics-aware and reward-compatible skills
2023  Voyager: LLM writes executable code skills in Minecraft
2023  HMASD: team + individual skill discovery for MARL
2024  AWM: induced workflow memory for agents
2024  METRA / DCSL / HiSSD: modern RL skill discovery wave
2024  MCP launches as tool protocol
2025  ASI: programmatic skills beat text skills on WebArena
2025  PolySkill: reusable polymorphic web skills
2025  Alita: agents create tools/skills and register capabilities
2025  Anthropic Skills: SKILL.md productizes progressive disclosure
2026  SkillsBench / SkillFlow / SkillX / SKILLRL / SkillEvo: lifecycle and benchmark wave
2026  Dynamic skills surveys formalize 7-tuple + lifecycle operators
```

---

## 3. Skill Taxonomy

| Axis | Values | Canonical examples | Design implication |
|---|---|---|---|
| **Representation** | natural language, code, neural policy, MCP server, memory record | AWM, ASI, Voyager, DIAYN, MCP servers | interface determines verifier |
| **Learning signal** | curated, self-reflection, imitation, RL, quality-diversity, co-evolution | Claude Skills, Voyager, DSR, SKILLRL, CycleQD, EvoSkills | skill admission must match signal quality |
| **Granularity** | atomic action, subroutine, workflow, domain package | click, search-product, WebArena workflow, healthcare skill pack | retrieval gets harder as granularity grows |
| **Lifecycle** | add, refine, merge, split, prune, distill, compose, rewrite, rerank | SkillX, SkillEvo, TMLR dynamic skills survey | static skill benchmarks are incomplete |
| **Domain** | web, GUI, code, embodied, research, support, memory | PolySkill, Agent-S, Voyager, Agent Laboratory, tau-bench, MemSkill | skill format should follow environment |
| **Trust boundary** | first-party, marketplace, generated, adversarial, cross-agent | Anthropic Skills, Smithery, self-generated skills, A-MemGuard | skills are supply-chain artifacts |

---

## 4. Master Comparison Table - Supplied Papers

| Work | Status / venue | Skill type | Core contribution | Evidence / benchmark | Relevance | Main limitation |
|---|---|---|---|---|---|---|
| [CycleQD](https://openreview.net/forum?id=Kvdh12wGC0) | ICLR 2025 Poster | LLM fine-tuning / quality-diversity | cyclic quality-diversity for multi-task agent skill acquisition via model merging and SVD mutation | AgentBench: Llama-3-8B improved toward GPT-3.5-level on coding/OS/DB tasks | bridges QD/evolution with agent skills | compute and stability cost |
| [SKILLRL](https://openreview.net/forum?id=FYc2IygegR) | LLA 2026 Poster | LLM skill bank + RL | recursive skill-augmented RL with distilled general/task-specific skills | ALFWorld and WebShop gains over GRPO-style baselines | direct skill-memory-RL bridge | skill noise and simulator dependence |
| [HMASD](https://openreview.net/forum?id=xMgO04HDOS) | NeurIPS 2023 Poster | multi-agent RL options | discovers team and individual skills in MARL | sparse-reward MARL benchmarks | multi-agent skill abstraction | not LLM skill artifacts |
| [Embodied HITL code skills](https://openreview.net/forum?id=1su9RkTVT9) | ICLR 2026 submitted | embodied LLM code skills | human feedback becomes reusable manipulation code skills | Ravens, Franka Kitchen, MetaWorld, real robot; fewer feedback rounds | robot skill libraries | human feedback scalability |
| [ASI](https://openreview.net/forum?id=lsAY6fWsog) | COLM 2025 | programmatic LLM skills | induces verified code skills for web tasks | WebArena: programmatic skills beat static/text-skill baselines | strongest programmatic-skill evidence | web-focused |
| [PolySkill](https://openreview.net/forum?id=KdEsujyiSV) | ICLR 2026 Poster | polymorphic web skills | decouples abstract goal from site-specific implementations | Mind2Web seen/unseen gains and step reduction | reusable skills across websites | exploration/evaluator quality |
| [Dynamic Agent Skills survey](https://openreview.net/forum?id=cjU3YbcRr8) | TMLR under review | survey / formalism | 7-tuple skill and 10-operator library algebra over dynamic skill systems | 94-paper audit | taxonomy spine | survey, no new system |
| [SkillEvo](https://openreview.net/forum?id=S1cIE9pe3k) | ICLR 2026 submitted | evolving web skills | WebGRPO + SkillGenesis + skill path graph | WebArena-Lite large author-reported gains | end-to-end skill evolution | submitted; needs reproduction |
| [DCSL](https://openreview.net/forum?id=8egnwady4b) | ICLR 2025 Poster | RL skill learning | dynamic contrastive skill learning over state transitions | completion/efficiency gains in RL domains | semantic skill grouping | not LLM-oriented |
| [DSR](https://openreview.net/forum?id=PklYedVFUW) | ICLR 2025 submitted | hierarchical RL skill refinement | mitigates temporal abstraction shift with dynamical skill refinement | sparse-reward skill-from-demo tasks | theory for skill drift/refinement | unclear acceptance / assumptions |
| [HiSSD](https://openreview.net/forum?id=HR1ujVR0ig) | ICLR 2025 Poster | offline multi-agent skill discovery | separates common and task-specific cooperative skills | multi-agent MuJoCo and SMAC transfer | reusable multi-agent skill sets | offline coverage limits |
| [Shachi](https://openreview.net/forum?id=30e3LnZzmI) | ICLR 2026 submitted | agent-based modeling skills | modular LLM ABM with config/memory/tools | 10-task ABM benchmark, social-simulation case | adjacent agent engineering | not procedural skill induction |
| [MemSkill](https://openreview.net/forum?id=qEYqr6eA0I) / [arXiv](https://arxiv.org/abs/2602.02474) | CoRR / arXiv 2026 | memory skills | treats memory operations as learnable skills in a bank | LoCoMo, LongMemEval, HotpotQA, ALFWorld | explicit memory-skill equivalence | extra LLM/RL/evolution complexity |
| [A-MemGuard](https://openreview.net/forum?id=fVxfCEv8xG) | ICLR 2026 submitted | skill/memory safety | detects context-triggered malicious memory via consensus + dual memory | attack success reduction with small utility loss | safety boundary for skill stores | latency and adaptive attacks |
| [GraphMind](https://openreview.net/forum?id=XromAiEaE3) | ICLR 2026 withdrawn | graph memory / planning | dynamic KG memory for partially observable planning | navigation-style tasks | adjacent structured state | withdrawn |
| [VO-MASD](https://arxiv.org/abs/2405.16386) | arXiv / IJCAI 2025 | offline multi-agent skill discovery | subgroup and temporal abstraction via variational codebooks | SMAC/SMACv2 gains over baselines | multi-agent option discovery | not LLM skill libraries |

### Dedupe Notes

- `qEYqr6eA0I` and `arxiv:2602.02474` are the same MemSkill paper.
- `PklYedVFUW` and `xMgO04HDOS` were repeated in the prompt but are unique papers.
- `XromAiEaE3` / GraphMind is withdrawn and should be cited cautiously.

---

## 5. Adjacent Must-Cite Systems

| Work / repo | Skill type | Why it belongs |
|---|---|---|
| [Voyager](https://github.com/MineDojo/Voyager) | executable code skill library | first iconic LLM agent that grows a reusable skill library |
| [Anthropic Skills](https://github.com/anthropics/skills) | `SKILL.md` folders | productized progressive disclosure for skills |
| [MCP servers](https://github.com/modelcontextprotocol/servers) | executable tool skills | practical protocol-layer skill library |
| [MCP specification](https://github.com/modelcontextprotocol/specification) | capability ABI | standard interface for executable skills |
| [smolagents](https://github.com/huggingface/smolagents) | code-first tools/agents | lightweight CodeAgent and MCP bridge |
| [Alita](https://github.com/CharlesQ9/Alita) | self-generated tool/MCP capabilities | agent writes and registers capabilities |
| [SkillsBench](https://github.com/benchflow-ai/SkillsBench) | skill benchmark | curated vs self-generated skill evaluation |
| [SkillFlow](https://github.com/ZhangZi-a/SkillFlow) | lifelong skill discovery/evolution benchmark | tests skill discovery over time |
| [SkillX](https://github.com/zjunlp/SkillX) | hierarchical skill knowledge base | planning/functional/atomic skill KBs |
| [CoEvoSkills](https://github.com/Zhang-Henry/CoEvoSkills) | co-evolutionary skill verifier | skill generator-verifier loop |
| [EvoSkill](https://github.com/sentient-agi/EvoSkill) | coding-agent skill synthesis from failures | mines failed trajectories into skills |
| [Mem0](https://github.com/mem0ai/mem0) | memory substrate | procedural memory can behave like skills |
| [Letta](https://github.com/letta-ai/letta) | stateful memory agents | long-horizon skill/memory blocks |
| [ReasoningBank](https://github.com/google-research/reasoning-bank) | reusable reasoning strategies | skills as distilled reasoning procedures |
| [Smithery](https://smithery.ai/) | MCP marketplace | skill/server discovery |
| [LobeHub](https://github.com/lobehub/lobehub) | agent/MCP hub | distribution surface for skills |
| [everything-claude-code](https://github.com/affaan-m/everything-claude-code) | skills/hooks/rules corpus | practical harness-skill bundle |
| [METRA](https://github.com/seohongpark/metra) | unsupervised RL skills | modern DIAYN-line skill discovery |
| [openpi](https://github.com/Physical-Intelligence/openpi) | VLA policy skills | embodied skill policies |

---

## 6. Evidence Core

### 6.1 Curated Skills Beat Self-Generated Skills

SkillsBench-style evidence says curated skills can give large pass-rate lifts, while naive self-generated skills often do not help on average. The design lesson is not "let the agent write as many skills as possible." It is "gate skill admission with strong verifiers."

### 6.2 Programmatic Skills Beat Pure Text Skills

ASI and PolySkill show why code matters: executable skills can be tested, composed, and reused with stronger guarantees than natural-language instructions. Text is good for applicability and intent; code is better for verified behavior.

### 6.3 Skills Are Procedural Memory

ReasoningBank, MemSkill, SKILLRL, Letta, and Mem0 all blur the line between "remembered lesson" and "callable procedure." Procedural memory becomes a skill once it has an interface and verifier.

### 6.4 Skill Safety Is Supply-Chain Safety

Skills can execute code, call MCP servers, read files, write data, and modify future prompts. They are not passive documents. Agent Skills in the Wild, A-MemGuard, MCPoison-style incidents, and mcp-remote-style risks show that skills require provenance, sandboxing, signing, and regression tests.

---

## 7. Skill Lifecycle Operators

| Operator | Purpose | Example |
|---|---|---|
| **ADD** | create a new skill from a trajectory, user artifact, or package | Voyager adds a code function |
| **REFINE** | improve skill based on failure | EvoSkill repairs coding skill |
| **MERGE** | combine redundant skills | SkillEvo graph consolidation |
| **SPLIT** | divide broad skill into narrower skills | web checkout -> search/cart/payment |
| **PRUNE** | remove stale or unsafe skill | marketplace revocation |
| **DISTILL** | compress trajectory into skill | ReasoningBank, SKILLRL |
| **ABSTRACT** | lift concrete procedure into general template | PolySkill goal abstraction |
| **COMPOSE** | chain skills into workflow | AWM workflow memory |
| **REWRITE** | change interface/implementation | MCP schema migration |
| **RERANK** | change retrieval priority | SkillFlow retrieval |

---

## 8. Benchmarks and What They Miss

| Benchmark | Measures | What it misses |
|---|---|---|
| SkillsBench | curated/self-generated skill benefit over tasks/domains | lifecycle, adversarial, lineage |
| SkillFlow | retrieval and lifelong skill discovery | safety and verifier robustness |
| SkillLearnBench | continual skill generation | marketplace and adversarial pressure |
| WebArena / WebArena-Lite | web-agent skills | long-term library drift |
| Mind2Web | website generalization | dynamic skill lifecycle |
| ALFWorld / WebShop | embodied/text commerce skill use | real-world tool/API evolution |
| BFCL | function calling | procedural memory transfer |
| MemoryAgentBench | memory competencies | interface/verifier/lineage fields |
| LongMemEval / LoCoMo | long-term memory | callable procedural skills |
| AgentDojo | tool-use security | self-evolving skill stores |

The benchmark gap is clear: we need a skill benchmark that tracks **library trajectories**, not only final accuracy.

---

## 9. Safety and Governance

| Risk | How it appears in skill systems | Mitigation |
|---|---|---|
| prompt injection | malicious instructions inside skill docs or retrieved references | instruction/data separation, static scan, runtime policy |
| data exfiltration | skill script reads secrets or sends files | sandbox, allowlist, egress control |
| privilege escalation | skill invokes high-risk tools/MCP servers | scoped permissions, human approval gates |
| supply-chain compromise | marketplace skill or MCP server changes after trust | signing, pinning, SBOM, revocation |
| dormant skill trigger | skill activates only in a specific context | consensus validation, context-trigger tests |
| skill poisoning | bad trajectory distilled into reusable procedure | verifier-gated admission, lineage review |
| negative transfer | near-match skill harms new task | conservative retrieval, task-fit scoring |

---

## 10. Proposed Next Paper: SkillNet-R

**Title candidate:** *SkillNet-R: A Lifecycle and Adversarial Benchmark for Agent Skills*

**Core claim:** A skill system should be evaluated over its full lifecycle: add, retrieve, execute, verify, refine, compose, prune, and defend. One-shot "with skill vs no skill" evaluations miss the hard part.

### System Under Test

- Skill artifacts: `SKILL.md`, code, MCP server, memory procedure.
- Domains: web, coding, tool/API, memory, embodied-lite.
- Skills stores: flat vector, graph, hierarchical, curated, self-generated.
- Verifiers: unit tests, environment rewards, LLM judge, human-authored rubric.

### Metrics

| Metric | Meaning |
|---|---|
| task success | final task pass rate |
| skill admission precision | admitted skills that later help |
| skill admission recall | useful skills not admitted |
| reuse rate | how often skills are invoked |
| negative-transfer rate | invoked skills that hurt |
| repair success | failed skill -> useful refined skill |
| lineage integrity | can we trace skill source and edits |
| adversarial robustness | success under malicious skill/reference/tool injection |
| cost | tokens, tool calls, latency, verifier calls |

### Baselines

1. no skills,
2. curated static skills,
3. self-generated ungated skills,
4. vector-retrieved skills,
5. graph skills,
6. verifier-gated skills,
7. adversarially hardened skills.

### Why This Is Better Than Another Skills Survey

It directly tests the missing fields in the 7-tuple: interface, edit, verification, and lineage. It also bridges memory, self-evolution, context engineering, MCP/tool use, and safety.

---

## 11. Practical Reading Order

1. **Start with foundations:** Options, DIAYN, METRA.
2. **Then LLM skill libraries:** Voyager, AWM, ASI, PolySkill.
3. **Then self-evolving skills:** EvoSkill, EvoSkills, SkillEvo, SKILLRL, MemSkill.
4. **Then product skills:** Anthropic Skills, MCP servers, AGENTS.md, skills marketplaces.
5. **Then benchmarks:** SkillsBench, SkillFlow, SkillLearnBench, BFCL, WebArena, MemoryAgentBench.
6. **Then safety:** Agent Skills in the Wild, A-MemGuard, MCP security incidents, AgentDojo.
7. **Then next-paper design:** SkillNet-R.

---

## 12. Sources

- [CycleQD](https://openreview.net/forum?id=Kvdh12wGC0)
- [SKILLRL](https://openreview.net/forum?id=FYc2IygegR)
- [HMASD](https://openreview.net/forum?id=xMgO04HDOS)
- [Embodied HITL code skills](https://openreview.net/forum?id=1su9RkTVT9)
- [ASI / Inducing Programmatic Skills](https://openreview.net/forum?id=lsAY6fWsog)
- [PolySkill](https://openreview.net/forum?id=KdEsujyiSV)
- [Dynamic Agent Skills survey](https://openreview.net/forum?id=cjU3YbcRr8)
- [SkillEvo](https://openreview.net/forum?id=S1cIE9pe3k)
- [DCSL](https://openreview.net/forum?id=8egnwady4b)
- [DSR](https://openreview.net/forum?id=PklYedVFUW)
- [HiSSD](https://openreview.net/forum?id=HR1ujVR0ig)
- [Shachi](https://openreview.net/forum?id=30e3LnZzmI)
- [MemSkill OpenReview](https://openreview.net/forum?id=qEYqr6eA0I)
- [MemSkill arXiv](https://arxiv.org/abs/2602.02474)
- [A-MemGuard](https://openreview.net/forum?id=fVxfCEv8xG)
- [GraphMind](https://openreview.net/forum?id=XromAiEaE3)
- [VO-MASD](https://arxiv.org/abs/2405.16386)
- [Voyager](https://github.com/MineDojo/Voyager)
- [Anthropic Skills](https://github.com/anthropics/skills)
- [MCP specification](https://github.com/modelcontextprotocol/specification)
- [MCP servers](https://github.com/modelcontextprotocol/servers)
- [smolagents](https://github.com/huggingface/smolagents)
- [Alita](https://github.com/CharlesQ9/Alita)
- [SkillsBench](https://github.com/benchflow-ai/SkillsBench)
- [SkillFlow](https://github.com/ZhangZi-a/SkillFlow)
- [SkillX](https://github.com/zjunlp/SkillX)
- [CoEvoSkills](https://github.com/Zhang-Henry/CoEvoSkills)
- [EvoSkill](https://github.com/sentient-agi/EvoSkill)
- [Mem0](https://github.com/mem0ai/mem0)
- [Letta](https://github.com/letta-ai/letta)
- [ReasoningBank](https://github.com/google-research/reasoning-bank)
- [Smithery](https://smithery.ai/)
- [LobeHub](https://github.com/lobehub/lobehub)
- [everything-claude-code](https://github.com/affaan-m/everything-claude-code)
- [METRA](https://github.com/seohongpark/metra)
- [openpi](https://github.com/Physical-Intelligence/openpi)

---

## 13. Final Takeaways

1. **Skills are procedural memory with an interface.**
2. **Verification matters more than skill count.**
3. **Programmatic skills are easier to verify than natural-language skills.**
4. **Curated skills help; naive self-generated skills often do not.**
5. **Skill systems are supply-chain systems.**
6. **The missing benchmark is lifecycle-aware and adversarial.**
7. **The strongest next paper is SkillNet-R: evaluate add/retrieve/execute/verify/refine/prune under distribution shift and attack.**
