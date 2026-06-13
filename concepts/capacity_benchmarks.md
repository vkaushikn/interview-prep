# Order-of-Magnitude Capacity Benchmarks ("Is this easy or hard?")

Once you've derived a required throughput (reads/sec, writes/sec, etc.) from your Approximations, judging "easy / hard / infeasible" means dividing by a rough per-node capacity for the relevant component, and looking at the resulting node count.

## Rough single-node benchmarks (order of magnitude — not precise, just enough to reason live)

| Component | Rough capacity per node |
|---|---|
| In-memory cache / KV store (Redis, Memcached) — simple key lookup | ~100K ops/sec |
| Disk-backed relational DB — indexed query | ~1K–10K qps |
| Message queue / log (Kafka-like) — sequential append | ~100K–1M msgs/sec per broker |
| Application server — business logic, no heavy I/O | ~1K–10K req/sec |

## The heuristic

$$\text{node count} \approx \frac{\text{required throughput}}{\text{per-node capacity}}$$

- **Node count in the 1–100 range:** doable with standard horizontal scaling — this is a sizing problem, not a design problem.
- **Node count in the 1,000s+ for a single access pattern:** the *design* is wrong, not under-provisioned. This is the signal to change the architecture (add a cache layer, precompute/fan-out, change the access pattern) rather than "add more machines."

**OR analog:** this is identical to capacity planning — "given per-unit throughput and a required total throughput, how many units do I need" is the same math as sizing fulfillment-center stations or server racks to hit a target.

## Applied to the News Feed numbers
- **20K reads/sec against an in-memory cache** (~100K ops/sec/node) → well under 1 node's worth (a small cluster for redundancy) — **easy**.
- **800K fan-out writes/sec against a queue** (~100K–1M msgs/sec/broker) → roughly 1–10 brokers — **doable, needs real infra, but a solved problem**.
- **4M reads/sec against a disk-backed posts table** (~1K–10K qps/node) → hundreds to ~1,000+ nodes for *one* query pattern — **infeasible; the design itself is the problem**, not the provisioning.

This last line is exactly why the pull (fan-out-on-read) model gets ruled out at this scale, and is the kind of "the numbers tell the story" statement that's much stronger than asserting a design choice on taste alone.
