# Frozen / Firm / Free Zones in Rolling-Horizon Planning

**The problem:** a rolling-horizon optimizer re-solves every period with updated information — in general a good thing (corrects for forecast error, as in [reference_vs_denormalization.md](reference_vs_denormalization.md)-style feedback). But if *some* decisions have already been **communicated to people who need lead time to act on them**, re-solving and changing those decisions every period creates chaos for those people, even if each individual re-solve is "more optimal."

**The trick (classic MRP/production-planning device):** partition the planning horizon into three zones:
- **Frozen:** decisions already committed/communicated — fixed parameters (not variables) in every subsequent re-solve, regardless of what new information arrives.
- **Firm:** tentatively planned, can still change, but changes carry a penalty (reflecting real disruption cost).
- **Free:** fully optimizable, no commitments yet.

As time advances, free → firm → frozen — each decision crosses these boundaries exactly once, at the moment its lead-time deadline arrives.

## Applied to the Courier Repositioning case study
A 2pm decision telling the 7pm night shift which zone to report to has a 5-hour lead time. From 2pm onward, every hourly re-solve (3pm, 4pm, ..., 6pm) must treat that assignment as **frozen** — a fixed input, not a variable — even though the LP would happily re-optimize it given the latest forecast.

**Bonus effect:** these frozen assignments also act as natural **anchor points for the end-of-horizon terminal-effects problem** ([discussed in the case study, D]) — they pin down part of the state near the horizon boundary without needing an explicitly engineered terminal cost.

## General applicability (Cloud/Compute)
Any system with **notice periods** has this structure — e.g., a capacity-scaling decision that requires advance notice to a cloud provider (reserved-instance changes, scheduled scale-up requests) is "frozen" the moment it's submitted, even if your internal forecast updates before it takes effect.

First surfaced in: Courier Repositioning case study (D — end-of-horizon / night-shift notice).
