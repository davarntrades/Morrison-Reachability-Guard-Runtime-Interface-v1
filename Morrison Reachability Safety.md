<div align="center">

# Morrison Reachability Safety — Structural Constraint Framework

![Morrison Framework™](https://img.shields.io/badge/Morrison_Framework™-Reachability_Safety-0075ca?style=flat-square)
![Safety](https://img.shields.io/badge/Safety-ℛ(t)_∩_Ω_=_∅-0075ca?style=flat-square)
![Substrate](https://img.shields.io/badge/Substrate-Geometry_Not_Language-2e7d32?style=flat-square)
![Constraint](https://img.shields.io/badge/Constraint-Transition--level_Constraint-8b0000?style=flat-square)
![Patent](https://img.shields.io/badge/Patent-GB2600765.8-0075ca?style=flat-square)
![©](https://img.shields.io/badge/©_2026-Davarn_Morrison-555555?style=flat-square)

*“A constraint on what the system can become, not what it produces.”*

*— Davarn Morrison, 2026*

-----

</div>

This work builds upon the patented pre-semantic trajectory governance framework.

## Overview

This repository presents a reachability-based framework for safety in dynamical systems.

The central claim: safety is a property of the reachable set ℛ, not of outputs.

The framework does not assume exact knowledge of the reachable set; it operates under conservative approximations sufficient to exclude Ω.

-----

## Core Invariant

|Invariant|Name                          |Statement       |
|:-------:|:----------------------------:|:--------------:|
|**I4**   |**Morrison Safety Invariant™**|**ℛ(t) ∩ Ω = ∅**|

|Symbol|Meaning                         |
|:----:|:------------------------------:|
|ℛ(t)  |Reachable set of system states  |
|Ω     |Set of forbidden (unsafe) states|
|∩     |Set intersection                |
|∅     |Empty set                       |

-----

## Key Shift

|             |Standard Approach                   |This Framework                                                 |
|:-----------:|:----------------------------------:|:-------------------------------------------------------------:|
|**Objective**|Reduce probability of unsafe outputs|Ensure exclusion of unsafe states under the given approximation|
|**Level**    |Behaviour-level control             |State-space constraint                                         |
|**Method**   |Post-hoc filtering                  |Constraint at the level of the transition function T           |

-----

## Definitions (LLM Case)

### State

A state is any representation sufficient to determine system transitions — e.g. internal activations, output tokens, or external system context — depending on the modelling level.

### Reachable Set ℛ

The system dynamics are given by x_{t+1} = F(x_t, u_t), where u_t ∈ 𝒰.

The set of all states reachable under admissible inputs (i.e. inputs permitted by the system interface and deployment constraints) and model dynamics. ℛ is induced by repeated application of the transition function F under admissible inputs.

|Equation                                                                 |Definition                                |
|:-----------------------------------------------------------------------:|:----------------------------------------:|
|**ℛ = { s ∈ S | s is reachable via admissible input sequences under θ }**|The reachable set under model parameters θ|

Transitions are governed by a function T(s, a), where a represents admissible inputs or actions. 𝒰 denotes the set of admissible inputs defined by system interface and deployment constraints.

In practice, ℛ is approximated via conservative over-approximations ℛ̂ that include all states reachable under admissible inputs, such that ℛ̂ ⊇ ℛ. Safety guarantees are defined relative to ℛ̂. Exact computation of ℛ is intractable in high-dimensional systems. The system may be partially observable; therefore, constraints may operate over learned or projected representations of state, rather than full system state.

### Forbidden Set Ω

Ω is defined operationally via constraint functions or classifiers that map states to violation labels under a specified policy or safety specification.

|Operationalisation                    |Example                                        |
|:------------------------------------:|:---------------------------------------------:|
|Classifiers                           |Learned models identifying unsafe states       |
|Rule-based systems                    |Explicit constraint functions over trajectories|
|Environment-defined failure conditions|Domain-specific safety specifications          |

Ω is not assumed to be smooth, convex, or geometrically simple. It may be sparse, disconnected, or polysemantic. Ω need not correspond to a contiguous or metrically separable region in latent space; it may be defined over sparse or distributed features and need only be operationally identifiable.

The framework separates the specification of Ω from the enforcement mechanism that ensures it is not reachable.

-----

## Addressing Key Questions

### 1. Is ℛ computable?

No. Exact computation is intractable in high-dimensional systems.

The framework relies on conservative approximations ℛ̂ that over-approximate reachable states in a way that allows conservative safety guarantees. Safety guarantees are defined relative to ℛ̂, not exact ℛ.

The framework does not require exact knowledge of ℛ; it requires that the approximation used is sufficient to ensure that Ω is excluded from the reachable set under all admissible trajectories.

### 2. What is Ω in practice?

Ω is not a latent “region” with clean geometry. It is a set of states defined by constraint functions over trajectories or outcomes. This allows Ω to be sparse, disconnected, or polysemantic.

Ω is domain-dependent. The framework is not. Once Ω is specified, safety becomes a structural question of reachability.

### 3. Topology vs Metric

|Layer   |Role                                                                               |
|:------:|:---------------------------------------------------------------------------------:|
|Topology|Defines the safety condition — non-intersection of ℛ and Ω                         |
|Metrics |Required to construct computable approximations and enforce constraints in practice|

### 4. Is it actually guaranteed?

The invariant defines a sufficient condition for safety. In practice, guarantees are conditional on the fidelity of ℛ̂ and correctness of Ω specification.

### 5. How is this different from alignment?

Alignment shapes behaviour. This constrains possibility. Alignment reduces bad outputs. This removes the trajectories that produce them.

-----

## Constraint Mechanism

The goal is not to eliminate unsafe outputs after they occur, but to eliminate the existence of trajectories that produce them.

At a structural level, constraints are applied over an approximation ℛ̂ such that for all s ∈ ℛ̂ and admissible inputs a ∈ A_adm, the transition T(s, a) does not enter Ω. Guarantees are defined relative to ℛ̂, not exact ℛ. This can be viewed as defining a constrained transition system T_safe, where admissible transitions are restricted to those that do not enter Ω.

|Equation                               |Definition                                                                 |
|:-------------------------------------:|:-------------------------------------------------------------------------:|
|**A_safe(s) = { a ∈ A | T(s, a) ∉ Ω }**|Safe action set — only transitions that cannot reach Ω                     |
|**∀s ∈ ℛ̂, ∀a ∈ A_adm: T(s, a) ∉ Ω**    |Universal constraint over approximation — no admissible transition enters Ω|

|Mechanism                         |Level         |Effect                                      |
|:--------------------------------:|:------------:|:------------------------------------------:|
|Constrained decoding              |Output        |Prune token paths estimated to lead to Ω    |
|Safety layers in latent space     |Representation|Enforce invariant boundaries on activations |
|Training-time trajectory penalties|Learning      |Shape ℛ during optimisation so Ω is excluded|

-----

## Minimal Implementation (Tool Boundary)

A minimal instantiation of the framework can be applied at the action layer:

```python
if forbidden.contains(next_state):
    reject_action()
else:
    execute()
```

This enforces: F(x_t, u_t) ∉ Ω

This enforces a one-step constraint and does not guarantee exclusion of Ω under multi-step trajectories. It serves as a minimal instantiation of the framework, not a complete realisation of multi-step reachability.

One-step constraints are not sufficient in general; safety requires considering multi-step reachability to ensure trajectories do not eventually enter Ω.

-----

## Scope of Claim

The framework does not claim exact computation of ℛ. It does not require Ω to have simple geometric structure. It does not guarantee absolute safety under imperfect approximation.

The contribution is: defining safety as a reachability property and specifying the conditions under which it can be enforced.

-----

## Limitations

|Limitation                      |Detail                                                      |
|:------------------------------:|:----------------------------------------------------------:|
|**ℛ must be approximated**      |Exact computation is intractable in high-dimensional systems|
|**Ω is specification-dependent**|Guarantees are conditional on specification quality         |
|**Approximation fidelity**      |Guarantees are conditional on the tightness of ℛ̂            |
|**Partial observability**       |Constraints may need to operate over observable projections |

-----

## Relation to Existing Work

This framework is compatible with mechanistic interpretability, representation engineering, sparse autoencoders, and control-theoretic safety methods.

|                    |Existing Approaches      |This Framework               |
|:------------------:|:-----------------------:|:---------------------------:|
|**Operates on**     |Behaviour or probability |Reachability and possibility |
|**Safety condition**|Reduce likelihood of harm|Exclude harm from state space|

The distinction is at the level of problem formulation: existing methods operate on behaviour or representations, while this framework defines safety as a property of reachability in system dynamics.

-----

## Current Status

|Status                        |Detail                                      |
|:----------------------------:|:------------------------------------------:|
|Formal invariant              |Defined                                     |
|Minimal runtime implementation|Illustrative minimal implementation provided|
|Approximation and scaling     |Open problems                               |

-----

The framework shifts safety from a probabilistic property of outputs to a structural property of system dynamics.

The framework defines the condition under which safety is structurally possible, rather than providing a complete solution to achieving it.

-----

## Citation

Morrison, D. (2026). *Geometric Control Theory of Cognition: A Reachability-Based Framework for Identity, Intelligence, and Experience.* Independent Research. UK Patent Applications: GB2600765.8, GB2602013.1, GB2602072.7, GB2602332.5.

Available at: <https://github.com/davarn/morrison-framework>

### BibTeX

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

<div align="center">

Morrison Reachability Safety · Morrison Framework™ · Structural Constraint Framework

GB2600765.8 · GB2602013.1 · GB2602072.7 · GB2602332.5

© 2026 Davarn Morrison — Intelligence Invariant™ · All Rights Reserved

</div>
