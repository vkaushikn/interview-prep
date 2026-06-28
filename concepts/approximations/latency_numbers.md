# Latency Numbers Worth Knowing

Orders of magnitude, not exact values — enough to reason about whether a design choice blows the latency budget.

## Human perception thresholds (Nielsen/Doherty)
- **<100ms:** feels instant.
- **<1s:** feels responsive — no perceived interruption of flow.
- **>1s:** starts feeling sluggish.
- **>10s:** user disengages.

For an interactive feed load, a reasonable target is **P99 ≈ 200-300ms end-to-end**.

## Access latency ladder
| Operation | Rough latency |
|---|---|
| Memory access | ~100ns |
| SSD read | ~0.1-1ms |
| Same-datacenter network round trip | ~0.5-1ms |
| Cross-region / cross-continent round trip | ~150-300ms |

## Why this matters
The cross-region number is dominated by **speed-of-light / routing physics**, not by server processing speed — no amount of engineering on either end fixes it. If a request has to cross continents (e.g., fetching a post stored in a datacenter in India for a reader in the US), that round trip alone can consume the entire latency budget before any DB or cache work happens.

**Implication for design:** geographic data placement (replicating/co-locating data near where it's read) is often the *dominant* term in the latency equation — not a secondary optimization.

First surfaced in: News Feed case study (C — Constraints, latency + geography).
