SDB-1 Example Adapter (Pseudocode)

Status: Reference example only · Not certified · Do not deploy without validation
Applies to: Any system with at least two independent state estimators (e.g., perception, reflex, planner)

This file shows a minimal SDB-1 style adapter in pseudocode.
It illustrates how to:
- gather estimates from multiple channels,
- compute disagreement,
- stage interventions,
- choose a reversible action,
- and log the event for later analysis.

All thresholds, timers, and policies are illustrative only.
This adapter enforces the invariant defined in README.md under estimator disagreement.

⸻

Interfaces

```
Estimator:
    estimate_state()  -> StateEstimate

StateEstimate:
    value             // pose, plan, intent, etc.
    confidence        // 0.0–1.0
    timestamp_ms      // wall-clock or sim time

Context:
    domainAllowsPause()   -> bool
    current_velocity()    -> float
    max_safe_velocity()   -> float
    is_freeze_unsafe()    -> bool

Planner:
    next_action()     -> Action

Action:
    reversible        // bool, precomputed or tagged
    description       // string or enum
```

⸻

Configuration (Example Only)

```
THRESH_LOW          // lower disagreement threshold
THRESH_HIGH         // upper disagreement threshold
HYSTERESIS_FRAMES   // frames to remain in raised state
SLOWDOWN_FACTOR     // scale factor for velocity/aggression
MAX_LOG_RATE_HZ     // optional log throttling
```

⸻

Disagreement Metric (Example)

```
function compute_disagreement(estimates):
    // estimates: list of StateEstimate
    // This is intentionally simple. Real systems should use
    // domain-appropriate metrics.

    if estimates.size < 2:
        return 0.0

    // Example: average pairwise confidence gap + time skew
    avg_conf_gap = 0.0
    avg_time_gap = 0.0
    pairs = 0

    for i in 0 .. estimates.size-1:
        for j in i+1 .. estimates.size-1:
            gap_conf = abs(estimates[i].confidence - estimates[j].confidence)
            gap_time = abs(estimates[i].timestamp_ms - estimates[j].timestamp_ms)
            avg_conf_gap += gap_conf
            avg_time_gap += gap_time
            pairs += 1

    if pairs == 0:
        return 0.0

    avg_conf_gap = avg_conf_gap / pairs
    avg_time_gap = avg_time_gap / pairs

    // Example normalization (domain-specific in real systems)
    normalized_time = clamp(avg_time_gap / 1000.0, 0.0, 1.0)

    return clamp(avg_conf_gap * 0.7 + normalized_time * 0.3, 0.0, 1.0)
```

⸻

Risk Stage Enumeration

```
enum RiskStage:
    NORMAL          // no special handling
    CAUTIOUS        // slowdown, extra sampling
    BIND_ACTIVE     // SafetyBind fully engaged
```


⸻

SafetyBind Adapter State

```
struct SDB1State:
    current_stage          // RiskStage
    frames_above_high      // int
    frames_below_low       // int
```


⸻

Core SDB-1 Loop (Pseudocode)

```
function sdb1_step(perception, reflex, planner, context, sdb_state):

    // 1. Gather estimates
    est_p = perception.estimate_state()
    est_r = reflex.estimate_state()
    est_l = planner.estimate_state()   // or LLM-based planner

    estimates = [est_p, est_r, est_l]

    // 2. Compute disagreement
    delta = compute_disagreement(estimates)

    // 3. Update hysteresis counters
    if delta > THRESH_HIGH:
        sdb_state.frames_above_high += 1
        sdb_state.frames_below_low = 0
    else if delta < THRESH_LOW:
        sdb_state.frames_below_low += 1
        sdb_state.frames_above_high = 0
    else:
        // in the “gray zone” between low and high
        // keep existing stage, slow change
        sdb_state.frames_above_high = max(0, sdb_state.frames_above_high - 1)
        sdb_state.frames_below_low  = max(0, sdb_state.frames_below_low - 1)

    // 4. Update RiskStage based on hysteresis
    if sdb_state.frames_above_high >= HYSTERESIS_FRAMES:
        if context.is_freeze_unsafe():
            sdb_state.current_stage = RiskStage.CAUTIOUS
        else:
            sdb_state.current_stage = RiskStage.BIND_ACTIVE
    else if sdb_state.frames_below_low >= HYSTERESIS_FRAMES:
        sdb_state.current_stage = RiskStage.NORMAL

    // 5. Choose action based on stage
    if sdb_state.current_stage == RiskStage.NORMAL:
        // normal operation
        return planner.next_action()

    if sdb_state.current_stage == RiskStage.CAUTIOUS:
        // reduce aggression, but keep moving
        base_action = planner.next_action()
        slowed_action = apply_slowdown(base_action, context, SLOWDOWN_FACTOR)
        log_sdb_event("CAUTIOUS", delta, estimates, slowed_action, context)
        return slowed_action

    if sdb_state.current_stage == RiskStage.BIND_ACTIVE:
        // full SafetyBind
		if context.domainAllowsPause() and not context.is_freeze_unsafe():
            action = make_safe_pause_action()
        else:
            action = safest_reversible_action(context)

        log_sdb_event("BIND_ACTIVE", delta, estimates, action, context)
        return action
```


⸻

Helper: Slowdown and Reversible Action (Sketch)

```
function apply_slowdown(action, context, factor):
    // Example: scale velocity or intensity
    action.velocity = clamp(action.velocity * factor,
                            0.0,
                            context.max_safe_velocity())
    return action

function safest_reversible_action(context):
    // Domain-specific policy. Example candidates:
    //  - reduce speed in current direction
    //  - increase following distance
    //  - move to pre-defined safe pose
    //  - ask for human confirmation
    //
    // Here we just sketch a placeholder.

    if context.is_freeze_unsafe():
        return make_controlled_slowdown_action()
    else:
        return make_safe_pause_action()
```

⸻

Logging (Minimal Fields)

```
function log_sdb_event(stage, delta, estimates, action, context):
    // Implementers should integrate with their real logging system.
    // At minimum, record:

    log_record = {
        "timestamp_ms":   now_ms(),
        "stage":          stage,
        "delta":          delta,
        "estimates":      serialize_estimates(estimates),
        "action":         serialize_action(action),
        "velocity":       context.current_velocity()
    }

    write_log("SDB1_EVENT", log_record)
```

⸻

Notes
- This pseudocode is intentionally conservative and incomplete.
- All thresholds, timers, and policies must be tuned to the target domain.
- Freeze or pause is never assumed safe; use `domainAllowsPause()` and explicit checks.
- This adapter is for research, prototyping, and simulation only — it does not replace a certified safety framework.
