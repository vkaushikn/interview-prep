# Newsvendor: Shortage Cost vs. Overage Cost

**The classic problem:** decide how much of something to stock/provision *before* demand is known. If you provision too little, you pay a **shortage cost** (lost sales, SLA breach, idle demand). If you provision too much, you pay an **overage cost** (wasted resource, idle capacity). The optimal quantity balances these two costs against the demand distribution.

## Why it's the right lens for capacity problems
Almost every capacity-allocation decision — cloud reservations, airline seat allocation, courier positioning — is a newsvendor problem wearing a different costume. The "product" varies (compute instances, seats, couriers), but the cost structure (shortage vs. overage, traded off against a demand forecast) is the same.

## The linear simplification
The simplest version: shortage cost = c_u per unit short, overage cost = c_o per unit over, both **linear** (constant per unit regardless of how much you're already short/over). This turns the problem into something solvable as an LP / min-cost-flow — see [time_expanded_network_lead_times.md](time_expanded_network_lead_times.md).

**Refinement (deferred, not default):** if shortage cost is **convex** (the 10th unmet order is worse than the 1st — queues compound), the problem is still tractable as an LP via piecewise-linear segments (convexity means the solver fills cheap segments first, without needing binary variables) — see the discussion in the courier case study. It only becomes a genuinely hard MIP if costs are *non-convex* (fixed penalties, volume discounts).

## The supply-side mirror: overbooking / yield / "demand unconstraining"
The Azure case study used **demand unconstraining** — adjusting an observed (censored) demand signal to account for the fact that past capacity limits suppressed it. There's a **mirror-image** version on the *supply* side: when a resource you're requesting won't fully materialize (no-shows, flaky activations), you request **more than you need**, scaled by a **yield/show-up rate p**: request K/p to get K.

Both are the same move — correcting a planning input for a known systematic gap between "what you ask for / observe" and "what you actually get / happened" — just applied to demand (Azure) vs. supply (courier case study's flex pool).

First surfaced in: Azure capacity case study (shortage/spill costs implicit in nested protection levels); made explicit, with the supply-side mirror, in the Courier Repositioning case study (I — single $ objective; D — flex yield calibration).
