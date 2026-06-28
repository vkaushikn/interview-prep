# Countdown Chains: Modeling a Deterministic Sojourn Time

**The problem:** a resource enters a system via a decision (an "activation"), stays for a **fixed duration** (a commitment period), and then automatically leaves — without a separate "removal" decision. Tracking this by remembering *when each cohort started* and computing exit times relative to a global clock gets bookkeeping-heavy fast (which cohorts are still active at time t? subtract which start times from which?).

**The trick:** instead of indexing by *start time*, index by **remaining duration**, a ∈ {1, ..., max}. Each period:
- New activations enter at a = max.
- Everyone's `a` decrements by 1 (a "shift register").
- Anyone who *was* at a = 1 simply doesn't exist next period — no explicit removal/exit decision needed. The countdown reaching zero **is** the exit.

This is the same structure as **age-structured inventory** or an **Erlang phase-type approximation** of a fixed service time — a chain of "stages" with deterministic transition (decrement) and a forced exit at the end of the chain.

## Why it's cleaner than cohort-by-start-time
The balance equation becomes purely local and shift-invariant: `S(z, h, a) = S(z, h-1, a+1)` for a < max, and `S(z, h, max) = (new activations this period)`. No need to track "which cohorts from the past are still valid" — the age variable *is* the remaining validity.

## Applied to the Courier Repositioning case study
Flex workers activated with a `min_commitment_hours` constraint are tracked by hours-remaining-on-commitment rather than by activation hour. This sidesteps the "remember to remove old cohorts at the right future time" bookkeeping entirely.

## General applicability (Cloud/Compute)
Any resource with a **fixed commitment window that auto-expires** — a reserved-instance term, a spot-instance interruption notice, a lease — has this shape. Modeling it as a countdown chain (rather than tracking absolute start/end timestamps per cohort) is the cleaner formulation whenever the commitment length is a small fixed integer number of periods.

First surfaced in: Courier Repositioning case study (C — flex worker state).
