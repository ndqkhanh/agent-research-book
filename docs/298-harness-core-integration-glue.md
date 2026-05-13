# 298 — `harness_core/` Integration Glue: how each project imports the shared library

**Anchor.** [`projects/harness_core/`](../projects/harness_core/) v0.0.1 + porting plan in [285-harness-core-porting-plan](285-harness-core-porting-plan.md).

**One-line definition.** Concrete **integration patterns** showing each in-tree project how to **import and adopt `harness_core/`** — a **per-package adoption matrix** (which projects need `foundation/`, which need `protocols/a2a/`, which need `routines/`, etc.), **before/after code samples** for the canonical migration cases (Polaris's existing PermissionBridge → `harness_core.foundation.permissions.PermissionBridge`; Lyra's existing memory → `harness_core.memory`), **import patterns** (vendored dependency vs git-submodule vs PyPI), and **migration recipes** for the most common patterns — making the porting plan from [285](285-harness-core-porting-plan.md) actionable per-project.

## Per-package adoption matrix

| Project | foundation | capability | protocols | runtime | security | operations | compliance | routines | memory | skills | ux |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| Polaris | ✅ existing | ✅ extend | ✅ adopt | LangGraph wrap | ✅ adopt | ✅ extend | per-program | ✅ via daemon | reference | ✅ adopt | ⚠ optional |
| Lyra | ✅ adapt | ✅ local-tier | ✅ adopt | LangGraph + OpenAI SDK | ✅ adopt | ✅ Phoenix-local | GDPR | ✅ adopt | **source** | ✅ adopt | ✅ adopt |
| Mentat-Learn | ✅ adapt | ✅ small-tier | ✅ adopt | OpenAI SDK + LangGraph | ✅ adopt | ✅ Phoenix-VPS | GDPR | ✅ adopt | ✅ adopt | ✅ adopt | ✅ adopt |
| Argus | ✅ provider | ✅ provider | ✅ AGNTCY backbone | LangGraph | ✅ provider | ✅ HA | SOC 2 | ✅ central | ✅ adopt | ✅ provider | ⚠ |
| Aegis-Ops | ✅ strict | ✅ runbook | ✅ ops MCPs | LangGraph | ✅ strict | ✅ recursive | SOC 2 + EU | ✅ central | ✅ adopt | ✅ ops | ✅ adopt |
| Cipher-Sec | ✅ strict | ✅ STRIDE | ✅ secure | LangGraph | ✅ max | ✅ strict | SOC 2 + 27001 | ✅ adopt | ✅ adopt | ✅ adopt | ⚠ |
| Atlas-Research | ✅ adopt | ✅ retrieval | ✅ adopt | LangGraph | ✅ adopt | ✅ adopt | minimal | ✅ adopt | ✅ adopt | ✅ adopt | ⚠ |
| Helix-Bio | ✅ strict | ✅ bio | ✅ bio MCPs | LangGraph | ✅ strict | ✅ adopt | HIPAA + FDA | ✅ adopt | ✅ adopt | ✅ adopt | ⚠ |
| Harmony-Voice | ✅ adopt | ✅ small | ✅ voice MCPs | OpenAI SDK | ✅ adopt | ✅ latency | GDPR | ✅ adopt | ✅ adopt | ✅ adopt | ✅ voice |
| Vertex-Eval | ✅ adopt | ✅ judge | ✅ provider | LangGraph | ✅ adopt | ✅ provider | SOC 2 | ✅ adopt | ✅ adopt | ✅ adopt | ⚠ |
| Gnomon | ✅ adopt | ✅ obs | ✅ provider | LangGraph | ✅ adopt | ✅ source | SOC 2 + audit | ✅ adopt | ⚠ | ✅ adopt | ⚠ |
| Orion-Code | ✅ strict | ✅ code | ✅ git/code MCPs | LangGraph + Teams | ✅ strict | ✅ adopt | SOC 2 | ✅ adopt | ✅ adopt | ✅ adopt | ✅ adopt |
| Syndicate | ✅ adopt | ✅ orchestration | ✅ multi-runtime | hybrid | ✅ MAST | ✅ MAST-aware | SOC 2 | ✅ adopt | ✅ adopt | ✅ adopt | ⚠ |
| Quanta-Proof | ✅ adopt | ✅ math | ✅ proof MCPs | LangGraph | ✅ adopt | ✅ adopt | minimal | ✅ adopt | ✅ adopt | ✅ adopt | ⚠ |
| Open-Fang | harmonize | reference | harmonize | own + A2A | reference | reference | minimal | harmonize | reference | reference | ⚠ |

## Three import patterns

### Pattern 1: PyPI dependency (recommended for stable v1.0+)

```toml
# pyproject.toml in your project
dependencies = ["harness-core>=1.0.0"]
```

```python
from harness_core.foundation.permissions import PermissionBridge, ActionKind
from harness_core.protocols.a2a import AgentCard
from harness_core.routines import RoutineConfig, RoutineConfigStore
```

### Pattern 2: Git submodule (during v0.x development)

```bash
git submodule add ../harness_core projects/harness_core
pip install -e projects/harness_core
```

### Pattern 3: Vendored copy (full control, e.g. Lyra-class personal-use)

```bash
cp -r ../harness_core/src/harness_core projects/your-project/vendored/
```

## Canonical migration: Polaris's existing PermissionBridge → `harness_core`

**Before** (in polaris-core):

```python
# projects/polaris/packages/polaris-core/src/polaris_core/permissions/__init__.py
class PermissionBridge:
    def __init__(self, mode): ...
    def decide(self, action): ...
```

