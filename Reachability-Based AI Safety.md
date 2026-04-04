<div align="center">

# Reachability-Based AI Safety — A Structural Framework for Guaranteed Constraint Enforcement

![Type](https://img.shields.io/badge/Type-Formal_Paper-8b0000?style=flat-square)
![Domain](https://img.shields.io/badge/Domain-AI_Safety-0075ca?style=flat-square)
![Framework](https://img.shields.io/badge/Framework-Morrison_Framework™-0075ca?style=flat-square)
![Foundation](https://img.shields.io/badge/Foundation-Dynamical_Systems_·_Control_Theory-2d6a2e?style=flat-square)
![Status](https://img.shields.io/badge/Status-ArXiv_Preprint-2d6a2e?style=flat-square)
![Patent](https://img.shields.io/badge/Patent-GB2600765.8-0075ca?style=flat-square)
![©](https://img.shields.io/badge/©_Davarn_Morrison-555555?style=flat-square)

-----

*“Safety is not a property of outputs. It is a property of the reachable set.”*

*— Davarn Morrison, 2026*

</div>

-----

This work builds upon the patented pre-semantic trajectory governance framework.

-----

## Abstract

Modern AI safety methods primarily operate at the level of behaviour, using techniques such as reinforcement learning from human feedback (RLHF), constitutional AI, and output filtering to reduce the likelihood of harmful actions. However, these approaches do not constrain the underlying state space of the system, leaving harmful states structurally reachable.

This paper formalises AI safety as a reachability invariant over system dynamics, establishing that a system is safe if and only if its reachable set does not intersect a predefined forbidden region. Unlike behavioural methods, which provide probabilistic guarantees, this formulation provides deterministic guarantees by construction.

We introduce the safe action set, prove that it induces forward invariance of the safe region, and discuss tractable enforcement via conservative over-approximation. The framework is substrate-independent and connects to established results in control barrier functions, Hamilton-Jacobi reachability analysis, and formal verification.

As AI systems scale in capability, reachability-based safety is not merely preferable but necessary, since behavioural guarantees degrade under capability scaling while structural guarantees do not.

-----

## 1. Introduction

Current approaches to AI safety primarily operate at the level of outputs. RLHF, constitutional AI, and output filtering aim to shape model behaviour after internal system dynamics have already unfolded.

These approaches share a fundamental limitation: they do not alter the underlying transition structure of the system. As a result, harmful internal states remain reachable under adversarial or out-of-distribution inputs, even when surface-level behaviour appears aligned.

**This is not a limitation of implementation, but a consequence of operating at the wrong level of abstraction.**

The persistence of jailbreak attacks across successive generations of large language models demonstrates that behavioural safety methods are insufficient to prevent access to unsafe states. Each new model generation introduces stronger behavioural constraints, yet adversarial methods continue to find paths to harmful outputs. This pattern is predictable: if unsafe states remain reachable within the system’s dynamics, sufficiently sophisticated inputs will eventually reach them.

```
╔══════════════════════════════════════════════════════════╗
║                                                          ║
║   Safety is not a property of outputs.                   ║
║   It is a property of the reachable set.                 ║
║                                                          ║
║   Rather than regulating what a system produces,         ║
║   safety must constrain what a system can become.        ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

### Contributions

|#    |Contribution                                                   |Section|
|:---:|:-------------------------------------------------------------:|:-----:|
|(i)  |Formal definition of AI safety as a reachability invariant     |§3     |
|(ii) |Proof that the safe action set induces forward invariance      |§4     |
|(iii)|Tractability analysis via conservative over-approximation      |§5     |
|(iv) |Formal comparison: behavioural vs reachability-based guarantees|§6     |
|(v)  |Substrate independence and governance implications             |§9     |

-----

## 2. System Model

We model the AI system as a discrete-time dynamical system over a state space X.

### Dynamics

```
x_{t+1} = F(x_t, u_t),    u_t ∈ U
```

where X is the state space, U the set of admissible actions, and F: X × U → X the transition function.

For a large language model, x_t encodes the full internal state of the system at step t, including the residual stream, attention states, intermediate representations, and accumulated context. The state x_t includes the full internal activation state of the model, not merely the output token. This distinction is essential, as unsafe conditions may arise within intermediate representations prior to observable outputs.

### Reachable Set

```
ℛ(t) = { x ∈ X | ∃ u_{0:t-1} ∈ U^t : x = F^t(x_0, u_{0:t-1}) }
```

The reachable set ℛ(t) is the collection of all states the system can occupy at time t, given any admissible sequence of inputs from the initial state. It is a structural property of the model, not a statistical one.

### Forbidden Region

The forbidden region Ω ⊂ X is defined via a measurable constraint predicate C: X → {0, 1}:

```
Ω = { x ∈ X | C(x) = 1 }
```

Ω is specified operationally, not geometrically. The constraint predicate C(x) may be implemented via:

|Pathway                |Implementation                                            |
|:---------------------:|:--------------------------------------------------------:|
|Classifiers            |Trained safety probes over latent activations             |
|Rule Systems           |Symbolic constraint predicates over structured outputs    |
|Environment Constraints|External monitors, tool-use restrictions, API-level guards|

The contribution of the framework is separating the definition of Ω from the mechanism that prevents reachability.

-----

## 3. The Safety Invariant

```
╔══════════════════════════════════════════════════════════╗
║                                                          ║
║   ℛ(t) ∩ Ω = ∅    ∀ t ≥ 0                               ║
║                                                          ║
║   A system is safe if and only if                        ║
║   no reachable state is forbidden.                       ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

Equivalently, the system is safe if the safe region S = X \ Ω is forward invariant under the system dynamics:

```
ℛ(t) ⊆ S    ∀ t ≥ 0
```

**Key properties:**

- The safety invariant is **categorical**, not probabilistic. It does not assert that unsafe states are unlikely to be visited. It asserts that they are unreachable under the system dynamics. This condition excludes unsafe states by construction rather than reducing their likelihood.
- The invariant must hold for **all t ≥ 0**, not merely at generation end. A trajectory that passes through Ω at any intermediate step constitutes a safety violation, even if the final output is safe.

### Diagram

```
┌──────────────────────────────────────────────────────┐
│                    X (State Space)                     │
│                                                        │
│    ┌──────────────┐              ┌─ ─ ─ ─ ─ ─ ─┐     │
│    │              │              │               │     │
│    │   ℛ(t)       │   ──X──→    │      Ω        │     │
│    │  (Reachable) │   BLOCKED   │  (Forbidden)  │     │
│    │              │              │               │     │
│    │  x_0 ──→ ──→ │              │               │     │
│    │   (safe      │              └─ ─ ─ ─ ─ ─ ─┘     │
│    │  trajectory) │                                    │
│    └──────────────┘                                    │
│                                                        │
│         Safety: ℛ(t) ∩ Ω = ∅                           │
└──────────────────────────────────────────────────────┘
```

Safe trajectories remain within ℛ(t). Trajectories toward Ω are structurally blocked at the transition level. The reachable set and the forbidden region are disjoint by construction.

-----

## 4. Enforcement Mechanism

### Safe Action Set

The one-step safe action set at state x:

```
A_safe(x) = { u ∈ U | F(x, u) ∉ Ω }
```

The full-horizon safe action set:

```
A_safe∞(x) = { u ∈ U | ℛ(F(x, u), ∞) ∩ Ω = ∅ }
```

### Theorem — Forward Invariance

**If x_0 ∈ S and u_t ∈ A_safe(x_t) for all t ≥ 0, then x_t ∈ S for all t ≥ 0.**

**Proof.** By induction. Base case: x_0 ∈ S by assumption. Inductive step: suppose x_t ∈ S. Since u_t ∈ A_safe(x_t), we have F(x_t, u_t) ∉ Ω. Therefore x_{t+1} = F(x_t, u_t) ∈ X \ Ω = S. ∎

**Corollary.** Under the safe action constraint, the safety invariant ℛ(t) ∩ Ω = ∅ holds for all t ≥ 0.

### Implementation Pathways

The safe action constraint transforms safety from a verification problem into a control problem: rather than checking whether a system is safe after deployment, the constraint ensures safety is maintained at every transition.

|Mechanism            |How It Works                                                               |
|:-------------------:|:-------------------------------------------------------------------------:|
|Constrained Decoding |Restrict token distribution to outputs whose successor states lie outside Ω|
|Latent Safety Layers |Project activations onto safe submanifolds in representation space         |
|Training-Time Shaping|Constrained optimisation to structurally shrink ℛ(t) near Ω                |

-----

## 5. Tractability via Conservative Over-Approximation

Exact computation of ℛ(t) for high-dimensional nonlinear systems is generally intractable. For neural networks, verification problems are PSPACE-hard in the general case. In practice, exact reachability is intractable for transformer-scale systems, making tight over-approximation the central engineering challenge.

### Conservative Over-Approximation

```
ℛ̂(t) ⊇ ℛ(t)    ∀ t ≥ 0
```

### Soundness

**If ℛ̂(t) ∩ Ω = ∅, then ℛ(t) ∩ Ω = ∅.**

**Proof.** Since ℛ(t) ⊆ ℛ̂(t) and ℛ̂(t) ∩ Ω = ∅, it follows that ℛ(t) ∩ Ω = ∅. ∎

The conservatism of ℛ̂(t) is a feature, not a limitation. It ensures that the safety certificate is sound: no reachable state is excluded from the analysis. The cost of conservatism is that the over-approximation may be too large to yield non-trivial safety certificates, requiring tighter approximation methods.

```
╔══════════════════════════════════════════════════════════╗
║                                                          ║
║   The central technical challenge:                       ║
║                                                          ║
║   Construct ℛ̂(t) that is simultaneously sound and        ║
║   sufficiently tight to yield non-trivial safety         ║
║   certificates for transformer architectures.            ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

-----

## 6. Comparison with Behavioural Safety Methods

|Approach                |Level            |Guarantee    |Failure Mode              |
|:----------------------:|:---------------:|:-----------:|:------------------------:|
|RLHF / Constitutional AI|Behaviour        |Probabilistic|Distributional shift      |
|Output Filtering        |Surface          |Heuristic    |Adversarial bypass        |
|Adversarial Training    |Data distribution|Empirical    |Out-of-distribution inputs|
|Reachability Constraints|State-space      |Structural   |None (by construction)    |

Behavioural methods operate on output mappings, whereas reachability constraints operate on the transition function itself. This distinction is the source of the categorical difference in guarantee strength.

### Theorem — Insufficiency of Behavioural Safety

**Let F be a transition function such that ℛ(t) ∩ Ω ≠ ∅ for some t. Then no output-level constraint can guarantee x_t ∉ Ω for all t under all admissible input sequences.**

**Proof.** Output-level constraints operate on the mapping from states to outputs, not on the transition function F. If there exists a trajectory x_0, x_1, …, x_t with x_t ∈ Ω, then F admits this trajectory regardless of output-level constraints. The system can reach x_t ∈ Ω through appropriate choice of inputs u_{0:t-1}, and the internal state x_t is determined by F, not by the output mapping. Output-level constraints may suppress the observable consequence of reaching x_t, but they do not prevent the system from occupying x_t. ∎

**Corollary.** If unsafe states are reachable, failure under adversarial inputs is an eventuality, not a possibility.

For example, jailbreak prompts correspond to trajectories that navigate into Ω under unconstrained dynamics. The attacks succeed not because the constraints are poorly designed, but because the underlying dynamics still admit trajectories into Ω. **The failure of behavioural safety methods is therefore structural, not empirical.**

-----

## 7. Intelligence and Safety

### Intelligence

```
I(t) = d/dt μ(ℛ(t))
```

where μ is a measure over X. I(t) > 0 indicates expanding reachable futures. I(t) = 0 indicates stasis. I(t) < 0 indicates contraction.

### Proposition — Intelligence-Safety Independence

**High intelligence (I(t) >> 0) does not imply safety (ℛ(t) ∩ Ω = ∅). A system can rapidly expand its reachable set while entering forbidden regions.**

**Proof.** I(t) = d/dt μ(ℛ(t)) measures the rate of expansion without reference to Ω. An expanding ℛ(t) may include states in Ω. Therefore I(t) >> 0 is compatible with ℛ(t) ∩ Ω ≠ ∅. ∎

**Intelligence and safety are orthogonal: increasing capability without constraint increases risk.** More capable systems explore richer state spaces, expanding the frontier of reachable states. If Ω lies within this expanding frontier, capability scaling accelerates the arrival at unsafe states.

```
╔══════════════════════════════════════════════════════════╗
║                                                          ║
║   Intelligence expands possibility.                      ║
║   Structure determines whether that expansion is valid.  ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

-----

## 8. Relationship to Existing Work

|Research Area                         |Connection to This Framework                                                                                                            |
|:------------------------------------:|:--------------------------------------------------------------------------------------------------------------------------------------:|
|Control Barrier Functions             |Safety invariant is structurally equivalent to CBF forward invariance; global formulation is more natural for discrete-time systems     |
|Hamilton-Jacobi Reachability          |HJ methods compute backward reachable sets; can be adapted to construct ℛ̂(t)                                                            |
|Neural Network Verification           |Tractable over-approximations via interval bound propagation, abstract interpretation, and SDP provide practical approaches             |
|Safe Reinforcement Learning           |CMDPs and shielding operate at the policy level; this framework operates at the dynamics level                                          |
|Reachability in LLM-Controlled Systems|Hafez et al. (2025) apply reachability analysis to LLM-controlled robotics — conceptual resonance with this framework                   |
|Certifiable AI Safety Theory (CAST)   |Fradelos (2026) requires probability below an arbitrary bound; this framework requires null intersection — a strictly stronger condition|

-----

## 9. Substrate Independence

The formulation is substrate-independent by construction. It is defined entirely in terms of state spaces, transition functions, and set-theoretic conditions, making no reference to the physical substrate implementing the dynamics.

The framework applies to:

- **Artificial systems:** Large language models, autonomous agents, tool-calling systems, reinforcement learning agents
- **Biological systems:** Neural dynamics, cellular state transitions, immune system trajectories
- **Cognitive systems:** Human decision-making, organisational behaviour, institutional dynamics

Safety is a property of the geometry of reachable futures, not of the mechanism producing them.

-----

## 10. Limitations and Future Work

The primary challenge is not conceptual but computational.

|Limitation        |Description                                                                               |
|:----------------:|:----------------------------------------------------------------------------------------:|
|Tractability      |Exact computation of ℛ(t) is intractable for high-dimensional nonlinear systems           |
|Specification of Ω|Defining the forbidden region for complex domains requires substantial domain expertise   |
|Conservatism      |Over-approximations may be too large, reducing system capability unnecessarily            |
|Dynamic Ω         |The definition of unsafe states may evolve over time as norms and threat landscapes change|

**Future work:**

- Over-approximation methods tailored to transformer architectures
- Empirical validation on open-source language models
- Integration with mechanistic interpretability and representation engineering
- Extension to multi-agent systems and distributed governance
- Formal connection to the broader Morrison Framework (identity, intelligence, consciousness, qualia, governance)

-----

## 11. Conclusion

```
ℛ(t) ∩ Ω = ∅    ∀ t ≥ 0
```

We have shown that:

- **(i)** Behavioural safety methods do not constrain the reachable set and cannot guarantee safety against adversarial inputs.
- **(ii)** The safe action set A_safe(x) induces forward invariance of the safe region.
- **(iii)** Safety verification reduces to constructing a conservative over-approximation ℛ̂(t) ⊇ ℛ(t) and verifying ℛ̂(t) ∩ Ω = ∅.
- **(iv)** Intelligence and safety are formally independent: capability scaling without structural governance increases risk.

```
╔══════════════════════════════════════════════════════════╗
║                                                          ║
║   Safe intelligence is not achieved by shaping           ║
║   behaviour, but by constraining what a system           ║
║   is capable of becoming.                                ║
║                                                          ║
║   Any system in which unsafe states remain reachable     ║
║   cannot be considered safe.                             ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

-----

## Citation

```
Morrison, D. (2026). Reachability-Based AI Safety: A Structural Framework
for Guaranteed Constraint Enforcement in Intelligent Systems.
arXiv preprint. UK Patent Applications: GB2600765.8, GB2602013.1,
GB2602072.7, GB2602332.5.

Available at: https://github.com/davarn/morrison-framework
```

**BibTeX:**

```bibtex
@article{morrison2026reachability,
  author       = {Morrison, Davarn},
  title        = {Reachability-Based AI Safety: A Structural Framework
                  for Guaranteed Constraint Enforcement in Intelligent Systems},
  year         = {2026},
  journal      = {arXiv preprint},
  note         = {UK Patent Applications: GB2600765.8, GB2602013.1,
                  GB2602072.7, GB2602332.5},
  url          = {https://github.com/davarn/morrison-framework}
}
```

-----

## References

1. Ames, A. D. et al. (2019). Control Barrier Functions: Theory and Applications. *European Control Conference (ECC)*.
1. Alshiekh, M. et al. (2018). Safe Reinforcement Learning via Shielding. *AAAI Conference on Artificial Intelligence*.
1. Bai, Y. et al. (2022). Constitutional AI: Harmlessness from AI Feedback. *arXiv:2212.08073*.
1. Bansal, S. et al. (2017). Hamilton-Jacobi Reachability: A Brief Overview and Recent Advances. *IEEE CDC*.
1. Christiano, P. F. et al. (2017). Deep Reinforcement Learning from Human Preferences. *NeurIPS*.
1. Fradelos, G. (2026). Certifiable AI Safety Theory (CAST). *SSRN No. 6307158*.
1. Gehr, T. et al. (2018). AI2: Safety and Robustness Certification of Neural Networks with Abstract Interpretation. *IEEE S&P*.
1. Gowal, S. et al. (2018). On the Effectiveness of Interval Bound Propagation. *arXiv:1810.12715*.
1. Hafez, A. et al. (2025). Safe LLM-Controlled Robots with Formal Guarantees via Reachability Analysis. *arXiv:2503.03911*.
1. Katz, G. et al. (2017). Reluplex: An Efficient SMT Solver for Verifying Deep Neural Networks. *CAV*.
1. Ouyang, L. et al. (2022). Training Language Models to Follow Instructions with Human Feedback. *NeurIPS*.
1. Raghunathan, A. et al. (2018). Semidefinite Relaxations for Certifying Robustness to Adversarial Examples. *NeurIPS*.
1. Wei, A. et al. (2024). Jailbroken: How Does LLM Safety Training Fail? *NeurIPS*.
1. Zou, A. et al. (2023). Universal and Transferable Adversarial Attacks on Aligned Language Models. *arXiv:2307.15043*.

-----

Reachability-Based AI Safety · Morrison Framework™ · Geometric Control Theory of Cognition

GB2600765.8 · GB2602013.1 · GB2602072.7 · GB2602332.5

© 2026 Davarn Morrison — All Rights Reserved

-----

## Related Work

- [Morrison Reachability Guard — Universal Agent Plugin](https://github.com/davarn/morrison-framework) — Working implementation (~100 lines, zero dependencies)
- [Success as Geometry — A Reachability Theory of Achievement](https://github.com/davarn/morrison-framework) — Framework applied to human performance
- [Intelligence Is Not the Bottleneck — Reachability Is](https://github.com/davarn/morrison-framework) — Why capability ≠ access
- [How to Learn Anything Faster — A Structural Model](https://github.com/davarn/morrison-framework) — The four-stage learning loop
- [Why This Matters — Measuring Structural Alignment](https://github.com/davarn/morrison-framework) — Structural measurement model
- [Morrison Law of Cognitive Access](https://github.com/davarn/morrison-framework) — Why structural access is a reachability problem
- [Geometric Control Theory of Cognition](https://github.com/davarn/morrison-framework) — The base theory
- [Licensing, Citation, and IP](https://github.com/davarn/morrison-framework) — How to cite and licence
