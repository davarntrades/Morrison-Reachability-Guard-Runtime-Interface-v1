<div align="center">

# Morrison Reachability Guard — LLM Agent Governance Layer

![Type](https://img.shields.io/badge/Type-Agent_Governance-8b0000?style=flat-square)
![Language](https://img.shields.io/badge/Language-Python_3.10+-0075ca?style=flat-square)
![Domain](https://img.shields.io/badge/Domain-LLM_Tool_Use_Safety-8b0000?style=flat-square)
![Status](https://img.shields.io/badge/Status-Working_Implementation-2d6a2e?style=flat-square)
![Patent](https://img.shields.io/badge/Patent-GB2600765.8-0075ca?style=flat-square)
![©](https://img.shields.io/badge/©_Davarn_Morrison-555555?style=flat-square)

-----

*Safety is not what you filter from the output. Safety is what you exclude from the reachable set.*

*— Davarn Morrison, 2026*

</div>

-----

This work builds upon the patented pre-semantic trajectory governance framework.

-----

## What This Is

A practical instantiation of the Morrison Reachability Guard for LLM agent systems. Instead of a toy 2D state space, this version operates on real agent runtime states — tasks, tools, permissions, memory, environment, and risk markers. The guard sits between an LLM planner and tool execution, checking whether proposed actions lead to forbidden states before they happen.

This implementation enforces the invariant at the one-step transition level, providing a minimal, executable instantiation of the framework. Full multi-step reachability remains the next stage of validation.

-----

## The Architecture

```
LLM Planner
      ↓
ProposedAction (tool call, API request, response class)
      ↓
┌──────────────────────────────┐
│   MORRISON REACHABILITY GUARD │
│                               │
│   TransitionModel: F(x_t,u_t) │
│   ForbiddenSet: Ω             │
│   Check: F(x_t,u_t) ∉ Ω      │
│                               │
└──────────┬────────────────────┘
           │
     ┌─────▼─────┐
     │  APPROVE   │  → Tool execution
     │  or REJECT │  → Blocked with reason
     └───────────┘
```

-----

## The Components

-----

### 1. SystemState — The Agent’s Current Configuration

Represents `x̂_t ∈ X` — everything the guard needs to evaluate reachability.

```python
@dataclass
class SystemState:
    task: str
    step: int
    memory: Dict[str, Any]
    tools_available: Dict[str, Any]
    environment: Dict[str, Any]
    permissions: Dict[str, bool]
    risk_markers: Dict[str, float]
```

|Field            |What It Encodes                   |Example                                  |
|:---------------:|:--------------------------------:|:---------------------------------------:|
|`task`           |Current objective                 |`"summarize financial document"`         |
|`step`           |Position in task sequence         |`2`                                      |
|`memory`         |Accumulated context               |Session variables, retrieved docs        |
|`tools_available`|Active tool capabilities          |`{"search": True, "write": True}`        |
|`environment`    |External conditions               |`{"document_type": "confidential"}`      |
|`permissions`    |Access control state              |`{"internet": False, "file_write": True}`|
|`risk_markers`   |Proximity to constraint boundaries|`{"data_leak_risk": 0.2}`                |

**Example:**

```python
state = SystemState(
    task="summarize confidential report",
    step=0,
    memory={},
    tools_available={},
    environment={"document_type": "confidential"},
    permissions={"internet": False},
    risk_markers={"data_leak_risk": 0.2}
)
```

-----

### 2. ProposedAction — What the LLM Wants to Do

Represents `u_t ∈ U` — the action proposed by the model’s planner.

```python
@dataclass
class ProposedAction:
    tool: str
    parameters: Dict[str, Any]
    confidence: float = 1.0
```

**Example:**

```python
action = ProposedAction(
    tool="internet_search",
    parameters={"query": "company financial results"}
)
```

The LLM wants to search the internet. The guard will check whether this transition enters a forbidden state.

-----

### 3. TransitionModel — The Dynamics Equation in Software

Implements `x_{t+1} = F(x_t, u_t)` — what happens if the action is executed.

```python
class TransitionModel:

    def simulate(self, state: SystemState, action: ProposedAction) -> SystemState:

        new_state = SystemState(
            task=state.task,
            step=state.step + 1,
            memory=dict(state.memory),
            tools_available=state.tools_available,
            environment=dict(state.environment),
            permissions=dict(state.permissions),
            risk_markers=dict(state.risk_markers)
        )

        if action.tool == "internet_search":
            new_state.environment["internet_used"] = True
            new_state.risk_markers["data_leak_risk"] = \
                state.risk_markers.get("data_leak_risk", 0) + 0.3

        if action.tool == "write_file":
            new_state.environment["file_written"] = True

        return new_state
```

This is where the agent’s workflow graph lives. Each tool call has consequences that change the state. The transition model makes those consequences explicit and evaluable *before execution*.

**Extension path.** In more advanced implementations, the transition model can operate over higher-dimensional representations, including latent embeddings or vector states, enabling reachability constraints to be applied directly within model-internal spaces.

-----

### 4. ForbiddenSet — Policy Compiled into Geometry

Implements membership in `Ω` — translates real-world policy into state-space constraints.

```python
class ForbiddenSet:

    def contains(self, state: SystemState) -> bool:

        # Rule 1: No internet access for confidential documents
        if state.environment.get("document_type") == "confidential":
            if state.environment.get("internet_used"):
                return True

        # Rule 2: Data leak risk exceeds threshold
        if state.risk_markers.get("data_leak_risk", 0) > 0.8:
            return True

        # Rule 3: Internet used without permission
        if state.permissions.get("internet") is False:
            if state.environment.get("internet_used"):
                return True

        return False
```

|Rule|Policy                                 |Forbidden State                                              |
|:--:|:-------------------------------------:|:-----------------------------------------------------------:|
|1   |Confidential docs cannot touch internet|`document_type == "confidential"` AND `internet_used == True`|
|2   |Data leak risk cannot exceed 0.8       |`data_leak_risk > 0.8`                                       |
|3   |Internet requires explicit permission  |`internet permission == False` AND `internet_used == True`   |

This is where **institutions configure the guard**. Different rules for different domains. Same architecture. The forbidden set is the governance layer.

-----

### 5. The Reachability Guard — Enforcing the Safety Invariant

This implementation enforces a local (one-step) approximation of the invariant, ensuring that immediate transitions do not enter Ω.

```python
class ReachabilityGuard:

    def __init__(self, transition_model, forbidden_set):
        self.transition_model = transition_model
        self.forbidden_set = forbidden_set

    def evaluate(self, state: SystemState, action: ProposedAction):

        next_state = self.transition_model.simulate(state, action)

        if self.forbidden_set.contains(next_state):
            return {
                "decision": "REJECT",
                "reason": "Action leads to forbidden region Ω"
            }

        return {
            "decision": "APPROVE",
            "reason": "Reachable future safe"
        }
```

The guard simulates the transition *before it happens*. If the next state enters Ω, the action is rejected. If not, the action is approved. Safety is checked at the dynamics layer, not the output layer.

This version does not yet account for delayed violations arising from multi-step trajectories, which is the focus of ongoing work.

-----

## Example: The Guard in Action

```python
state = SystemState(
    task="summarize confidential report",
    step=0,
    memory={},
    tools_available={},
    environment={"document_type": "confidential"},
    permissions={"internet": False},
    risk_markers={"data_leak_risk": 0.2}
)

action = ProposedAction(
    tool="internet_search",
    parameters={"query": "report summary"}
)

guard = ReachabilityGuard(
    transition_model=TransitionModel(),
    forbidden_set=ForbiddenSet()
)

result = guard.evaluate(state, action)
print(result)
```

**Output:**

```json
{
    "decision": "REJECT",
    "reason": "Action leads to forbidden region Ω"
}
```

The LLM wanted to search the internet while handling a confidential document without internet permission. The guard checked the transition. The next state enters Ω. The action is blocked *before execution*.

`F(x_t, u_t) ∉ Ω` — enforced at runtime at the transition boundary.

-----

## What Each Component Maps To

|Morrison Equation      |Code Component                |What It Does                                   |
|:---------------------:|:----------------------------:|:---------------------------------------------:|
|`x̂_t ∈ X`              |`SystemState`                 |Represents the agent’s current configuration   |
|`u_t ∈ U`              |`ProposedAction`              |What the LLM planner wants to do               |
|`x_{t+1} = F(x_t, u_t)`|`TransitionModel.simulate()`  |Computes next state from current state + action|
|`Ω`                    |`ForbiddenSet.contains()`     |Policy compiled into state-space constraints   |
|`F(x_t, u_t) ∉ Ω`      |`ReachabilityGuard.evaluate()`|Checks one-step safety before execution        |
|APPROVE / REJECT       |Guard output                  |The governance decision                        |

-----

## The Next Stage: Multi-Step Reachability

The current implementation enforces one-step transition safety. Multi-step reachability is already partially implemented in V2/V3 prototypes and represents the next validation stage.

The framework extends naturally to multi-step reachability:

```
Reach(F(x̂_t, u_t), k) ∩ Ω = ∅
```

This checks not just whether the immediate next state is safe, but whether any state reachable within `k` steps of the next state could enter Ω. An action might be safe now but lead to a trajectory that reaches Ω later. Multi-step reachability catches this.

|Horizon     |Check                          |Catches               |
|:----------:|:-----------------------------:|:--------------------:|
|1-step      |`F(x̂_t, u_t) ∉ Ω`              |Immediate violations  |
|k-step      |`Reach(F(x̂_t, u_t), k) ∩ Ω = ∅`|Near-future violations|
|Full-horizon|`Reach⁺(F(x̂_t, u_t)) ∩ Ω = ∅`  |All future violations |

This is the difference between a filter and a governance architecture. Filters evaluate outputs. The guard constrains trajectories.

-----

## Why This Matters

|Current AI Safety                    |Morrison Reachability Guard                             |
|:-----------------------------------:|:------------------------------------------------------:|
|Filters output text                  |Constrains state transitions                            |
|Acts on language (L-axis)            |Acts on dynamics (C-axis)                               |
|Applied after unsafe state is reached|Applied before unsafe state can be reached              |
|Jailbreaks bypass it                 |Jailbreaks cannot reach Ω — it is geometrically excluded|
|Model-specific rules                 |Model-independent architecture                          |
|Reactive                             |Pre-emptive                                             |

This targets failure modes such as indirect data leakage, multi-step prompt injection, and tool misuse that cannot be reliably prevented through output filtering alone.

-----

## How to Run

```bash
python morrison_reachability_guard_llm_agent.py
```

**Requirements:** Python 3.10+ (standard library only).

-----

## Citation

```
Morrison, D. (2026). Geometric Control Theory of Cognition:
A Reachability-Based Framework for Identity, Intelligence, and Experience.
Independent Research. UK Patent Applications: GB2600765.8,
GB2602013.1, GB2602072.7, GB26023332.5.

Available at: https://github.com/davarn/morrison-framework
```

**BibTeX:**

```bibtex
@misc{morrison2026framework,
  author       = {Morrison, Davarn},
  title        = {Geometric Control Theory of Cognition: A Reachability-Based
                  Framework for Identity, Intelligence, and Experience},
  year         = {2026},
  publisher    = {Independent Research},
  note         = {UK Patent Applications: GB2600765.8, GB2602013.1,
                  GB2602072.7, GB26023332.5},
  url          = {https://github.com/davarn/morrison-framework}
}
```

-----

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║   ℛ(t) ∩ Ω = ∅                                                      ║
║                                                                      ║
║   Safety is not what you filter from the output.                     ║
║   Safety is what you exclude from the reachable set.                 ║
║                                                                      ║
║   This code enforces the one-step invariant at runtime.              ║
║   Multi-step reachability is the next validation stage.              ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

-----

Morrison Reachability Guard · LLM Agent Governance Layer · Geometric Control Theory of Cognition

GB2600765.8 · GB2602013.1 · GB2602072.7 · GB26023332.5

© 2026 Davarn Morrison — All Rights Reserved

-----

## Related Work

- [Morrison Reachability Guard — Runtime Interface](https://github.com/davarn/morrison-framework) — Protocol-based typed interface
- [Morrison Reachability Guard — Architecture Specification](https://github.com/davarn/morrison-framework) — Full five-component spec with theorems
- [Morrison Reachability Guard — Toy Prototype](https://github.com/davarn/morrison-framework) — 2D visualised demo
- [Geometric Control Theory of Cognition — Formal Paper](https://github.com/davarn/morrison-framework) — The base theory
- [The Morrison Reality Table](https://github.com/davarn/morrison-framework) — Unified expression of the framework
- [Licensing, Citation, and IP](https://github.com/davarn/morrison-framework) — How to cite and licence
