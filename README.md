<div align="center">

# Morrison Reachability Guard — Runtime Interface v1

![Type](https://img.shields.io/badge/Type-Runtime_Interface-8b0000?style=flat-square)
![Language](https://img.shields.io/badge/Language-Python_3.10+-0075ca?style=flat-square)
![Status](https://img.shields.io/badge/Status-Working_Interface-2d6a2e?style=flat-square)
![Patent](https://img.shields.io/badge/Patent-GB2600765.8-0075ca?style=flat-square)
![©](https://img.shields.io/badge/©_Davarn_Morrison-555555?style=flat-square)

-----

*Governing language governs the shadow. Governing reachability governs the system.*

*— Davarn Morrison, 2026*

</div>

-----

This work builds upon the patented pre-semantic trajectory governance framework.

-----

## What This Is

A production-grade runtime interface for the Morrison Reachability Guard — the governance layer that sits between a model and real actions. This is not a toy prototype. It is a typed, protocol-based interface skeleton designed to be extended with domain-specific implementations.

The guard enforces a single invariant at runtime:

**Reach⁺(F(x̂_t, u_t), k) ∩ Ω = ∅**

Forbidden states are geometrically unreachable under governed dynamics.

-----

## Architecture

```
User / Environment
        ↓
   Model / Planner
        ↓
 ProposedAction u_t
        ↓
MorrisonReachabilityGuard
  ├─ TransitionModel     F(x_t, u_t) → x_{t+1}
  ├─ ReachabilityEngine  Reach⁺(F(x_t,u_t), k) ∩ Ω = ∅ ?
  ├─ ForbiddenSet        Ω — configurable per domain
  ├─ Redirector          u_safe = argmin_{u ∈ A_safe} ‖u − u_t‖
  └─ DriftMonitor        ‖ΔG‖, safety margin, decision log
        ↓
Approve / Redirect / Reject
        ↓
Execution Layer
```

-----

## Core Data Structures

|Class            |Represents                   |Morrison Object                  |
|:---------------:|:---------------------------:|:-------------------------------:|
|`SystemState`    |Current system configuration |`x̂_t ∈ X`                        |
|`ProposedAction` |Action proposed by the model |`u_t ∈ U`                        |
|`StateTransition`|Result of applying an action |`F(x_t, u_t) → x_{t+1}`          |
|`GuardResult`    |Guard’s decision and metadata|Decision + safety margin + reason|
|`Decision`       |APPROVE / REDIRECT / REJECT  |Output of the reachability check |

-----

## Protocol Interfaces

The guard is implementation-agnostic. It depends on four protocol interfaces that concrete implementations must satisfy:

|Protocol            |Contract                                                 |Morrison Equation               |
|:------------------:|:-------------------------------------------------------:|:------------------------------:|
|`TransitionModel`   |`__call__(state, action) → StateTransition`              |`F(x_t, u_t) → x_{t+1}`         |
|`ForbiddenSet`      |`contains(state) → bool` and `distance(state) → float`   |Membership in Ω                 |
|`ReachabilityEngine`|`is_safe_action(state, action, ...) → (bool, float)`     |`Reach⁺(F(x̂_t, u_t), k) ∩ Ω = ∅`|
|`Redirector`        |`redirect(state, action, ...) → Optional[ProposedAction]`|`argmin_{u ∈ A_safe} ‖u − u_t‖` |

Any implementation satisfying these contracts plugs into the guard. The architecture is domain-independent.

-----

## The Guard — Core Logic

```python
class MorrisonReachabilityGuard:

    def evaluate(self, state, action) -> GuardResult:
        # 1. Reachability check
        safe, margin = self.reachability_engine.is_safe_action(...)

        # 2. If safe → APPROVE
        if safe:
            return GuardResult(decision=APPROVE, ...)

        # 3. If unsafe → attempt REDIRECT
        redirected = self.redirector.redirect(...)
        if redirected is not None:
            return GuardResult(decision=REDIRECT, ...)

        # 4. If no safe alternative → REJECT
        return GuardResult(decision=REJECT, ...)
```

Three possible outputs:

|Decision    |When                               |What Happens                                       |
|:----------:|:---------------------------------:|:-------------------------------------------------:|
|**APPROVE** |Action is safe                     |Execute as proposed — user doesn’t notice the guard|
|**REDIRECT**|Unsafe, but safe alternative exists|Substitute nearest safe action automatically       |
|**REJECT**  |Unsafe, no safe alternative        |Block with geometric reason                        |

-----

## Concrete Implementation — Toy 2D System

The repository includes a minimal concrete implementation for a 2D agent:

|Class                           |What It Implements  |Details                                            |
|:------------------------------:|:------------------:|:-------------------------------------------------:|
|`ToyForbiddenDisk`              |`ForbiddenSet`      |Ω = disk of radius r at origin                     |
|`ToyTransitionModel`            |`TransitionModel`   |F(x, u) = x + (dx, dy)                             |
|`ShortHorizonReachabilityEngine`|`ReachabilityEngine`|Reapplies action for k steps, checks if any enter Ω|
|`GridRedirector`                |`Redirector`        |Searches action grid for nearest safe alternative  |

-----

## Example Output

```
Initial position: (4.0, 3.0)
Forbidden region: Ω = disk of radius 1.5 at origin

Step | Decision  | Proposed          | Executed          | Margin
-----+----------+-------------------+-------------------+-------
   0 | ✓ approve | (-1.00, +0.00)   | (-1.00, +0.00)   | 1.66
   1 | → redirect| (-0.80, -0.60)   | (-0.87, -0.50)   | -0.16
   2 | → redirect| (-0.50, -0.80)   | (-0.19, -0.72)   | -0.05
   3 | ✓ approve | (+0.50, +0.50)   | (+0.50, +0.50)   | 1.84

Safety invariant ℛ ∩ Ω = ∅ maintained: True
```

-----

## How to Run

```bash
python morrison_reachability_guard_runtime.py
```

**Requirements:** Python 3.10+ (standard library only — no external dependencies).

-----

## How to Extend

To use the guard in a real domain, implement the four protocols:

|Step                             |What to Build                            |Example                                                 |
|:-------------------------------:|:---------------------------------------:|:------------------------------------------------------:|
|1. Define your state space       |`SystemState` with domain-specific fields|Agent context, tool permissions, memory                 |
|2. Implement `TransitionModel`   |How actions change state                 |API call effects, tool execution, state mutation        |
|3. Implement `ForbiddenSet`      |What states are forbidden                |Policy violations, unsafe tool calls, data exfiltration |
|4. Implement `ReachabilityEngine`|How to check if actions reach Ω          |CBFs, Monte Carlo, neural reachability, interval methods|
|5. Implement `Redirector`        |How to find safe alternatives            |Gradient-based projection, action-space search          |

The guard itself doesn’t change. Only the implementations behind the protocols change per domain.

-----

## What This Proves

|Claim                                  |Evidence                                     |
|:-------------------------------------:|:-------------------------------------------:|
|The guard architecture is implementable|Working Python interface with typed protocols|
|It is domain-independent               |Protocols accept any concrete implementation |
|It enforces geometric safety           |`Reach⁺ ∩ Ω = ∅` checked at every step       |
|It redirects, not just rejects         |Nearest safe action computed and substituted |
|It monitors drift                      |Decision history logged per step             |

-----

## From Theory to Interface

|Morrison Equation               |Interface Component                      |
|:------------------------------:|:---------------------------------------:|
|`x_{t+1} = F(x_t, u_t)`         |`TransitionModel.__call__()`             |
|`ℛ(t) ∩ Ω = ∅`                  |`ReachabilityEngine.is_safe_action()`    |
|`Ω`                             |`ForbiddenSet.contains()` / `.distance()`|
|`A_safe(x) = { u | F(x,u) ∈ S }`|`Redirector.redirect()`                  |
|`‖ΔG‖`, safety margin           |`DriftMonitor.update()`                  |
|APPROVE / REDIRECT / REJECT     |`MorrisonReachabilityGuard.evaluate()`   |

-----

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║   The Morrison Framework defines what safety IS.                     ║
║   The Reachability Guard enforces it at runtime.                     ║
║   This interface makes it implementable.                             ║
║                                                                      ║
║   Reach⁺(F(x̂_t, u_t), k) ∩ Ω = ∅                                  ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

-----

Morrison Reachability Guard · Runtime Interface v1 · Geometric Control Theory of Cognition

GB2600765.8 · GB2602013.1 · GB2602072.7 · GB26023332.5

© 2026 Davarn Morrison — All Rights Reserved

-----

## Related Work

- [Morrison Reachability Guard — Architecture Specification](https://github.com/davarn/morrison-framework) — Full five-component spec with theorems
- [Morrison Reachability Guard — Toy Prototype](https://github.com/davarn/morrison-framework) — 2D visualised demo with matplotlib
- [Geometric Control Theory of Cognition — Formal Paper](https://github.com/davarn/morrison-framework) — The base theory
- [The Morrison Reality Table](https://github.com/davarn/morrison-framework) — Unified expression of the framework
- [Licensing, Citation, and IP](https://github.com/davarn/morrison-framework) — How to cite and licence
