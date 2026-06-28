# Time-Expanded Networks: Modeling Lead Times in Flow Problems

**The problem:** in a multi-period allocation problem, moving a resource from location A to location B isn't instantaneous — it takes some lead time L. A unit that departs A at time t doesn't become available at B until t+L. A naive period-by-period model (state at time t depends only on state at t-1 plus this period's decisions) can't represent "in transit, committed but not yet arrived."

**The trick:** expand the network so that each node is a **(location, time)** pair, not just a location. An arc from (A, t) to (B, t+L) represents "depart A at t, arrive B at t+L" — the resource is "in" that arc, counted as neither A's nor B's available stock, for the duration. The state-transition / conservation equation at (B, t+L) then includes an inflow term from this arc.

## Why this matters
This turns "lead time" from a special case requiring custom logic into **just another arc in a standard min-cost flow / LP** — the conservation equations are uniform across the whole network, time-expansion does the bookkeeping.

## Applied to the Courier Repositioning case study
- State at (zone z, hour h): drivers stationed there.
- Arcs: x(z1, z2, h) — drivers departing z1 at hour h, arriving z2 at hour h+travel_time(z1,z2). While in transit, they count toward neither zone's available capacity (D, decision variables).
- A **demand sink** at each (zone, hour) with capacity = forecasted demand — flow into this sink is "throughput"; anything short of capacity is "shortage" (see [newsvendor_shortage_overage.md](newsvendor_shortage_overage.md)).

## Connection to decomposition
The number of (z1, z2) arc-pairs that matter is bounded by which pairs have travel_time ≤ planning horizon — pairs with longer travel times are effectively disconnected within the horizon. Clustering the travel-time graph into near-independent components ("virtual cities") is a **free decomposition**: it's not an approximation chosen for tractability, it's recognizing a decoupling that's already true of the underlying time-expanded network.

First surfaced in: Courier Repositioning case study (C — state definition).
