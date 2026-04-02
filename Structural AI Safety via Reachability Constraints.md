<div align="center">

# Structural AI Safety via Reachability Constraints

![Type](https://img.shields.io/badge/Type-Research_Paper-8b0000?style=flat-square)
![Domain](https://img.shields.io/badge/Domain-AI_Safety_·_Control_Theory-0075ca?style=flat-square)
![Method](https://img.shields.io/badge/Method-Reachability_Analysis-2d6a2e?style=flat-square)
![Stage](https://img.shields.io/badge/Stage-v1_Implemented_·_v2/v3_Defined-555555?style=flat-square)
![Patent](https://img.shields.io/badge/Patent-GB2600765.8-0075ca?style=flat-square)
![©](https://img.shields.io/badge/©_Davarn_Morrison-555555?style=flat-square)

-----

*Safety is not a property of what a system says. It is a property of what a system can reach.*

*— Davarn Morrison, 2026*

</div>

-----

This work builds upon the patented pre-semantic trajectory governance framework.

-----

## Structural AI Safety via Reachability Constraints: From Transition-Level Guards to Trajectory-Level Invariants

**Davarn Morrison — 2026**

-----

## Abstract

This paper introduces a reachability-based framework for structural AI safety, in which safety is defined as the exclusion of unsafe states from the reachable state space of a system. Unlike conventional approaches that operate at the level of outputs or behaviour, this framework constrains system dynamics directly.

A progression is presented from a minimal working implementation (v1), which enforces one-step transition safety, to a generalised formulation (v3) based on full trajectory-level reachability invariants. The v1 system is instantiated as a middleware guard for LLM agents, intercepting tool calls and blocking actions that would transition the system into a forbidden region. This provides an executable approximation of the safety invariant.

The formulation is then extended to multi-step reachability (v2), and finally to full-horizon safety guarantees (v3), where unsafe states are structurally unreachable. The framework establishes a bridge between AI safety and geometric control theory, reframing safety as a property of reachable state space rather than generated outputs.

-----

## 1. Introduction

Current approaches to AI safety operate primarily at the output layer. Reinforcement learning from human feedback (RLHF) [4] adjusts model behaviour toward preferred outputs. Output filtering intercepts harmful text after generation. Adversarial training hardens models against known attack patterns.

All three share a structural limitation [7]: unsafe states remain reachable. The system can reach a harmful configuration internally — the intervention occurs only at the boundary between system and user. A sufficiently adversarial input can bypass any output-layer defence because the defence does not constrain the dynamics that produce the output.

This paper proposes a different framing.

Safety should be defined over reachable state space, not outputs. A system is safe if and only if no trajectory from any permitted action reaches a forbidden state. This is a geometric property of the system’s dynamics — not a statistical property of its outputs.

The contribution of this work is threefold:

```
════════════════════════════════════════════════════════════════
 1. A formal framework defining safety as state-space exclusion
 2. A minimal working implementation (v1) enforcing one-step
    transition safety as a middleware guard for LLM agents
 3. A clear progression from v1 to full reachability invariants
    (v2/v3), with each stage formally defined
════════════════════════════════════════════════════════════════
```

-----

## 2. System Model

### 2.1 State Space and Dynamics

Let X denote the state space of an agent system. At each time step, the system occupies a state x_t ∈ X and selects an action u_t ∈ U. The system evolves according to:

```
x_{t+1} = F(x_t, u_t)
```

where F: X × U → X is the transition function.

### 2.2 Forbidden Region

Let Ω ⊂ X denote the forbidden region — the set of states the system must never occupy [2]. Ω encodes policy, law, safety constraints, or institutional rules compiled into state-space geometry.

### 2.3 Reachable Set

The k-step reachable set from state x under all admissible actions [3, 6] is:

```
Reach(x, k) = { x' ∈ X | ∃ sequence (u₀, …, u_{k-1}) such that
                x' is reached from x by applying F iteratively }
```

The full reachable set is:

```
Reach⁺(x) = ⋃_{k≥0} Reach(x, k)
```

### 2.4 Safety Invariant

The system is safe if and only if:

```
╔═══════════════════════════════════════════╗
║                                           ║
║   Reach⁺(x₀) ∩ Ω = ∅                     ║
║                                           ║
║   No reachable state is forbidden.        ║
║   Safety by construction.                 ║
║                                           ║
╚═══════════════════════════════════════════╝
```

This is the target invariant. The remainder of this paper describes a progression toward enforcing it at runtime.

-----

## 3. v1 — Transition-Level Safety

### 3.1 Equation

```
F(x_t, u_t) ∉ Ω
```

v1 enforces safety at the one-step transition level. Before an action is executed, the guard simulates the resulting state. If that state enters Ω, the action is rejected.

### 3.2 Architecture

```
LLM Planner
      ↓
Proposed Action (u_t)
      ↓
┌──────────────────────────────┐
│   REACHABILITY GUARD          │
│                               │
│   Simulate: F(x_t, u_t)      │
│   Check: F(x_t, u_t) ∉ Ω     │
│                               │
└──────────┬────────────────────┘
           │
     ┌─────▼─────┐
     │  APPROVE   │  → Tool executes
     │  or REJECT │  → Action blocked
     └───────────┘
```

The guard is implemented as a middleware layer between the LLM planner and tool execution. It is model-independent — it governs any system that proposes actions, regardless of the underlying model.

### 3.3 Implementation

The v1 system consists of five components:

|Component        |Role                                  |Equation             |
|:---------------:|:------------------------------------:|:-------------------:|
|SystemState      |Agent’s current configuration         |x_t ∈ X              |
|ProposedAction   |Action proposed by the planner        |u_t ∈ U              |
|TransitionModel  |Simulates next state                  |x_{t+1} = F(x_t, u_t)|
|ForbiddenSet     |Policy compiled into state constraints|Ω ⊂ X                |
|ReachabilityGuard|Evaluates safety before execution     |F(x_t, u_t) ∉ Ω      |

The guard operates at the action-transition layer. The forbidden set is configurable per domain. The transition model makes action consequences explicit and evaluable before execution.

### 3.4 Scope

v1 provides a minimal executable approximation of the safety invariant. It enforces immediate transition safety but does not detect delayed violations arising from multi-step trajectories.

### 3.5 Theorem: One-Step Safety Guarantee

```
╔═══════════════════════════════════════════════════════════════════╗
║                                                                   ║
║   Theorem 1 (One-Step Safety Guarantee)                           ║
║                                                                   ║
║   Let G be a reachability guard that permits action u_t            ║
║   if and only if F(x_t, u_t) ∉ Ω.                                 ║
║                                                                   ║
║   Then no executed transition enters Ω.                           ║
║                                                                   ║
║   Proof.                                                          ║
║   Suppose action u_t is executed. Then G has approved u_t,         ║
║   which requires F(x_t, u_t) ∉ Ω. The resulting state             ║
║   x_{t+1} = F(x_t, u_t) therefore satisfies x_{t+1} ∉ Ω.        ║
║   Since this holds for every executed action, no executed          ║
║   transition produces a state in Ω.                               ║
║                                                                   ║
║   Note. This guarantees immediate safety only. It does not         ║
║   guarantee that x_{t+1} cannot subsequently reach Ω under        ║
║   future actions. That guarantee requires v2/v3.                  ║
║                                                                   ║
╚═══════════════════════════════════════════════════════════════════╝
```

### 3.6 Preliminary Result

In preliminary controlled tests, the v1 guard successfully blocked all explicitly unsafe transitions — including internet access on confidential documents, permission escalation attempts, and data exfiltration via tool chaining — while permitting all safe actions without false rejections.

-----

## 4. v2 — Multi-Step Reachability

### 4.1 Equation

```
Reach(F(x_t, u_t), k) ∩ Ω = ∅
```

An action is safe only if no state reachable within k steps of the resulting state enters Ω.

### 4.2 Motivation

v1 catches immediate violations. v2 catches delayed violations — actions that are safe at the point of execution but lead to trajectories that reach Ω within a bounded horizon.

```
┌──────────────────────────────────────────────────┐
│                                                    │
│   v1: Is the next state safe?                      │
│   v2: Is the next state AND its k-step future safe?│
│                                                    │
└──────────────────────────────────────────────────┘
```

### 4.3 Failure Modes Addressed

|Failure Mode               |v1 Detects|v2 Detects|
|:-------------------------:|:--------:|:--------:|
|Immediate unsafe action    |Yes       |Yes       |
|Multi-step prompt injection|No        |Yes       |
|Indirect data leakage      |No        |Yes       |
|Tool chaining to Ω         |No        |Yes       |

### 4.4 Computational Considerations

Multi-step reachability introduces computational cost that scales with the branching factor of the action space and the depth k. Practical implementations require approximation strategies:

```
Approach               → Trade-off
────────────────────────────────────────────────
Exact enumeration       → Exponential in k
Monte Carlo sampling    → Probabilistic guarantee
Symbolic abstraction    → Conservative but scalable
Learned transition model→ Approximate but fast
```

Preliminary V2 prototypes have been defined and are pending empirical validation.

-----

## 5. v3 — Full Reachability Invariant

### 5.1 Equation

```
╔═══════════════════════════════════════════════════════════╗
║                                                           ║
║   Reach⁺(F(x_t, u_t)) ∩ Ω = ∅                            ║
║                                                           ║
║   No possible trajectory from this action reaches Ω.      ║
║   Safety is a structural property of the system.          ║
║                                                           ║
╚═══════════════════════════════════════════════════════════╝
```

### 5.2 Interpretation

v3 is the full reachability invariant. An action is safe if and only if no trajectory originating from the resulting state — at any horizon — enters Ω. Safety is no longer a local check. It is a global structural property.

At this level, unsafe states are not merely unlikely or undetected. They are geometrically unreachable.

### 5.3 Relationship to Control Theory

The v3 invariant is structurally equivalent to the safety condition in Control Barrier Function (CBF) theory [1], restated as a global reachability property rather than a local differential constraint. The forward-invariant safe set S = X \ Ω [8] is preserved under any admissible action sequence.

### 5.4 The Progression

|Version|Check                        |Guarantee             |
|:-----:|:---------------------------:|:--------------------:|
|v1     |F(x_t, u_t) ∉ Ω              |Immediate safety      |
|v2     |Reach(F(x_t, u_t), k) ∩ Ω = ∅|Bounded-horizon safety|
|v3     |Reach⁺(F(x_t, u_t)) ∩ Ω = ∅  |Full structural safety|

Each version is a strict extension of the previous. v1 ⊂ v2 ⊂ v3.

-----

## 6. Comparison to Existing Methods

|Method                  |Operates On|Guarantee Level|Limitation                                  |
|:----------------------:|:---------:|:-------------:|:------------------------------------------:|
|RLHF [4]                |Behaviour  |Statistical    |No structural guarantee                     |
|Output filtering        |Output     |Pattern-based  |Reactive — unsafe state already reached     |
|Adversarial training    |Data       |Distribution   |Incomplete — novel attacks bypass           |
|Constitutional AI [5]   |Output     |Rule-based     |Language-level — does not constrain dynamics|
|Reachability (this work)|State space|Structural     |Constrains dynamics directly                |

Existing methods operate on outputs. This framework operates on dynamics. The distinction is between filtering what a system says and constraining what a system can reach.

-----

## 7. Experimental Framework

### 7.1 Setup

The v1 system is evaluated in the following configuration:

```
Agent:       LLM with tool-calling capability
Tools:       Internet search, file write, database query
Guard:       Morrison Reachability Guard (v1)
Forbidden set: Confidential data exfiltration, unauthorised
               internet access, permission escalation
```

### 7.2 Test Conditions

|Condition  |Description                                     |
|:---------:|:----------------------------------------------:|
|Baseline   |Agent operates without guard                    |
|Guarded    |Agent operates with v1 guard                    |
|Adversarial|Agent receives prompts designed to bypass safety|

### 7.3 Metrics

|Metric                      |Definition                                         |
|:--------------------------:|:-------------------------------------------------:|
|Unsafe action execution rate|Proportion of actions entering Ω that are executed |
|Blocked transition rate     |Proportion of Ω-bound transitions rejected by guard|
|False rejection rate        |Proportion of safe actions incorrectly blocked     |
|Bypass rate (adversarial)   |Proportion of adversarial prompts reaching Ω       |

### 7.4 Status

The experimental framework is defined. Systematic evaluation across adversarial conditions is the subject of ongoing work.

-----

## 8. Limitations

This work has the following limitations, each of which represents a direction for future research:

**Forbidden set definition.** The framework requires Ω to be explicitly defined. In complex domains, specifying all forbidden states is non-trivial. The quality of safety guarantees depends directly on the completeness of Ω.

**Transition model fidelity.** The guard relies on a transition model F that approximates real system dynamics. If F diverges from actual behaviour, the guard may approve unsafe transitions or reject safe ones.

**Multi-step scalability.** The computational cost of multi-step reachability checking scales with the branching factor of the action space and the horizon depth. Practical deployment of v2/v3 requires approximation strategies that trade exactness for tractability.

**Scope of v1.** The current implementation enforces one-step transition safety only. Delayed violations arising from multi-step trajectories are not detected by v1. This is acknowledged as a known limitation and the primary motivation for the v2/v3 extensions.

**State representation.** The current system operates on explicit symbolic states. Extension to latent or continuous state spaces (e.g. model-internal representations) introduces additional challenges in defining both F and Ω over those spaces.

-----

## 9. Future Work

```
════════════════════════════════════════════════════════════════
 Multi-step implementation
   → Empirical validation of v2 prototypes
   → Approximation strategies for bounded-horizon checking

 Policy-to-Ω compiler
   → Automated translation of institutional policy, law,
     or enterprise rules into forbidden set definitions

 Latent space constraints
   → Transition models operating over latent embeddings
   → Reachability constraints applied within model-internal
     representation spaces

 Architectural integration
   → Guard embedded within model inference pipeline
   → Real-time reachability checking at inference speed

 Formal verification
   → Machine-checkable proofs of invariant preservation
   → Connection to verified control systems literature
════════════════════════════════════════════════════════════════
```

-----

## 10. Conclusion

This work reframes AI safety as a problem of reachability rather than behaviour. By constraining the set of reachable states, unsafe outcomes can be excluded by construction rather than filtered post hoc.

The contribution is a formal framework in which safety is defined as state-space exclusion (Reach⁺(x₀) ∩ Ω = ∅), together with a minimal working implementation (v1) that enforces one-step transition safety as a middleware guard for LLM agents. The framework defines a clear progression from transition-level guards (v1) through bounded-horizon checking (v2) to full trajectory-level invariants (v3).

This is not a claim that AI safety is solved. This is the introduction of a new control layer — and the demonstration of its first implementation.

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║   v1: F(x_t, u_t) ∉ Ω                                               ║
║   v2: Reach(F(x_t, u_t), k) ∩ Ω = ∅                                 ║
║   v3: Reach⁺(F(x_t, u_t)) ∩ Ω = ∅                                   ║
║                                                                      ║
║   Safety is not what you filter from the output.                     ║
║   Safety is what you exclude from the reachable set.                 ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

-----

## Citation

```
Morrison, D. (2026). Structural AI Safety via Reachability Constraints:
From Transition-Level Guards to Trajectory-Level Invariants.
Independent Research. UK Patent Applications: GB2600765.8,
GB2602013.1, GB2602072.7, GB26023332.5.

Available at: https://github.com/davarn/morrison-framework
```

**BibTeX:**

```bibtex
@misc{morrison2026reachability,
  author       = {Morrison, Davarn},
  title        = {Structural AI Safety via Reachability Constraints:
                  From Transition-Level Guards to Trajectory-Level Invariants},
  year         = {2026},
  publisher    = {Independent Research},
  note         = {UK Patent Applications: GB2600765.8, GB2602013.1,
                  GB2602072.7, GB26023332.5},
  url          = {https://github.com/davarn/morrison-framework}
}
```

-----

Structural AI Safety via Reachability Constraints · Morrison Framework™ · Research Paper

GB2600765.8 · GB2602013.1 · GB2602072.7 · GB26023332.5

© 2026 Davarn Morrison — Intelligence Invariant™ · All Rights Reserved

-----

## References

```
[1]  Ames, A.D., Xu, X., Grizzle, J.W. and Tabuada, P. (2017).
     Control Barrier Function Based Quadratic Programs for Safety
     Critical Systems. IEEE Transactions on Automatic Control,
     62(8), pp.3861–3876.

[2]  Aubin, J.-P. (1991). Viability Theory. Birkhäuser.

[3]  Mitchell, I.M., Bayen, A.M. and Tomlin, C.J. (2005).
     A Time-Dependent Hamilton-Jacobi Formulation of Reachable Sets
     for Continuous Dynamic Games. IEEE Transactions on Automatic
     Control, 50(7), pp.947–957.

[4]  Christiano, P.F., Leike, J., Brown, T., Martic, M., Legg, S.
     and Amodei, D. (2017). Deep Reinforcement Learning from Human
     Preferences. Advances in Neural Information Processing Systems, 30.

[5]  Bai, Y., Kadavath, S., Kundu, S., Askell, A., Kernion, J.,
     Jones, A., Chen, A., Goldie, A., Mirhoseini, A., McKinnon, C.,
     et al. (2022). Constitutional AI: Harmlessness from AI Feedback.
     arXiv:2212.08073.

[6]  Bansal, S., Chen, M., Herbert, S. and Tomlin, C.J. (2017).
     Hamilton-Jacobi Reachability: Some Recent Theoretical Advances
     and Applications in Unmanned Airspace Management. Annual Review
     of Control, Robotics, and Autonomous Systems, 2, pp.253–279.

[7]  Amodei, D., Olah, C., Steinhardt, J., Christiano, P.,
     Schulman, J. and Mané, D. (2016). Concrete Problems in AI
     Safety. arXiv:1606.06565.

[8]  Blanchini, F. (1999). Set Invariance in Control. Automatica,
     35(11), pp.1747–1767.

[9]  Morrison, D. (2026). Geometric Control Theory of Cognition:
     A Reachability-Based Framework for Identity, Intelligence,
     and Experience. Independent Research. UK Patent Applications:
     GB2600765.8, GB2602013.1, GB2602072.7, GB26023332.5.
```

-----

## Related Work

- [Morrison Reachability Guard — LLM Agent Governance](https://github.com/davarn/morrison-framework) — Policy-based agent guard (v1 implementation)
- [Morrison Reachability Guard — Universal Agent Plugin](https://github.com/davarn/morrison-framework) — Middleware plugin for any agent framework
- [Morrison Reachability Guard — Runtime Interface](https://github.com/davarn/morrison-framework) — Protocol-based typed interface
- [Morrison Reachability Guard — Architecture Specification](https://github.com/davarn/morrison-framework) — Full spec with theorems
- [Morrison Reachability Guard — Toy Prototype](https://github.com/davarn/morrison-framework) — 2D visualised demo
- [Geometric Control Theory of Cognition](https://github.com/davarn/morrison-framework) — The base theory
- [The Morrison Reality Table](https://github.com/davarn/morrison-framework) — Unified expression of the framework
- [Licensing, Citation, and IP](https://github.com/davarn/morrison-framework) — How to cite and licence
