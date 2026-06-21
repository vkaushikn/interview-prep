# Case Study: GPU Reserved Instance / Spot Capacity Allocation

## Problem Statement

> A cloud provider runs a fleet of GPUs serving ML workloads. Customers can buy capacity in two ways:
> - **Reserved Instances (RI)**: purchased for a fixed term, guaranteed availability, premium price.
> - **Spot instances**: steeply discounted (~60-70%), but the provider can preempt them at any time to free capacity for RI customers.
>
> The provider must decide how to allocate capacity between RI and Spot, and which customers to admit as RI customers, while guaranteeing each RI customer a low probability of preemption.

---

## Step 0 — The Decomposition Fork

The problem visibly invites a **joint chance-constrained LP**: choose customer allocations to maximize weighted revenue subject to per-customer SLA constraints (P(preemption) ≤ ε for each RI customer), where the constraints involve correlated random variables (simultaneous utilization across all customers).

That formulation is analytically intractable as stated — correlated utilization profiles don't simplify the chance constraints into closed form. The decomposed path (simulation-first, then a small deterministic LP for equity) is the right practical answer here. Worth naming the joint formulation upfront to show you see it, then pivoting to the decomposition.

---

## A — Assumptions & Approximations

### Assumptions (scope)
- **Supply is known and deterministic**: physical GPU capacity follows a known time series (e.g., Month 1 = 100 GPUs, Month 2 = 200 GPUs). No supply uncertainty in v1.
- **RI term = 1 month** (simplest case; multi-month terms are a named extension — see Open Items).
- **Spot is a residual, not a quantity decision**: once the RI book is set, Spot fills idle capacity dynamically. Spot pricing is managed via dynamic pricing (Uber-style surge) to clear the spot market. No separate Spot quantity optimization is needed.
- **The overbooking ratio is the one decision**: you sell RI slots beyond physical capacity, betting that not all RI customers peak simultaneously (statistical multiplexing). Spot customers are the evictable buffer when simultaneous RI demand spikes.
- **Fault buffer**: a fixed percentage of physical capacity (e.g., 5%) is reserved for hardware failures and maintenance. This is a constant, not modeled stochastically.
- **Two customer modes**:
  - **(a) Existing customers with history**: utilization profile (time_of_day → gpu_used) is available.
  - **(b) New customers / changing profiles**: no history, or history no longer representative. Handled separately (see D).

### Approximations (sizing)
- Effective capacity = physical_capacity × (1 − fault_buffer).
- Simulation granularity: 15-minute intervals captures job-level burstiness without being intractable.
- Customer utilization profiles are fit to **parametric distributions** (Beta for utilization fraction bounded in [0,1]; Gamma or mixture model if usage is bimodal — low-utilization idle state vs. high-utilization active state). Bootstrapping from raw historical samples is not used: it can't generalize to new customers, can't handle structural breaks, and overfits to past correlation patterns.

---

## C — Constraints

- **Physical capacity**: total simultaneous RI utilization across all customers must not exceed effective capacity. The overbooking ratio is the lever; the SLA constraint is what bounds it.
- **Per-customer SLA**: P(customer *i* is preempted when they need capacity) ≤ ε. This is a **per-customer** guarantee, not just an aggregate one — RI customers are buying a service-level promise, not just a statistical average. The simulation (in D) evaluates this per-customer directly.
- **Proportionality floor** (equity): if demand must be cut, no customer is zeroed out while others receive their full request. Every admitted customer gets at least (requested allocation × global cut fraction). This is a constraint in the Stage 3 LP, not the objective.

---

## I — Incentives & Trade-offs

