# Case Study: Courier Repositioning Across Zones (Capacity/OR)

## Problem Statement

> A food-delivery platform has a fleet of couriers who can be in one of several city zones at any time. At the start of each hour, the platform can "reposition" idle couriers between zones (at some cost/time penalty) before new orders arrive. Demand (orders/hour) varies by zone and by hour of day, and is uncertain. If a zone is under-supplied, orders wait longer (hurting customer experience and cancellation rate); if over-supplied, couriers sit idle (wasted labor cost).
>
> **Design a system that decides, each hour, how many couriers to reposition into each zone.**

This is a *spatial-temporal allocation* problem — a different shape from the Azure capacity case study (a *time-based reservation* problem), but the same OR family (newsvendor-style shortage/overage trade-offs under a forecast).

---

## A — Assumptions & Approximations

### Assumptions (scope/model simplifications)
- **Point forecast for the next N hours**, per zone and per hour. A standard-deviation/uncertainty estimate may also be available, but v1 does not use it (deterministic-first, refine later — same philosophy as Azure's offline-first approach).
- **Rolling horizon**: solve over the full N-hour window each time, but only *execute* the next hour's decisions before re-forecasting and re-solving. (Standard receding-horizon/MPC pattern.)
- **Deterministic travel-time matrix** travel_time(zone1, zone2) — doesn't depend on time of day, not stochastic. (Stochastic travel time → stochastic simulation is a named future refinement.)
- **Linear under-serving (shortage) cost** — each unmet order in a zone-hour costs a constant $c, regardless of how many others are also unmet that hour. (Piecewise-linear *convex* shortage cost is a refinement that stays an LP — see [newsvendor_shortage_overage.md](../concepts/newsvendor_shortage_overage.md) — but isn't needed for v1.)
- **Two labor pools:**
  - **Base pool:** fixed total headcount (closed system, pure conservation). Cost is sunk for the shift — the only "cost" of a repositioning decision is that the driver is unavailable (counts toward neither origin nor destination zone's capacity) while in transit.
  - **Flex pool:** elastic — new workers can be activated in any zone "for free" (they appear where needed), but activation commits to paying for a fixed `min_commitment_hours`, after which they leave automatically.
- **Orders are uniform** (same delivery-time/throughput-consumption per order); deliveries don't carry over across hour boundaries — "throughput per driver per hour" already absorbs this.
- **Decomposition into "virtual cities"** — a *data-dependent* approximation, not a Fermi estimate: cluster the travel-time graph; zone pairs with travel_time > planning horizon are structurally decoupled (no repositioning decision can ever connect them within the horizon), so the city decomposes into near-independent sub-problems. The number and size of these clusters is determined empirically from the travel-time matrix, not assumed up front. (This is a genuinely different *kind* of "approximation" than Fermi sizing in the System Design ACID — it's a placeholder for an analysis step whose *output type* is specified now, not a number computed now.)

---

## C — Constraints

### State representation: a time-expanded network
The state is modeled as a **time-expanded network** (see [time_expanded_network_lead_times.md](../concepts/time_expanded_network_lead_times.md)) — each node is a (zone, hour) pair:

- **S_FT(z, h):** base-pool drivers stationed in zone z at hour h.
- **S_Flex(z, h, a):** flex-pool drivers in zone z at hour h with `a` hours remaining on their commitment (a **countdown chain** — see [countdown_chain_deterministic_sojourn.md](../concepts/countdown_chain_deterministic_sojourn.md) — rather than tracking cohorts by start time).
- **T_FT(z1, z2, h1):** base-pool drivers in transit from z1 to z2, departed at h1, arriving at h1 + travel_time(z1,z2). While in transit, they count toward neither zone's capacity.
- **T_Flex(z1, z2, h1, a):** same for flex, carrying their countdown along.

### Decision variables
- **x(z1, z2, h):** drivers (base or flex) dispatched from z1 to z2 departing at hour h.
- **y(z, h):** new flex workers activated in zone z at hour h (an *inflow* into the system — unlike base pool, flex headcount isn't fixed).

### Balance / conservation equations
`S_FT(z,h) = S_FT(z,h-1) + (arrivals into z at h from transit) - (departures from z decided at h)`, and analogously for S_Flex with the countdown shift: `S_Flex(z,h,a) = S_Flex(z,h-1,a+1)` for a < max, `S_Flex(z,h,max) = y(z,h)`.

### Demand/throughput constraint
`throughput(z,h) ≤ demand(z,h)` and `throughput(z,h) ≤ capacity(z,h)` (capacity derived from S_FT + S_Flex via per-driver throughput rate). `shortage(z,h) = demand(z,h) - throughput(z,h) ≥ 0`. This is the "demand sink" framing — there's no incentive to provision capacity beyond demand, so shortage falls out automatically as unmet demand, without a separate `max(0, ...)` term.

### Integrality
Decision variables (driver counts) are naturally integers, but at the scale involved (tens-to-hundreds per zone-hour), the **LP relaxation's integrality gap is negligible** — solve as continuous, round with simple ad-hoc heuristics (round down, allocate leftover units to the steepest marginal-shortage-cost zone-hours). Not worth a MIP.

### Frozen / Firm / Free decision zones
Some decisions have **lead times to the people executing them** — e.g., telling the 7pm night shift which zone to report to requires a decision by ~2pm. Once communicated, that assignment becomes a **frozen** input to every subsequent hourly re-solve (3pm-6pm), not a variable — otherwise the plan thrashes hourly and operations descend into chaos. See [frozen_firm_free_zones.md](../concepts/frozen_firm_free_zones.md).

---

## I — Incentives & Trade-offs

Because of the choices made in A (linear shortage cost, no separate idle-cost for base pool), **I collapses to a single $ objective**: minimize (flex activation cost + total shortage cost). This is worth stating explicitly *as a finding*, not skipping over — it's a direct consequence of A's simplifications, not evidence that I doesn't apply:

- Linear shortage cost → the optimizer is indifferent to *which* zone-hour bears a shortage, as long as the total is minimized. A richer I-level tension (e.g., minimizing total $ cost vs. keeping shortage *variance* across zones low — a fairness/SLA-compliance concern) is **deferred, not absent** — it re-emerges if/when shortage cost becomes convex.
- No idle-cost for base pool (sunk regardless of utilization) → only an *opportunity cost* (capacity that could've served elsewhere). Flex has a real overage cost (premium paid even if demand doesn't materialize).
- **Organizational conflict** (zone managers have their own metrics and may not execute repositioning as recommended) is a genuine I-level issue — but the model itself can't resolve it. It's addressed as a **rollout** concern in D.

---

## D — Decisions

### Core algorithm
A **time-expanded multi-commodity min-cost-flow LP** (base pool + flex pool as two commodities), solved on a rolling horizon (N hours, execute only hour 1), with frozen/firm/free zones for decisions with execution lead time, LP relaxation + simple rounding for integrality.

### End-of-horizon (terminal effects)
A hard N-hour cutoff gives the optimizer zero incentive to value anything past hour N — it can "spend down" the system into a bad terminal state. Standard fixes, in order of preference:
1. **Extend the horizon across the boundary** if any forecast (even low-confidence) exists for the next period — avoids a special case entirely.
2. If not, **check empirically (historical analysis)** whether terminal effects actually matter for this system before engineering a fix.
3. In this system specifically, **frozen night-shift assignments act as natural anchor points** for the state near the horizon boundary — a chunk of the "terminal state" is already pinned down by real commitments, which may make (1)/(2) moot for the unfrozen remainder.

### Rollout: the model produces a plan, not an order
Three real-world frictions surfaced in critique:
- **(a) Zone managers have their own metrics** and may not execute a repositioning as recommended — an I-level organizational issue the model can't fix directly.
- **(b) Forecast uncertainty** means managers' "gut" (e.g., "Downtown won't actually get that many orders") may carry real information the forecast lacks.
- **(c) Flex activation is unreliable** in practice — not everyone requested actually shows up.

All three resolve into one shape: **the LP's output is a recommendation to humans, and several of its parameters are calibrated yield/compliance rates, not assumed-1 constants.**
- **(c)** is the supply-side mirror of Azure's demand-unconstraining: if historical flex show-up rate is p, request K/p to get K. See [newsvendor_shortage_overage.md](../concepts/newsvendor_shortage_overage.md).
- **(b)** → give managers a **forecast-override interface** ("I don't believe this demand number"), not a decision-override interface. Re-solve with the adjusted forecast. Keeps one source of truth, and overrides become measurable feedback (consistently right → fix the forecast model; consistently wrong → useful feedback to the manager).
- **(a)** → **phased rollout**: shadow mode (log recommendations, compare against what managers actually did, build an evidence base) → advisory → mandatory-where-trust-has-been-earned. Same three-layer shape as Azure's offline/online split, with humans occupying the "online correction" layer.

---

## See also
- [practice_modes.md](../frameworks/practice_modes.md) — this case study was built in Teacher mode; a Critique-mode pass (HVAC-style: something breaks, can the design recover) is a natural follow-up.
- Related concepts: [time_expanded_network_lead_times.md](../concepts/time_expanded_network_lead_times.md), [countdown_chain_deterministic_sojourn.md](../concepts/countdown_chain_deterministic_sojourn.md), [frozen_firm_free_zones.md](../concepts/frozen_firm_free_zones.md), [newsvendor_shortage_overage.md](../concepts/newsvendor_shortage_overage.md).

## Open items / things to revisit
- Stochastic travel times (named refinement, deferred to "stochastic simulation algorithms").
- Piecewise-linear convex shortage cost (stays an LP, deferred as v1.1).
- Whether base-pool drivers' end-of-shift positions persist to tomorrow, or they "go home" to fixed locations — determines how much the terminal-effects problem matters for base pool specifically.
- Sizing/decomposition (number of virtual cities, zones per cluster) — explicitly deferred to empirical analysis of the travel-time graph.
