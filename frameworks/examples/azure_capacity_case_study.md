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

#### Version 1 (Gemini)
*   **Dynamic Token-Bucket Rate Limiter:** Implement a centralized allocation engine that manages virtual tokens for capacity access based on customer class tier.
*   **Multi-Queue Priority Scheduler:**
    *   *Queue 1 (Strict Pass-Through):* Bypasses the scheduler entirely via pre-allocated physical capacity slices reserved exclusively for contracted RIs.
    *   *Queue 2 (Knapsack Solver):* Packs Whale and Small-Paying requests into the shared pool using a real-time Mixed-Integer Linear Programming (MILP) solver optimizing for max revenue density.
    *   *Queue 3 (Opportunistic Throttler):* Fills the remaining gaps with Free Tier tasks using low-priority preemptible VM states that can be instantly destroyed via an automated backpressure signal.
*   **Telemetry Feedback Loop:** A fast control loop checks localized cluster memory utilization every 500ms to calculate an active "eviction risk factor score," automatically stopping the entry of lower-tier workloads if regional headroom drops below a critical mathematical curve.

#### Version 2 (Kaushik) — Three-Tier Offline/Online Decomposition
Modeled on the Amazon FC scheduling pattern (rolling-horizon MILP scheduler + dynamic control module):

*   **Nightly Mega-Solver (hours to solve):** Re-optimize a historical day with full hindsight, accounting for eviction and all tier constraints. Output is not a one-shot allocation, but an **operating curve** — e.g., "reserve X% of cluster for RI (never touch, even if smaller than contracted RI total), Y% threshold before throttling free tier," expressed as a function of time-of-day / utilization.
*   **Hourly Mini-Solvers:** Lighter-weight re-optimization attuned to the current day's actual pattern, adjusting the nightly targets based on trailing last-hour data. Bridges the gap between yesterday's hindsight curve and today's reality.
*   **Online Controller (fast, PID-like):** Just tracks the current target curve (as adjusted by the hourly solve) — admit/evict decisions are reactive, cheap, and don't require solving anything in the hot path.
*   **Why this resolves the C-vs-D tension:** the "eviction hierarchy" and per-tier reservation thresholds aren't hard-coded rules *or* a live MILP — they're outputs of the offline/hourly layers, re-derived periodically. The online controller only has to react to deviation from the target curve.

---

## Notes / Open polish items
- **C vs. D tension (Version 1):** C hard-codes a linear eviction order (Free → Small Paying → Whale), while D proposes a MILP optimizing for revenue density. If the optimizer truly chooses based on revenue density, the eviction order is itself a decision variable, not a fixed constraint. Either resolution is defensible — but naming the tradeoff explicitly ("I hard-constrained eviction order because optimizing it in real time adds solve-time risk at the exact moment capacity needs freeing") is a strong, authentic design statement. **Version 2 resolves this** by deriving the eviction thresholds offline/hourly rather than hard-coding or live-solving them.
- **Solver latency as a system-design constraint (Version 1):** a 500ms MILP solve cadence is itself a latency budget. What's the fallback if the solver doesn't converge in time? (Same "exact vs. heuristic/warm-start" tradeoff as the caching primitive.) **Version 2 sidesteps this** — the only real-time component is the cheap PID-like controller.
- **State & failure mode of the allocation engine itself:** where does token-bucket/queue state (V1) or target-curve state (V2) live — centralized vs. sharded per region? If the controller crashes, is the system fail-open (admit everything, risk SLA breach) or fail-closed (honor only pre-allocated RI slices, reject the rest)?
- **Missing validation close:** what would you measure post-launch to confirm the design works — e.g., RI rejection rate = 0%, Whale SLA attainment %, free-tier preemption frequency.
- **Temporal stationarity assumption (Version 2):** the operating curve implicitly assumes "last week today ≈ this week today" — which breaks for special events (e.g., a World Cup match driving an unusual traffic pattern in July). Generally a fair assumption, and expressing the curve as a *ratio/shape* (rather than absolute levels) makes it fairly robust to scale shifts. Still need a metric/guardrail that flags when the live pattern is deviating enough from the target curve that the curve itself (not just the online controller's tracking) is stale — i.e., distinguish "controller is lagging the target" from "the target itself is wrong today."

## Round 2 — Deeper Weaknesses (genuinely hard, open problems)

### 1. Compositional shift vs. scale shift
A single ratio-based curve implicitly assumes the tier *mix* is stable — it's a top-down allocation. A structural shift (new Whale onboards, marketing campaign floods Free Tier) changes the mix itself, not just the total volume, and a ratio curve doesn't directly capture that.

- **Bottom-up forecasting:** forecast each tier's demand independently rather than splitting an aggregate forecast by fixed ratio — a new Whale shows up directly in the Whale forecast instead of being diluted across the aggregate. This is the hierarchical forecast reconciliation problem (top-down vs. bottom-up vs. middle-out) from the Microsoft multi-echelon work.
- **Regime detection:** CUSUM/EWMA on each tier's demand series to distinguish normal noise within the curve's shape from a genuine compositional shift — this is what would trigger an out-of-cycle re-solve. Notably, this is the same regime-detection mechanism needed for gain scheduling (Round 1, #4) — a compositional shift *is* a regime change.
- **Local re-optimization in the hourly mini-solve (Kaushik's extension):** give the hourly solve real freedom, not just a magnitude tweak within a fixed shape. Concretely: monitor job arrival rates by bucket, build a simple next-hour forecast, and solve a small forward-looking problem that is allowed to *locally redefine* the operating curve for the next window — within some policy-defined bounds on how far it can deviate from the nightly curve. The nightly solve still sets the long-run shape/guardrails; the hourly solve gets bounded authority to reshape the curve in response to what's actually arriving right now.

### 6. Policy-driven feedback into training data (endogeneity)
This is effectively the **off-policy evaluation** problem from causal inference / RL — historical data was generated under the current policy, so "what would demand look like under a different policy" can't be read directly off that data. A real, deep field; naming it correctly is itself a signal, even without a complete solution.

Pragmatic mitigations:
- **Controlled experimentation:** run different operating-curve variants in different regions (holdout/canary regions) to get genuine causal variation instead of relying purely on observational history generated by one policy.
- **Dampened policy updates:** bound how much the curve can shift week-over-week, slowing the feedback loop enough that monitoring can catch a bad trajectory before it compounds.
- **Churn as a first-class signal:** track Small-Paying churn/retry rates as a leading indicator and feed it back into the offline objective as a penalty term, so the optimizer is at least aware it's shaping future demand.
