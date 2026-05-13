# 271 — Agent Isolation Patterns: sandbox + worktree + container + capability-based defense in depth

**Anchors.** OpenAI — *Codex sandbox* — sandboxed code execution environment. Anthropic — *Claude Code's Permission Modes* — https://code.claude.com/docs/en/permissions, sandboxed environments for `claude-code` execution. gVisor — https://gvisor.dev/ — kernel-level sandbox for containers. Firecracker — https://firecracker-microvm.github.io/ — micro-VMs for fast startup, used by AWS Lambda. POSIX capabilities + Linux namespaces. SELinux / AppArmor for Mandatory Access Control. Companions: [18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md), [02-subagent-delegation](02-subagent-delegation.md), [06-permission-modes](06-permission-modes.md), [07-permission-bridge-research-edition](../projects/polaris/docs/blocks/07-permission-bridge-research-edition.md), [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md), [269-prompt-injection-2026](269-prompt-injection-2026.md), [270-agent-supply-chain-security](270-agent-supply-chain-security.md), [273-agent-security-synthesis-2026](273-agent-security-synthesis-2026.md).

**One-line definition.** A 2026 framework for **isolating compromised agents** so blast radius is bounded — five layers of progressively stronger isolation each appropriate to a different threat tier: (1) **worktree isolation** (git worktree per agent run; filesystem changes contained to one branch); (2) **process isolation** (separate process per agent; OS-level memory + signal isolation); (3) **container isolation** (Docker / Podman; namespace + cgroup boundaries; LangGraph node-level containerization); (4) **micro-VM isolation** (Firecracker / Kata Containers; hardware virtualization for stronger boundaries with minimal overhead); (5) **capability-based isolation** (POSIX caps, SELinux/AppArmor, gVisor — **mandatory access control** at the kernel level so even root-equivalent code in the sandbox can't escape) — and the production rule is **match isolation tier to action consequence**: read-only research agent gets worktree; code-executing agent gets container; user-data-touching agent gets micro-VM; consequential action agent (file delete, money transfer) gets bright-line gate plus capability-based MAC.

## Why this paper matters (isolation is the last line of defense; without it, prompt injection becomes pwn)

Prompt injection ([269](269-prompt-injection-2026.md)) and supply-chain compromise ([270](270-agent-supply-chain-security.md)) will sometimes succeed despite the prevention layers. **Isolation is the engineering discipline that turns "the agent got compromised" from "the platform got compromised" into "this one worktree got compromised, recover and move on."** The blast radius of a compromised agent is exactly its isolation envelope; design that envelope deliberately, and the worst case is contained.

The 2026 isolation landscape has matured significantly. **Worktree isolation** ([18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md)) — each agent run in its own git worktree branched from `main` — is the cheapest first layer; filesystem modifications contained to one branch, easy to discard, no cross-agent contamination. **Process isolation** — a separate OS process per agent run with `pid` + `mount` namespaces — prevents memory access between runs and gives clean kill semantics. **Container isolation** — Docker / Podman with `seccomp` filters, dropped capabilities, read-only root, scoped network — is the production-default for code-executing agents and reaches Linux container security maturity (still kernel-shared, escape exploits exist but are rare). **Micro-VM isolation** — Firecracker (~125ms cold start, <5MB overhead per VM) or Kata Containers — provides hardware virtualization boundaries with the operational ergonomics of containers; AWS Lambda uses Firecracker for exactly this reason. **Capability-based isolation** — POSIX capabilities + SELinux / AppArmor / gVisor — adds Mandatory Access Control where even compromised root in the sandbox cannot exit; this is what high-security shops layer on top of containers or micro-VMs.

The production rule: **match isolation tier to action consequence**. A read-only research agent reading public web pages can run in a worktree (no consequential actions, low blast radius). A code-executing agent that runs Python on attacker-controlled input needs container isolation minimum, micro-VM ideal. A consequential-action agent that can send-email / transfer-money / modify-prod-config needs both bright-line gates ([14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md)) for human approval **and** capability-based MAC for the code itself. The rule is **each isolation layer constrains a strictly larger set of attacks at the cost of more operational complexity**; pick the tier that matches your threat model.

Take this seriously and three things change. **First**, you classify every agent's **action consequence** explicitly — what is the worst thing this agent can do if compromised? — and pick isolation tier accordingly. **Second**, you adopt **defense in depth** — worktree + container is better than worktree alone; container + capability-based MAC is better than container alone. **Third**, you **assume compromise** and design recovery — when isolation works, the compromised agent is killed, its worktree discarded, its memory partition flushed, the SRE runbook executes, and the platform continues; the difference between a compromise that becomes an incident and one that becomes a security catastrophe is exactly this design.

## Problem it solves (containing the blast radius of a compromised agent)

1. **Compromised agent has full filesystem access.** Without worktree isolation, agent modifies any file the user can.
2. **Agent runs share memory.** Without process isolation, malicious agent reads other runs' state.
3. **Code execution is unbounded.** Without container isolation, attacker-controlled code escapes.
4. **Container escape exploits exist.** Without micro-VM or MAC, kernel exploits cross container boundaries.
5. **Privileged operations unfiltered.** Without capability-based MAC, root in container = full host access on escape.
6. **Network egress unfiltered.** Without network policy, compromised agent exfiltrates data anywhere.
7. **Cross-agent contamination via shared memory.** Without partition isolation, poisoning propagates.
8. **Recovery time slow without isolation.** Without clean kill semantics, recovery requires manual cleanup.
9. **Audit trail unclear without process boundary.** Per-agent process boundaries map to per-agent audit logs.
10. **Resource exhaustion attacks.** Without cgroup limits, runaway agent consumes all CPU / memory / disk.

## Core idea in one paragraph

Agent isolation is **layered defense**: each layer constrains a strictly larger set of attacks at the cost of more operational complexity. **Layer 1 — worktree** ([18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md)): git worktree per agent run; `git worktree add ../run-abc123 -b run/abc123`; filesystem changes contained to one branch; cheap (~5GB disk per worktree, near-zero CPU); discard cheap (`git worktree remove`); essential first layer for any agent that modifies files. **Layer 2 — process isolation**: separate OS process per agent run with `pid` + `mount` namespaces; prevents memory access between runs; clean kill via `SIGKILL`; standard for agent runtimes. **Layer 3 — container isolation**: Docker / Podman with `seccomp` syscall filter, dropped POSIX capabilities (start with no capabilities, add only what's needed), read-only root filesystem, scoped network policy (egress allowlist), cgroup CPU + memory + disk limits; production-default for code-executing agents; ~125MB overhead per container, ~1s startup. **Layer 4 — micro-VM isolation**: Firecracker (AWS Lambda-shape) or Kata Containers; hardware virtualization boundary; ~5MB per VM, ~125ms cold start; **stronger isolation than containers at minimal overhead**; appropriate for high-trust-required execution like running attacker-controlled code or executing skills from low-trust marketplaces. **Layer 5 — capability-based isolation**: POSIX capabilities + SELinux / AppArmor type-enforced MAC + gVisor user-space kernel; **mandatory access control at kernel level**; even compromised root in the sandbox cannot escape; appropriate for the highest-consequence operations and adversarial-input execution. **The production rule** is: pick the lowest tier that matches your threat model, then **stack additional layers** for defense in depth. Match tier to action consequence; bright-line gate consequential operations regardless of isolation tier; assume compromise and design clean recovery via durability + rollback ([266-agent-durability-and-idempotency](266-agent-durability-and-idempotency.md), [267-agent-sre](267-agent-sre.md)).

## Mechanism (step by step)

### (a) Layer 1 — worktree isolation

```bash
# Per-agent run
git worktree add ../run-abc123 -b run/abc123 main
cd ../run-abc123
# Agent modifies files here; only this branch affected

# Cleanup
git worktree remove ../run-abc123
git branch -D run/abc123
```

Cost: ~5GB disk per worktree (full repo copy), near-zero CPU. Effectively free. **Essential first layer for any agent that modifies files.** Combined with [18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md) for subagent delegation.

### (b) Layer 2 — process isolation

```python
# Each agent run = separate subprocess
import subprocess
process = subprocess.Popen(
    ["python", "-m", "harness_core.runner", "--agent-config", config_path],
    cwd=worktree_path,
    env={"AGENT_RUN_ID": run_id, **scoped_env},
    preexec_fn=os.setsid,  # new session, clean kill
)
```

OS-level pid + mount namespaces; memory between processes is isolated. `SIGKILL` cleanly terminates.

### (c) Layer 3 — container isolation (Docker / Podman)

```yaml
# docker-compose.yml for agent run
services:
  agent-run:
    image: harness-core:1.2.3
    read_only: true                    # read-only root
    tmpfs:
      - /tmp:rw,size=100m              # writable scratch only
    cap_drop: [ALL]                    # drop all POSIX caps
    cap_add: []                        # explicit add (none here)
    security_opt:
      - "no-new-privileges:true"
      - "seccomp:./seccomp-profile.json"
    networks:
      - egress-allowlist               # custom network with egress rules
    volumes:
      - "${WORKTREE}:/workspace:rw"    # only the worktree is writable
    cpus: "0.5"
    mem_limit: "1G"
    pids_limit: 256
```

Custom seccomp profile blocks dangerous syscalls (e.g., `ptrace`, `mount`, `unshare`). `cap_drop: ALL` removes all POSIX capabilities; explicitly add only what's needed (typically none for code-execution sandbox). Read-only root prevents persistent compromise. Egress allowlist via network policy.

**Container escape risk** still exists (kernel exploits like CVE-2022-0185, CVE-2023-32629); not zero but rare and patched quickly. Adequate for most production agents.

### (d) Layer 4 — micro-VM isolation (Firecracker / Kata)

```bash
# Firecracker example: micro-VM per agent run
firectl --kernel-image=vmlinux \
        --root-drive=rootfs.img \
        --memory=512 \
        --cpu=2 \
        --network-interface=tap0 \
        --jailer-uid=$AGENT_UID \
        --jailer-gid=$AGENT_GID
```

Hardware-level virtualization boundary (KVM); kernel exploits in the guest cannot reach the host kernel. ~125ms cold start, ~5MB overhead per VM. AWS Lambda uses this for exactly this reason — 10s of millions of invocations per second with hard isolation.

**Trade-offs**: more operational complexity than containers; networking + storage configuration heavier; image format differs.

### (e) Layer 5 — capability-based isolation (SELinux / AppArmor / gVisor)

**SELinux example** — type-enforced MAC:

```
# /etc/selinux/contexts/agent.te
type agent_t;
type agent_exec_t;
type agent_data_t;

# Agent can only read/write its own data
allow agent_t agent_data_t:file { read write };
# Cannot exec other binaries
neverallow agent_t * : file execute;
```

**gVisor example** — user-space kernel intercepting syscalls:

```bash
docker run --runtime=runsc \
           --security-opt seccomp=./seccomp.json \
           harness-core:agent-runner
```

gVisor runs the container under its own Sentry kernel; syscalls are intercepted in user-space, so even if a guest exploit lands, host kernel is unaffected. ~10-30% performance overhead vs native Docker.

### (f) Network egress control

Even within isolation, network policy is critical:

```yaml
# Network policy
egress_allowlist:
  - "api.anthropic.com:443"
  - "api.openai.com:443"
  - "*.googleapis.com:443"
  - "registry.npmjs.org:443"
  - "pypi.org:443"
  - "github.com:443"
deny_all_other: true
```

Implemented via Docker user-defined networks, Kubernetes NetworkPolicy, or eBPF-based egress filters (Cilium).

### (g) Filesystem allowlists

Beyond worktree, restrict reads:

```yaml
filesystem_allowlist:
  read:
    - /workspace
    - /usr/local/lib/python3.12
    - /opt/harness_core/skills/vendored
  write:
    - /workspace
    - /tmp
```

Implemented via mount namespaces, container `volumes` config, or bind-mount whitelist.

### (h) Resource limits (cgroups)

```
cpu_limit: 2 cores
memory_limit: 4 GB
disk_io_limit: 100 MB/s
process_count: 64
open_files: 1024
network_bandwidth: 10 Mbps
```

Prevents resource-exhaustion attacks (fork bombs, memory hogs, disk fillers, network floods).

### (i) Bright-line gate composition

Even with strong isolation, consequential actions ([14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md)) — sending email, transferring money, deleting files, modifying prod config — require human approval. Bright-line gates are **orthogonal to isolation** — both layers needed.

```
[Compromised agent] → [container isolation: blocked from arbitrary fs writes]
                    → [bright-line gate: cannot send-email-to-external without user approval]
```

### (j) Recovery: clean kill + worktree discard + memory partition flush

When isolation works, recovery is fast:

```bash
# Detect compromise (anomaly alert from observability)
harness_core agents kill --run-id abc123  # SIGKILL the process tree
harness_core worktrees discard --run-id abc123  # remove worktree
harness_core memory flush --partition agent-abc123  # remove poisoned writes
harness_core sbom audit --since "2026-05-09T10:00:00Z"  # check supply chain
# Run postmortem
```

Clean recovery in seconds; the platform continues unaffected.

## Empirical results (table — May 2026 isolation benchmarks)

| Layer | Cold start | Overhead per instance | Isolation strength |
|---|---:|---:|---|
| Worktree | <100ms | ~5GB disk | Filesystem only |
| Process | <50ms | ~10MB memory | OS-level memory + signals |
| Container (Docker) | ~1s | ~125MB | Linux namespaces + cgroups + seccomp |
| Micro-VM (Firecracker) | ~125ms | ~5MB + ~50MB kernel | Hardware virtualization |
| Container + gVisor | ~1.5s | ~150MB | User-space kernel + namespaces |
| Container + SELinux/AppArmor | ~1s | ~125MB + policy CPU | Container + Mandatory Access Control |

| Threat | Worktree | Process | Container | Micro-VM | Capability-MAC |
|---|---|---|---|---|---|
| Filesystem write outside scope | ✅ | ✅ | ✅ | ✅ | ✅ |
| Read other runs' memory | ❌ | ✅ | ✅ | ✅ | ✅ |
| Privilege escalation in OS | ❌ | ❌ | ⚠ (some bypass) | ✅ | ✅ |
| Container escape (kernel exploit) | ❌ | ❌ | ❌ | ✅ | ✅ |
| Network egress unbounded | ❌ | ❌ | ✅ (with policy) | ✅ | ✅ |
| Resource exhaustion | ❌ | ❌ | ✅ (cgroups) | ✅ | ✅ |
| Side-channel attack on host kernel | ❌ | ❌ | ❌ | ⚠ | ⚠ |

## Variants and ablations

- **Layered combination.** Worktree + container + capability-MAC is the production-grade combination.
- **Per-tool isolation.** Different MCP servers in different containers per call.
- **gVisor for code-execution sandboxes.** Lower overhead than full micro-VM, stronger than Docker alone.
- **Firecracker per A2A request.** AWS Lambda-shape architecture for inbound A2A calls.
- **WebAssembly sandboxes** (Wasmtime, Wasmer) for tool-call execution; strong isolation, fast startup.
- **Linux user namespaces** for rootless containers.
- **bubblewrap / `bwrap`** for lightweight Linux sandboxing.
- **Subprocess + chroot + drop-privileges** for minimal Python sandboxes.
- **Firejail / Bubblewrap GUI sandboxing** for desktop-agent contexts.

## Failure modes and limitations

- **Container escape via kernel exploit.** Patches lag; assume residual risk.
- **Side-channel attacks.** Spectre / Meltdown class; mitigated at hardware/OS level.
- **Resource starvation of host.** Without cgroups, runaway agent affects neighbors.
- **MAC policy churn.** SELinux/AppArmor policies require maintenance.
- **Networking escape via DNS.** Egress allowlist must include DNS resolution; abuse possible.
- **Time-based exfiltration.** Compromised agent leaks via timing side-channels.
- **Persistent state leakage.** Even with read-only root, persistent volumes (memory store) can be vector.
- **Cross-tenant attacks** in multi-tenant SaaS.
- **Operational cost** at scale (10k containers vs 10k worktrees).
- **Compatibility issues** — some tools require capabilities you can't drop.
- **gVisor performance overhead** (~10-30%) unacceptable for some workloads.

## When to use, when not

**Adopt layered isolation** for any production agent — minimum: worktree + process. Code-executing agents add container; consequential-action agents add micro-VM or MAC. The strongest cases are **paying-customer SaaS agents** (multi-tenant), **regulated agents** (audit + isolation), **code-executing agents** (untrusted input), and **agent-platform-as-a-service** (Vertex Agent Engine, Bedrock Agents, custom).

**Skip strong isolation** for fully-trusted local prototypes, single-user offline tools, or read-only research agents on public data. **Never skip** worktree isolation if the agent modifies files — it's nearly free and catches the most common case.

## Implications for harness engineering

- **Match isolation tier to action consequence.** [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md), [07-permission-bridge-research-edition](../projects/polaris/docs/blocks/07-permission-bridge-research-edition.md) — bright-line action kinds gate isolation tier.
- **Worktree as default.** [18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md), [02-subagent-delegation](02-subagent-delegation.md) — every agent run gets one.
- **Container-runtime choice in `harness_core/runtime/`.** Docker / Podman / gVisor / Firecracker; pluggable.
- **Network egress allowlist policy.** Per-agent or per-routine; default-deny.
- **Filesystem allowlist via mount.** Read scopes match the agent's task.
- **Resource limits via cgroups.** Per-agent CPU + memory + disk + processes + network.
- **Recovery primitives.** [267-agent-sre](267-agent-sre.md) — kill + discard + flush procedures.
- **Audit logging per process.** [264-agent-observability-stack-2026](264-agent-observability-stack-2026.md), [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md), [24-observability-tracing](24-observability-tracing.md) — process boundary = audit boundary.
- **Bright-line gates orthogonal.** Isolation contains; bright-line halts.
- **MCP server isolation.** [256-mcp-2025-2026-evolution](256-mcp-2025-2026-evolution.md), [17-mcp-adapter](../projects/polaris/docs/blocks/17-mcp-adapter.md) — per-server containers.
- **Memory partition isolation.** [233-memory-scaling-for-agents](233-memory-scaling-for-agents.md) — untrusted memory tier separate.
- **Cross-agent A2A isolation.** [254-a2a-protocol-deep-dive](254-a2a-protocol-deep-dive.md), [251-multi-agent-teams-2026-synthesis](251-multi-agent-teams-2026-synthesis.md) — peer agents in separate containers.
- **Skill-execution isolation.** [04-skills](04-skills.md), [09-skill-engine](../projects/polaris/docs/blocks/09-skill-engine.md), [257-agent-skill-marketplace-landscape](257-agent-skill-marketplace-landscape.md) — installed skills run in container.
- **Trust-tier-driven tier selection.** [255-agntcy-oasf-acp-deep-dive](255-agntcy-oasf-acp-deep-dive.md), [270-agent-supply-chain-security](270-agent-supply-chain-security.md) — lower-trust artifact = stronger isolation.
- **Trajectory simulation in isolated sandbox.** [142-trajectory-simulation-agents](142-trajectory-simulation-agents.md), [49-agents-of-chaos-red-teaming](49-agents-of-chaos-red-teaming.md) — chaos exercises run in isolation.

**One-line takeaway for harness designers.** **Agent isolation is layered defense — worktree + process + container + micro-VM + capability-based MAC, with each layer constraining strictly more attacks at the cost of operational complexity; match isolation tier to action consequence (read-only research = worktree, code-executing = container, consequential = micro-VM + MAC), stack layers for defense in depth, assume compromise will sometimes occur, and design recovery via clean kill + worktree discard + memory partition flush; the difference between a compromise that becomes a contained incident and one that becomes a security catastrophe is exactly this design.**
