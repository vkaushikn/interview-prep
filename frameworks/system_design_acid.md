# Case Study & System Design Framework: ACID

Use this rubric to structure technical case studies and high-level infrastructure design rounds. It forces an upfront focus on physical boundaries before proposing architectural decisions.

## Step 0 (Business-Case / OR interviews): The Decomposition Fork

Before starting A, for problems where a large coupled optimization is visible on first reading the prompt (often *why* the interviewer chose it) — name it, and explicitly ask whether the interviewer wants to go deep on that formulation, or explore whether/how it decomposes into something more tractable to implement.

This is a **dual-purpose move**, not just an analytical one:
1. It proves you *see* the large formulation — so choosing not to build it isn't a capability gap, it's a judgment call you're making visibly.
2. It **calibrates the interview** — "I want to see your OR chops, build the model" vs. "I want to see something that could actually ship" are different interviews requiring different 40 minutes, and guessing wrong is a *severe* failure mode (not just lost time — it actively demonstrates the opposite of what that interviewer is screening for).

Asking this fork early converts a blind, high-stakes guess into information, and reads as senior judgment regardless of which way the interviewer answers.

**Why this is front-loaded for OR/business-case problems specifically, but not System Design:** in System Design, decomposition tends to *emerge* through A/C/I as a consequence (e.g., a celebrity hybrid, a geographic cluster). In OR/business-case problems, the choice between "one global coupled model" and "a decomposed/pragmatic system" is often *the* central modeling decision, and it's usually visible immediately — worth surfacing before sinking effort into either path.

## A — Assumptions & Approximations
Two distinct activities, both belonging to this first step:

*   **Assumptions (scope/model simplifications):** Define what's in and out of scope before designing — e.g., which features are included, which are explicitly deferred (recommendation systems, infinite scroll, etc.), and what the simplified model of the problem looks like.
*   **Approximations (sizing the problem):** Back-of-envelope / Fermi estimates of scale — total users, throughput (QPS), read/write ratios, peak-to-average load factors, typical payload sizes, and the *shape* of key distributions (e.g., is a relevant population roughly uniform, or does it have long-tail outliers?).
*   **Data Growth:** Project short-term and multi-quarter data accumulation vectors.

Sizing isn't optional — without rough numbers, C/I/D can't be evaluated concretely (e.g., whether a design choice is "good" often depends entirely on the scale and distribution shape assumed here).

## C — Constraints
*   **Physical Bounds:** Define hardware limitations (e.g., total available GPU/TPU memory, node-to-node interconnect bandwidth, or network fabric topology).
*   **Operational SLAs:** Identify strict SLA boundaries, tail latency budgets (P99 or P99.9 metrics), and hard execution memory ceilings.
*   **Resource Availability:** Note compute supply delays, provisioning leads, or cluster allocation limits.

## I — Incentives & Trade-offs
*   **Organizational Conflict:** Isolate conflicting motivations between teams (e.g., infrastructure engineering optimizing for cost/utilization vs. product teams demanding immediate elastic capacity).
*   **Architectural Pivots:** Explicitly weigh fundamental trade-offs:
    *   Speed vs. Absolute Consistency
    *   High Cluster Utilization vs. Failover Headroom
    *   Algorithmic Complexity vs. System Maintainability

## D — Decisions
*   **Technical Blueprint:** Map the constraints into concrete production designs.
*   **Control Mechanisms:** Propose specific execution loops (e.g., a real-time PID controller checking fast-access caching against an offline hindsight optimization target curve).
*   **State & Storage:** Define the data storage layer matching the input workload criteria.

---

## Open polish items
- Add an explicit translation step between "I" and "D" — force the design to land on named components (sharding key, replication factor, cache layer, queue, consistency model), not just principles.
- Add a failure/reliability lens (node failure, network partition, degraded mode) — currently only implicit in the "Speed vs. Absolute Consistency" trade-off.
- Consider splitting "Decisions" into the architecture itself vs. a closing validation/iteration step (what would you measure post-launch to confirm the design works).
