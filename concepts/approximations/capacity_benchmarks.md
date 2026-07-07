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

## The ÷10 ladder (a mnemonic, not a lookup table)

Don't memorize the table above as trivia — memorize the *pattern*: each layer of durability/coordination you bolt on costs roughly **10x** throughput versus the layer below it, per node.

| Layer | ~ops/sec/node | What you paid for |
|---|---|---|
| Pure RAM (in-memory KV) | ~1M | nothing — no disk, no consensus |
| SSD-backed, no replication | ~100K | disk I/O enters |
| Durable DB write (fsync + indexes) | ~10K | durability tax |
| Replicated/quorum write (consensus) | ~1K | coordination tax |

Being within 10x of the "real" number is the bar in an interview — the process of reasoning down the ladder is what's being graded, not recall of a spec sheet.

Three more anchors to keep alongside it:
- **Network per node:** ~1–10 Gbps ≈ 100MB–1GB/sec. Use this when payload size is the constraint, not op count.
- **Storage per node — two different numbers, don't conflate them:**
  - *Hard ceiling* (what a managed instance can physically hold): ~50–100TB (e.g., Aurora tops out near 128TB).
  - *Practical shard target* (what you'd actually provision): a few TB — chosen not because more won't fit, but to keep rebalance/recovery time on failure sane. `total data size ÷ few TB ≈ node count` uses this second number, not the hard ceiling.
- **RAM per node:** ~0.5–4TB realistic on production hardware (exotic boxes reach ~24TB but are rare/expensive — don't assume you get one). This is often the *real* binding constraint, ahead of storage and even the QPS ladder above: an index needs to live mostly in RAM to serve fast lookups, and a multi-TB **index** (not the whole table) can demand more RAM than a node realistically has, even while the underlying table's disk footprint is nowhere near its storage ceiling.

## Row-size estimation buckets

For sizing a metadata table (rows × bytes/row → total bytes), collapse every column into one of three buckets instead of trying to recall exact type sizes:

| Bucket | Size | What goes here |
|---|---|---|
| Number | 8 bytes | Any ID, timestamp, counter, boolean, enum |
| Short text | ~50 bytes | Name, email, username, short label |
| Long text / URL / path | ~150 bytes | URLs, file paths, descriptions |

Apply a **1.5–2x multiplier** at the end for row/index overhead (transaction metadata, padding, the primary-key index itself).

Why these three numbers: 8 bytes = one int64 = a CPU word, covers any numeric field. 50 bytes ≈ a couple of English words, covers names/emails/labels. 150 bytes covers real-world URLs/paths, which commonly run 50–200 characters. Getting the order of magnitude right matters far more than precision here — a 2x error in row size rarely moves the answer across a TB/PB/EB boundary, but using the wrong number of zeros does.

**Unit ladder** (in case it's not already automatic): KB → MB → GB → TB → PB → EB = 10³ → 10⁶ → 10⁹ → 10¹² → 10¹⁵ → 10¹⁸ bytes, each step exactly 1000x.

**Time conversion used throughout this file's QPS math:** 1 day ≈ 10⁵ seconds (real: 86,400); 1 year ≈ 3×10⁷ seconds, or just multiply a daily number by 365.

## Single-number quick reference (conservative anchors, not ranges)

Ranges are accurate but hard to recall under pressure. The fix: collapse each range to **one number, always the conservative (lower-capacity) end** — overestimating capacity and calling something "fine" when it isn't is a correctness failure live in the room; underestimating just costs a little unnecessary headroom. A defensible, deliberately-conservative number is also robust against a hard-to-please interviewer: there's no "actually the real number is higher" rebuttal that hurts you, since you already rounded down on purpose.

Only **three numbers are actually hardware facts**, reused across every component below — disk ceiling, RAM ceiling, network bandwidth. What differs per component is (a) which dimensions are even relevant, and (b) the operation-specific throughput, which differs because of *why* each system is fast or slow (durability tax, sequential-vs-random access, etc.) — not because the hardware changed.

| | Disk | RAM | Throughput | Network |
|---|---|---|---|---|
| **DB** | 100TB | 2TB | 1K writes/sec, 10K reads/sec (writes pay a durability/replication tax reads don't) | 100MB/sec |
| **Cache** | — (no durability requirement — data loss on restart is acceptable) | 2TB *(reused)* | 100K/sec, read+write symmetric (no durability tax to split them) | 100MB/sec *(reused)* |
| **Queue** | 100TB *(reused, for retention)* | will not bind (no random-access index — consumers track a sequential offset) | 100K/sec put+get | 100MB/sec *(reused)* |

**Always run the network check, not just the throughput-count check** — it's easy to forget because the count number can look safe while the byte rate isn't. `ops/sec × avg payload size` vs. the 100MB/sec anchor: DB reads (10K/sec) have ~10KB of byte-budget per op before bandwidth binds; cache (100K/sec) only has ~1KB per op, already at the edge; queue messages vary the most in size, so this is the one most likely to actually bite. A low op-count alone never proves you're safe — multiply it out first. (This is also the check that catches a design smell, not just a sizing miss — e.g., "querying full photo objects through a DB" fails this math by 20-500x, which is really a signal the blob/metadata split was skipped, not that more bandwidth is needed.)

## The 60-second recipe

1. Get a total scale number from your stated Assumptions (e.g., "100M keys, read-heavy, 500K reads/sec at peak").
2. Pick the ladder rung matching the access pattern (in-memory cache → the 1M rung).
3. Divide: `total ÷ per-node anchor = node count`, order of magnitude only. Say it out loud and move on.

This is the same division as the heuristic above — the ladder just gives you the per-node number to plug in without having to recall a precise spec sheet for every component type.