**1. Revenue vs. SLA breach risk.** Selling more RI slots increases revenue linearly; each additional slot sold also incrementally increases P(simultaneous demand > capacity). The overbooking ratio balances these. The marginal benefit of selling one more RI slot is (RI_price − Spot_price) × expected_utilization (since that slot earns Spot revenue anyway if the RI customer doesn't use it at that moment). The marginal cost is the increase in P(SLA breach) × breach penalty.

**2. Portfolio selection: correlation is a resource.** Customers with complementary utilization profiles (e.g., a US company peaking 9am–5pm ET and an Indian company peaking 9am–5pm IST — a ~12-hour offset) allow a higher safe overbooking ratio than two US companies with identical profiles. Selecting for low-correlation customers is a form of revenue extraction beyond just the overbooking ratio. Pure correlation-chasing, however, is greedy — it could systematically exclude valid customers who happen to share a time zone with others already in the book.

**3. Relationship vs. proportionality.** When capacity must be cut, two objectives compete:
- **Relationship / priority**: preferred customers (high spend, strategic accounts, long tenure) should be protected — a weighted objective in the allocation.
- **Proportionality**: every customer gets a fair fraction of what they requested — a floor constraint, not an objective.
Keeping them in different roles (objective vs. constraint) lets them compose without fighting. The interviewer's definition of "equitable" sets the floor level and the priority weights.

**4. Expected-case optimization vs. robustness.** A policy tuned to the median simulation scenario can perform poorly in the P90 scenario (correlated peak demand, an unexpected large customer burst). Running 100s of simulation scenarios and selecting the policy robust across the P90 worst case is more defensible operationally — at some cost in expected-case revenue.

---

## D — Decisions

### The algorithm: three-stage decomposition

**Stage 1 — Simulation (tightness measurement)**

For each existing customer, fit a parametric distribution to their utilization profile (per 15-minute slot). Run the simulation: sample simultaneously from all customer distributions, sum across customers to get total GPU demand at each time slot. Repeat for 100s of replications to get a distribution of peak demand.

- If P99 peak demand ≤ effective capacity across replications: the full book is feasible as-is. Sell every customer their requested allocation. Done.
- If P99 peak demand > effective capacity: the book needs cutting → Stage 2.

The simulation output also gives per-customer preemption probability directly (fraction of replications where that customer's demand is unmet), so the per-customer SLA constraint (C) is evaluated here without any analytical approximation.

**Stage 2 — Revenue-maximizing cut**

Rank customers by **revenue per unit of marginal contribution to the P99 peak**:
- For each customer, re-run the simulation without them and measure how much the P99 peak drops. This is their marginal contribution to the binding constraint.
- Rank by (revenue contribution) / (marginal P99 reduction).

Trim customers from the bottom of this ranking — reduce their allocation or remove them — until P99 peak ≤ effective capacity. This produces the **revenue-maximizing feasible portfolio** without requiring a combinatorial search.

**Stage 3 — Equity adjustment (LP / heuristic)**

Take the Stage 2 portfolio as the starting point. Apply:
- **Proportionality floor constraint**: no customer's allocation falls below (their requested allocation × minimum_floor_fraction).
- **Relationship weights in the objective**: preferred customers' allocations are maximized first within the feasible set.

This is a small LP (the combinatorial problem was solved in Stage 2; this is just a linear adjustment within the feasible set) or a simple heuristic if the LP is overkill for the scale.

**Robust policy selection**: instead of optimizing for the median simulation replication, evaluate candidate policies (Stage 2/3 outputs) across the full set of 100s of replications and select the policy that performs best at the P90 worst case. Same CEM/simulation-optimization pattern as Azure Alternative D.

---

### Mode (b): new customers and changing profiles

**New customers (no history)**:
- Cluster existing customers by workload type (training-heavy, inference-heavy, batch/scheduled), fit cluster-level distributions.
- Assign new customer to a cluster based on stated workload type → use cluster distribution as prior in the simulation.
- Update toward customer-specific distribution as usage history accumulates.

**Changing workload profiles (e.g., a customer shifting from training to inference)**:
- Account Manager (or Customer Success) provides an override to the fitted distribution — the simulation re-runs with the updated input distribution.
- The AM's override becomes measurable: track whether actual subsequent usage matches the adjusted distribution. Consistently accurate → AM insight improves the model; consistently wrong → useful feedback to the AM. Same forecast-override interface as the [courier repositioning case study](courier_repositioning_case_study.md)'s zone manager override.

---

## Open Items / Things to Revisit

- **Multi-month RI terms** (12-month is the real-cloud norm): Month 1's RI decision locks capacity across Months 2–12, entangling with a growing supply trajectory. This becomes a multi-period, multi-cohort allocation problem — the same time-expanded network structure as the [courier case study](courier_repositioning_case_study.md), with RI cohorts as commodities flowing across (month, customer) nodes.
- **Simulation granularity**: 15-minute intervals assumed. Finer captures sub-job burstiness; coarser is faster. Empirical validation needed.
- **Spot pricing algorithm**: the dynamic pricing model that clears the spot market was deferred. It interacts with the RI allocation (more RI sold → less idle capacity → higher spot prices, which in turn signals demand back to the RI-vs-Spot decision).
- **Iterative customer negotiation**: the current model is provider-dictated. An alternative is an iterative bidding process where customers see a proposed allocation and can adjust their request — which gives the provider better information about true demand elasticity.

---

## See Also
- [newsvendor_shortage_overage.md](../concepts/newsvendor_shortage_overage.md) — the shortage/overage structure behind Trade-offs 1 and 3; the supply-side yield correction (fault buffer) mirrors demand unconstraining.
- [azure_capacity_case_study.md](azure_capacity_case_study.md) — the nested protection levels structure (RI = protected class, Spot = evictable buffer) and the CEM/simulation-optimization pattern (robust policy selection across scenarios).
- [courier_repositioning_case_study.md](courier_repositioning_case_study.md) — forecast-override interface (AM input → override distribution, not output policy); time-expanded network (relevant for multi-month RI extension).
- [practice_modes.md](../frameworks/practice_modes.md) — built in Teacher mode; Critique-mode pass (e.g., "a customer's workload shifts suddenly mid-term and your simulation's prior is stale — what breaks?") is a natural follow-up.
