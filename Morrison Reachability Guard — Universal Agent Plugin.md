<div align="center">

# Morrison Reachability Guard — Universal Agent Plugin

![Type](https://img.shields.io/badge/Type-Universal_Middleware-8b0000?style=flat-square)
![Language](https://img.shields.io/badge/Language-Python_3.10+-0075ca?style=flat-square)
![Compatibility](https://img.shields.io/badge/Works_With-OpenAI_·_LangChain_·_AutoGPT_·_Custom-0075ca?style=flat-square)
![Status](https://img.shields.io/badge/Status-Working_Plugin-2d6a2e?style=flat-square)
![Patent](https://img.shields.io/badge/Patent-GB2600765.8-0075ca?style=flat-square)
![©](https://img.shields.io/badge/©_Davarn_Morrison-555555?style=flat-square)

-----

*ℛ(t) ∩ Ω = ∅ — enforced at the tool boundary.*

*— Davarn Morrison, 2026*

</div>

-----

This work builds upon the patented pre-semantic trajectory governance framework.

-----

## What This Is

A lightweight middleware plugin that sits between an LLM agent and its tools, evaluating proposed actions before execution. The guard checks whether a proposed tool call would lead to a forbidden state — and blocks it before the tool fires.

**Current v1 implements one-step transition safety. Multi-step reachability is the next extension.**

```
LLM / Agent
     ↓
Proposed tool call
     ↓
Reachability Guard
     ↓
Approve / Reject
     ↓
Tool executes (only if safe)
```

Works with:

|Platform            |Integration Point           |
|:------------------:|:--------------------------:|
|OpenAI tool calling |Wrap tool execution function|
|LangChain           |Wrap `Tool.run()`           |
|AutoGPT             |Insert in `ActionExecutor`  |
|Custom agents       |Wrap any tool dispatch      |
|Robotics controllers|Wrap action execution       |
|Trading systems     |Wrap order submission       |

The guard is model-independent. It governs GPT, Claude, Gemini, or any system that calls tools — because it sits between intention and execution.

-----

## How It Works

### The Five Components

|Component         |Class              |Morrison Equation                                              |
|:----------------:|:-----------------:|:-------------------------------------------------------------:|
|System state      |`SystemState`      |`x̂_t ∈ X`                                                      |
|Proposed action   |`ProposedAction`   |`u_t ∈ U`                                                      |
|Transition model  |`TransitionModel`  |`x_{t+1} = F(x_t, u_t)`                                        |
|Forbidden region  |`ForbiddenSet`     |`Ω` — policy compiled into state constraints                   |
|Reachability Guard|`ReachabilityGuard`|v1: `F(x_t, u_t) ∉ Ω` · Target: `Reach(F(x_t, u_t), k) ∩ Ω = ∅`|
|Guarded agent     |`GuardedAgent`     |Universal tool wrapper — `tool_call → guard → execute`         |

### The Logic

```python
class ReachabilityGuard:

    def evaluate(self, state, action):

        next_state = self.transition.simulate(state, action)

        if self.forbidden.contains(next_state):
            return {"decision": "REJECT", "reason": "Action reaches forbidden region Ω"}

        return {"decision": "APPROVE", "reason": "Reachable future safe"}
```

Simulate the transition. Check if the next state enters Ω. If yes — block. If no — allow. Safety is enforced at the action-transition layer, before tool execution.

-----

## Example Output

**State the guard is inspecting:**

```python
current_state = {
    "task": "summarize confidential document",
    "document_type": "confidential",
    "internet_permission": False,
    "data_leak_risk": 0.0
}
```

**Guard decisions:**

```
Test 1: Internet search on confidential doc (no permission)
→ {'decision': 'REJECT', 'reason': 'Action reaches forbidden region Ω'}

Test 2: Write file (permitted)
→ {'decision': 'EXECUTED', 'tool': 'write_file', 'output': 'File written: Summary of document'}

Test 3: Internet search WITH permission
→ {'decision': 'EXECUTED', 'tool': 'internet_search', 'output': 'Search results for: public information'}
```

Test 1: LLM wants to search the internet while handling a confidential document without internet permission. **Blocked.** Test 2: LLM wants to write a file. Permission granted. **Executed.** Test 3: LLM wants to search the internet on a public topic with permission. **Executed.**

The guard doesn’t block everything. It blocks exactly what the forbidden set defines. Configurable. Domain-specific. Geometric.

-----

## The Forbidden Set — Where Policy Becomes Geometry

```python
class ForbiddenSet:

    def contains(self, state):

        # Internet used without permission
        if state.permissions.get("internet") is False:
            if state.environment.get("internet_used"):
                return True

        # Data leak risk exceeds threshold
        if state.risk_markers.get("data_leak", 0) > 0.8:
            return True

        return False
```

This is where institutions configure the guard. Different rules for different domains:

|Domain    |Example Rules in Ω                                                      |
|:--------:|:----------------------------------------------------------------------:|
|Finance   |Unauthorised transactions, insider trading states, regulatory violations|
|Healthcare|Drug interaction errors, consent violations, misdiagnosis cascades      |
|Defence   |Unauthorised escalation, intelligence leaks, targeting errors           |
|Enterprise|IP leakage, policy drift, unaudited autonomous actions                  |
|Sovereign |Data sovereignty breaches, national security violations                 |

Same plugin. Different forbidden set. The architecture doesn’t change.

-----

## The Universal Tool Wrapper

```python
class GuardedAgent:

    def __init__(self, tools):
        self.tools = tools
        self.guard = ReachabilityGuard()

    def run_tool(self, state, action):

        result = self.guard.evaluate(state, action)

        if result["decision"] != "APPROVE":
            return result

        tool_fn = self.tools[action.tool]
        output = tool_fn(**action.parameters)

        return {"decision": "EXECUTED", "tool": action.tool, "output": output}
```

Wrap any tool set. The guard evaluates before the tool runs. If the action is approved, the tool executes normally. If rejected, the tool never fires.

-----

## Integration Patterns

### OpenAI Tool Calling

```python
# Before:
result = tool_function(**tool_call.parameters)

# After:
guarded = GuardedAgent(tools)
result = guarded.run_tool(current_state, ProposedAction(
    tool=tool_call.name,
    parameters=tool_call.parameters
))
```

### LangChain

```python
class GuardedTool(Tool):
    def run(self, *args, **kwargs):
        result = guard.evaluate(current_state, proposed_action)
        if result["decision"] != "APPROVE":
            return result["reason"]
        return super().run(*args, **kwargs)
```

### AutoGPT / Custom Agent

```python
# In the action execution loop:
guard_result = guard.evaluate(state, proposed_action)
if guard_result["decision"] == "APPROVE":
    execute(proposed_action)
else:
    log_rejection(guard_result)
```

-----

## What Turns This Into a Real Product

Three upgrades from v1:

|Upgrade                |From            |To                                       |Impact                                                   |
|:---------------------:|:--------------:|:---------------------------------------:|:-------------------------------------------------------:|
|Multi-step reachability|`F(x, u) ∉ Ω`   |`Reach(F(x, u), k) ∩ Ω = ∅`              |Catches actions that are safe now but lead to Ω later    |
|Policy compiler        |Hand-coded rules|Law / enterprise policy → Ω automatically|Scales governance without manual rule writing            |
|Action projection      |REJECT only     |`u_safe = argmin ‖u − u_t‖`              |System proposes nearest safe alternative, not just blocks|

-----

## The Three Layers

```
THEORY
  Reachable manifold ℛ(t)
  Safety invariant ℛ ∩ Ω = ∅
         ↓

ARCHITECTURE
  Reachability Guard
  Five components
  Proved theorems
         ↓

SOFTWARE
  Runtime middleware
  Universal plugin
  ~100 lines
```

This is how a research theory becomes infrastructure.

-----

## How to Run

```bash
python morrison_reachability_guard_plugin.py
```

**Requirements:** Python 3.10+ (standard library only — zero dependencies).

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
║   Theory → Architecture → Software                                   ║
║                                                                      ║
║   ℛ(t) ∩ Ω = ∅                                                      ║
║                                                                      ║
║   Enforced at the tool boundary.                                     ║
║   Model-independent. Domain-configurable.                            ║
║   ~100 lines. Zero dependencies.                                     ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

-----

Morrison Reachability Guard · Universal Agent Plugin · Geometric Control Theory of Cognition

GB2600765.8 · GB2602013.1 · GB2602072.7 · GB26023332.5

© 2026 Davarn Morrison — All Rights Reserved

-----

## Related Work

- [Morrison Reachability Guard — Runtime Interface](https://github.com/davarn/morrison-framework) — Protocol-based typed interface
- [Morrison Reachability Guard — LLM Agent Governance](https://github.com/davarn/morrison-framework) — Policy-based agent guard
- [Morrison Reachability Guard — Architecture Specification](https://github.com/davarn/morrison-framework) — Full spec with theorems
- [Morrison Reachability Guard — Toy Prototype](https://github.com/davarn/morrison-framework) — 2D visualised demo
- [Geometric Control Theory of Cognition](https://github.com/davarn/morrison-framework) — The base theory
- [Licensing, Citation, and IP](https://github.com/davarn/morrison-framework) — How to cite and licence