```python
# polaris-daemon usage
from polaris_core.permissions import PermissionBridge, PermissionMode, ActionKind
bridge = PermissionBridge(mode=PermissionMode.NORMAL)
```

**After** (after porting):

```python
# projects/polaris/packages/polaris-core/src/polaris_core/permissions/__init__.py
# Re-export from harness_core for backwards compatibility
from harness_core.foundation.permissions import (
    ActionKind,
    BrightLineCode,
    Decision,
    PermissionBridge,
    PermissionMode,
    ResearchAction,
    Verdict,
)

__all__ = [...]  # same as before
```

**Migration window**: 30 days. Existing `polaris_core.permissions` imports continue to work; new code imports from `harness_core.foundation.permissions` directly.

## Canonical migration: Lyra's memory → `harness_core.memory`

Lyra is the **source** for `harness_core.memory` — reverse of typical migration:

**Before** (lyra-core is the only consumer):

```python
from lyra_core.memory import AutoMemory, ReasoningBank, MemoryToolset
```

**After** (lyra-core re-exports from harness_core):

```python
# projects/lyra/packages/lyra-core/src/lyra_core/memory/__init__.py
from harness_core.memory import (
    AutoMemory,
    ReasoningBank,
    MemoryToolset,
    SqliteReasoningBank,
    # ... all 13 modules
)
```

Other consumers (Mentat-Learn, Polaris) import from `harness_core.memory` directly.

## Initialization recipe per project

```python
# Each project's main entrypoint
from harness_core.foundation.permissions import (
    BrightLineCode,
    PermissionBridge,
    PermissionMode,
)
from harness_core.operations.observability import HIREmitter, HIREventKind
from harness_core.protocols.a2a import AgentCard, AuthSpec, Capability, EndpointSpec
from harness_core.routines import RoutineConfigStore
from pathlib import Path


def bootstrap_project(project_name: str, data_dir: Path):
    # 1. Permission Bridge with project-specific bright-lines
    bridge = PermissionBridge(mode=PermissionMode.NORMAL)
    bridge.add_bright_line(BrightLineCode("PROJECT_PROD_CHANGE", "modify production config"))

    # 2. HIR emitter for observability
    emitter = HIREmitter(events_dir=data_dir / "hir")

    # 3. Routine store
    routine_store = RoutineConfigStore(data_dir / "routines.db")

    # 4. A2A agent card published at /.well-known/agent.json
    card = AgentCard(
        agent_id=project_name,
        version="0.1.0",
        vendor="harness-engineering",
        capabilities=[],
        endpoints=EndpointSpec(submit_task=f"https://{project_name}.tailnet/a2a/v1/tasks"),
        auth=AuthSpec(schemes=["oauth2.1"]),
    )

    return {"bridge": bridge, "emitter": emitter, "routine_store": routine_store, "card": card}
```

## Per-package import cheat sheet

```python
# L1 Foundation
from harness_core.foundation.permissions import PermissionBridge, ActionKind, ResearchAction, Verdict, Decision, PermissionMode, BrightLineCode

# L3 Protocol — A2A
from harness_core.protocols.a2a import AgentCard, Capability, EndpointSpec, AuthSpec, SignatureSpec
from harness_core.protocols.a2a.server import A2AServer, TaskHandle, Artifact, TaskStatus

# L6 Operations — Observability
from harness_core.operations.observability import HIREmitter, HIREventKind, SpanContext

# Routines
from harness_core.routines import RoutineConfig, RoutineConfigStore, TriggerSpec, TriggerKind, RoutineStatus
from harness_core.routines.triggers import CronEngine, CronTrigger
```

## Cross-project A2A discovery example

```python
# Project A consumes Project B's A2A capability
from harness_core.protocols.a2a import AgentCard
import httpx

# 1. Fetch the well-known agent card
resp = httpx.get("https://atlas-research.tailnet/.well-known/agent.json")
card_data = resp.json()
card = AgentCard(**card_data)

# 2. Verify signature against published JWKs
public_keys = httpx.get(card.auth.oauth2_metadata_url + "/jwks.json").json()
assert card.verify(public_keys), "agent card signature invalid"

# 3. Submit a task
resp = httpx.post(
    card.endpoints.submit_task,
    headers={"Authorization": f"Bearer {oauth_token}"},
    json={"capability_id": "literature-review", "input": {"topic": "agent scaling"}},
)
task = resp.json()
print(task["task_id"])
```

## Phased per-project rollout

| Project | Migration phase (per [285](285-harness-core-porting-plan.md)) | Effort |
|---|---|---|
| Polaris | M7 first | ~2-4 weeks (most existing code) |
| Lyra | M7 second (memory source) | ~2-3 weeks (re-export) |
| Argus | M7 third (provider role) | ~3-4 weeks |
| Mentat-Learn | M8 | ~2 weeks |
| Aegis-Ops | M8 | ~2-3 weeks |
| Cipher-Sec | M8 | ~3 weeks |
| Atlas / Helix / Harmony / Vertex / Gnomon / Orion / Syndicate / Quanta-Proof / Open-Fang | M8 parallel | ~1-2 weeks each |

## One-line takeaway

**Integration with `harness_core/` is a 30-day per-project migration: re-export from your existing module to `harness_core.*` for backwards compat, then refactor consumer code to import directly; the per-package adoption matrix tells you which sub-packages each project needs (Polaris uses everything, Harmony-Voice skips memory/skills heavy parts, Open-Fang harmonizes rather than rebuilds), with three import patterns (PyPI for stable, git submodule for v0.x, vendored for personal-use Lyra-class) covering all deployment scenarios.**
