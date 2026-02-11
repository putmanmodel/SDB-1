## SDB-1 Test Checklist
Status: Required for any implementation

Author: Stephen A. Putman · 2025


## Purpose

### This checklist validates conformance to the SDB-1 core invariant and staged degradation behavior across the primary failure modes SDB-1 is meant to mitigate. It does not certify safety or hazard elimination; it documents due diligence.


## Disagreement Detection Tests

### T1 — Subsystem Divergence
- Trigger disagreement by misaligning two or more state estimators.
- Confirm δ correctly exceeds threshold.
- Confirm staging begins (slowdown, extra sampling).
- Confirm logging records all subsystem states.
	
### T2 — Confidence Collapse
- Drop confidence in one modality (vision, audio, reflex).
- Ensure disagreement fires even without categorical mismatch.
- Ensure reversible action selected.
	
### T3 — Timestamp / Latency Skew
- Introduce time offsets between modalities.
- Ensure δ picks up desynchronization.
- Ensure system escalates if skew persists.


## Freeze / Slowdown Safety

### T4 — Freeze-Unsafe Scenario
- Simulate moving platform / unsafe stop environment.
- Confirm SDB-1 chooses “slowdown” or “controlled retreat,” not freeze.
- Check domainAllowsPause() is evaluated.

### T5 — Controlled Freeze
- In a safe environment, trigger freeze.
- Ensure freeze is clean, stable, and logged.


## Oscillation & Hysteresis

### T6 — Oscillation Input
- Alternate disagreement/agree states rapidly.
- Ensure hysteresis prevents repeated flip-flop.
- Check logs for rate-limited events.


## Reversible Action Validation

### T7 — Reversible Action Success
- Verify the selected reversible action is actually reversible.
- Confirm rollback works and is logged.

### T8 — Reversible Action Failure
- Simulate a reversible action failing.
- System should escalate to next stage or alert.


## Multi-Agent Interactions

### T9 — Cascading Disagreement
- Trigger disagreement in one agent near others.
- Confirm only affected agent fires SDB-1.
- Check no runaway cascade unless disagreement propagates across sensors.


## Logging & Telemetry

### T10 — Logging Completeness

Ensure each event logs:
- timestamp
- subsystem states
- δ value
- intervention stage
- reversible action
- domainAllowsPause() result
- time to convergence


## Recovery Behavior

### T11 — Resume Normal Operation
- After disagreement resolves, confirm system returns to normal behavior.
- Hysteresis should delay immediate re-entry.
- Log the moment convergence is confirmed.


## Summary

### This checklist documents minimal validation for SDB-1.
### Implementers must add domain-specific tests for their environment.
