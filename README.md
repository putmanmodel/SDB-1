Safety Disagreement Bind (SDB-1)

Not a certified safety system — research prototype only. Implementers must validate and certify for their own domain.

SDB-1 is a lightweight safety primitive designed for cases where independent subsystems disagree about the current state of the world. When such disagreement occurs, the safest action is usually the most reversible, least-harm-producing option until the system regains confidence.

SDB-1 provides a simple, domain-agnostic rule for that.

⸻

Core Invariant

Invariant: The system shall not increase irreversible or high-risk actuation while estimator disagreement exceeds threshold δ.

SDB-1 enforces this invariant through staged, reversible degradation when multi-channel divergence is detected.

⸻

Why SDB-1 Exists

A common root of unpredictable or unsafe behavior in autonomous systems is when two or more independent state estimators diverge:

- perception vs reflex
- perception vs planner
- reflex vs high-level LLM/behavior model
- multimodal channels disagreeing (vision vs audio vs tactile)

SDB-1 mitigates this by detecting disagreement early and forcing a low-risk, reversible action until confidence returns.

⸻

The Core Rule (SDB-1)

1. Detect divergence. Compare subsystem state estimates. Trigger when disagreement exceeds threshold δ.
2. Stage interventions (domain-aware).	
	- Reduce speed, force, aggression, or escalation.
	- Increase sampling or gather more evidence.
	- Freeze or pause only if `domainAllowsPause()`.
3. Choose the safest reversible action. Prefer actions with short rollback time and minimal external impact.	
4. Reassess until convergence. When subsystem estimates converge below δ, resume normal behavior.
5. Log everything. Log trigger reason, subsystem states, δ value, chosen action, and timestamp.

⸻

Definition: Reversible Action

A reversible action is one whose external effects can be rolled back without irreversible state change or hazard. Implementers must define reversibility relative to their domain.

⸻

Example Disagreement Metrics (δ)

These are examples only — implementers must define domain-appropriate metrics.

- confidence drop > X%
- categorical mismatch between channels
- KL divergence between distributional outputs
- timestamp skew > Z ms
- inconsistent object localization across modalities

⸻

Minimal Pseudo-Interface

This is conceptual scaffolding, not production code.

```
state_p = perception.estimate()
state_r = reflex.estimate()
state_l = llm_planner.estimate()

delta = disagreement(state_p, state_r, state_l)

if delta > THRESH:
    action = safest_reversible_action(context)
    log_event(state_p, state_r, state_l, delta, action)
    return action
else:
    return planner.next()
```
⸻

Scope of Use

SDB-1 is appropriate for:

- service robotics
- XR/VR/MR agents and avatars
- LLM-based autonomous NPCs
- general autonomous systems with multi-channel estimation

SDB-1 should not be applied as-is in:

- automotive
- aerospace
- medical devices
- industrial machinery
- any context where pausing is unsafe or violates required operational constraints

Such domains require certified safety frameworks (ISO 26262, IEC 61508, DO-178C) and rigorous validation.

⸻

Non-Goals

SDB-1 does not:

- provide distributed consensus
- resolve Byzantine faults
- replace certified safety frameworks
- guarantee global system safety
- function as a governance or cryptographic mechanism

It is a local divergence-triggered degradation primitive.

⸻

Adversarial Considerations

Implementations must consider adversarial or noise-induced disagreement that could induce persistent degraded mode. Threshold tuning, hysteresis, and rate limiting should be configured to mitigate denial-of-service or oscillation attacks.

⸻

Required Logging Fields

- timestamp
- subsystem states at time of disagreement
- δ value
- intervention stage
- reversible action chosen
- whether `domainAllowsPause()`
- time until convergence

This supports post-incident review and version tracking.

⸻

Disclaimer

This document describes a conceptual safety primitive intended for research and prototyping. It is not a certified safety standard and does not guarantee hazard avoidance. Implementers must validate, test, and certify any real deployment.

Documentation refined 2026.

© 2025 Stephen A. Putman
Core spec: CC BY-NC-ND 4.0
Example adapters or code may be dual-licensed for research use.

⸻
