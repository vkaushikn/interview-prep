# Incident Response & Troubleshooting Framework: ACID + HVAC

A structured, non-linear debugging protocol designed to handle massive-scale distributed system failures under tight operational timelines.

## Phase 1: Core Triage (ACID)

### A — Analyze
*   **Telemetry Inspection:** Pull high-level dashboards to isolate explicit failure symptoms.
*   **Signature Mapping:** Look for anomalies in system logs (e.g., resource starvation, steep latency spikes, or thread pool exhaustion).

### C — Compartmentalize
*   **Triage Fork:** Run a sharp isolation pivot to cut the search space in half:
    *   *Is this an Input Data Issue?* (Corrupted payloads, broken upstream data contracts, schema drifts).
    *   *Is this a System Engine Issue?* (Code bugs, deadlocks, infrastructure regressions, cluster network degradation).

### I — Investigate (Via the HVAC Loop)
Execute this sub-routine once the high-level domain is isolated:
*   **H (Hypothesis):** Formulate a logical root-cause theory grounded in system mechanics (e.g., PID controller oscillation loops, or a multi-node race condition).
*   **V (Validation):** Test the hypothesis against telemetry baselines or a known previous stable deployment version.
*   **A (Alternative):** If validation refutes the theory, pivot cleanly to a secondary hypothesis without lingering on the initial assumption.
*   **C (Confirmation):** Lock in the verified root cause by aligning precise transactional error timestamps across downstream dependencies.

### D — Design Mitigation & Fixes
*   **Tactical Action:** Deploy an immediate short-term mitigation hack to stabilize the environment and protect SLAs (e.g., feature flagging, rolling back a bad config, or spinning up temporary headroom).
*   **Strategic Action:** Design and track a long-term architectural code fix to fully eliminate the root failure pattern.

---

## Open polish items
- Add a scope/blast-radius triage axis (one region/shard/segment vs. everything) — likely faster than the input-vs-engine fork at massive scale, and drives urgency/escalation.
- Add an explicit communication/escalation step (declaring severity, looping in stakeholders) — independent of root-cause progress, and often weighted heavily for senior responders.
- Close the loop after D: verify the mitigation against the original telemetry baseline, and capture a postmortem/prevention item (what monitoring/guardrail would catch this earlier).
- Consider renaming to avoid the ACID acronym collision with the System Design framework — same letters, different meanings, could read as confusing if both come up in one conversation.
