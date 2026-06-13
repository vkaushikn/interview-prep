# Case Study: Azure Multi-Tier Compute Capacity Allocation

## 1. The Core Prompt & Problem Statement
> **Date:** June 11, 2026
> **Scenario:** Azure has a specific, finite capacity of physical compute. However, incoming customers come from wildly different operational and economic classes:
> 1. **High-Priority Customers:** Users who hold strict Reserved Instances (RIs) with contractual SLAs.
> 2. **Big Whale Customers:** High-revenue enterprise accounts driving massive baseline financial value.
> 3. **Smaller Paying Customers:** On-demand or pay-as-you-go users with fluctuating workloads.
> 4. **Free Tier Users:** Low-margin or trial accounts that absorb excess capacity but must be throttled first.
>
> **The Challenge:** How do we design an automated system to allocate available capacity across these conflicting tiers under tight physical resource constraints?

---

## 2. Applying the ACID Design Framework

To design this allocation system systematically without breaking down into ad-hoc logic, we apply the **ACID** mnemonic:

### A — Assumptions
*   **Capacity Pools:** Assume total regional capacity ($C_{total}$) is split into hard-reserved blocks for RIs and an elastic shared pool for on-demand/free tiers.
*   **Workload Predictability:** High-priority and Whale customers have predictable baseline usage curves, while smaller paying and free-tier users present highly volatile, bursty request patterns.
*   **Churn & Headroom:** Assume a fixed 5% hardware failure/maintenance overhead that cannot be allocated to any tier under standard operating conditions.

### C — Constraints
*   **Contractual SLAs ($P_{99.99}$):** High-priority Reserved Instances must clear a 0% allocation rejection rate constraint.
*   **Hardware / SKU Boundaries:** Compute allocations cannot cross physical family boundaries (e.g., an unutilized ND-series GPU cluster cannot easily satisfy an on-demand general-purpose CPU burst without massive financial/efficiency penalties).
*   **Strict Linear Eviction Priority:** If a region runs out of physical capacity and a High-Priority RI user suddenly requests their contracted nodes, the system must follow a hard-coded eviction hierarchy:
    $$\text{Free Tier} \longrightarrow \text{Small Paying (On-Demand)} \longrightarrow \text{Whale (Overage Blocks)}$$

### I — Incentives & Trade-offs
*   **Revenue Protection vs. System Availability:** Prioritizing "Big Whale" overflow workloads protects near-term revenue but runs the risk of starving the contractual safety buffers needed for long-term High-Priority RI renewals.
*   **Utilization vs. Friction:** Running the cluster at 98% utilization maximizes infrastructure efficiency but causes catastrophic cascading latency or `AllocationFailed` rejections for lower-tier paying users the second an unpredictable spike hit.
*   **Eviction Cost:** Evicting a free-tier user is text-book simple (kill container/state), while forcing an on-demand small-paying customer off a node requires structured state-saving or checkpoint migration pipelines, introducing massive overhead.

### D — Decisions (The Architectural Blueprint)
*   **Dynamic Token-Bucket Rate Limiter:** Implement a centralized allocation engine that manages virtual tokens for capacity access based on customer class tier.
*   **Multi-Queue Priority Scheduler:**
    *   *Queue 1 (Strict Pass-Through):* Bypasses the scheduler entirely via pre-allocated physical capacity slices reserved exclusively for contracted RIs.
    *   *Queue 2 (Knapsack Solver):* Packs Whale and Small-Paying requests into the shared pool using a real-time Mixed-Integer Linear Programming (MILP) solver optimizing for max revenue density.
    *   *Queue 3 (Opportunistic Throttler):* Fills the remaining gaps with Free Tier tasks using low-priority preemptible VM states that can be instantly destroyed via an automated backpressure signal.
*   **Telemetry Feedback Loop:** A fast control loop checks localized cluster memory utilization every 500ms to calculate an active "eviction risk factor score," automatically stopping the entry of lower-tier workloads if regional headroom drops below a critical mathematical curve.

---

## Notes / Open polish items
- **C vs. D tension:** C hard-codes a linear eviction order (Free → Small Paying → Whale), while D proposes a MILP optimizing for revenue density. If the optimizer truly chooses based on revenue density, the eviction order is itself a decision variable, not a fixed constraint. Either resolution is defensible — but naming the tradeoff explicitly ("I hard-constrained eviction order because optimizing it in real time adds solve-time risk at the exact moment capacity needs freeing") is a strong, authentic design statement.
- **Solver latency as a system-design constraint:** a 500ms MILP solve cadence is itself a latency budget. What's the fallback if the solver doesn't converge in time? (Same "exact vs. heuristic/warm-start" tradeoff as the caching primitive.)
- **State & failure mode of the allocation engine itself:** where does token-bucket/queue state live — centralized vs. sharded per region? If the controller crashes, is the system fail-open (admit everything, risk SLA breach) or fail-closed (honor only pre-allocated RI slices, reject the rest)?
- **Missing validation close:** what would you measure post-launch to confirm the design works — e.g., RI rejection rate = 0%, Whale SLA attainment %, free-tier preemption frequency.
