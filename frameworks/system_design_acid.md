# Case Study & System Design Framework: ACID

Use this rubric to structure technical case studies and high-level infrastructure design rounds. It forces an upfront focus on physical boundaries before proposing architectural decisions.

## A — Assumptions
*   **Operational Scale:** Define the total footprint upfront (e.g., total compute cluster size, node counts, or throughput requirements).
*   **Workload Profiles:** Establish read/write ratios, peak-to-average load factors, and typical data payload sizes.
*   **Data Growth:** Project short-term and multi-quarter data accumulation vectors.

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
