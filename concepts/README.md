# Concepts

A running glossary of "classic" distributed-systems concepts encountered while working through system design problems.

Each entry should aim to:
- State the textbook concept in plain terms.
- Translate it to an OR/control-theory analog where one exists (the goal is to anchor new vocabulary to things already understood, not memorize definitions cold).
- Note which case study/example first surfaced it.

## Index
- [Read-Your-Own-Writes (RYOW) Consistency](read_your_own_writes.md) — staleness/propagation-delay between a write and the read path seeing it.
- [Order-of-Magnitude Capacity Benchmarks](capacity_benchmarks.md) — per-node throughput rules of thumb for judging "easy / hard / infeasible" from a derived throughput number.
- [Latency Numbers Worth Knowing](latency_numbers.md) — human perception thresholds and the memory/SSD/network access ladder, incl. why cross-region round trips dominate.
