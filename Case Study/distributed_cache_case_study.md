# Case Study: Distributed Cache

## Problem Statement
Design an in-memory key-value store that scales horizontally across many nodes, sitting in front of a slower backing store (DB), to serve a read-dominated workload with low latency.

---

## A — Assumptions & Approximations

### Assumptions (scope)
- **Cache-aside pattern** — this is not the source of truth; data loss on restart is acceptable, since the backing store still owns the data.
- **Read-dominated** workload (~95/5 read:write split assumed) — this is *why* distributing reads across nodes is worth doing at all.
- **TTL-based expiry** required.
- **Capacity-bounded memory** — needs an eviction policy once full.
- **Single region for v1** — cross-region replication is an explicit extension, not in scope.

### Approximations (sizing)
| Quantity | Estimate |
|---|---|
| Total keys | 50M |
| Avg value size | ~1KB |
| Replication factor | 2 |
| Peak read QPS | 500K/sec |

**Derived:**
- Raw data: 50M × 1KB ≈ 50GB; with RF=2 → ~100GB total.
- Per-node memory budget ~40GB usable → **~3 nodes minimum, driven by capacity, not throughput.**
- Checked against the ÷10 ladder ([capacity_benchmarks.md](../concepts/capacity_benchmarks.md)): in-memory ops ≈ 1M/sec/node; 500K/sec peak fits comfortably inside a single node's op budget. **The binding constraint is memory capacity, not request throughput** — worth stating explicitly, since it changes the answer to "what if QPS doubles?" (answer: doesn't move node count; data size does).

---

## C — Constraints
- **CAP: prefer AP over CP.** A strongly-consistent cache is a slower, harder-to-operate version of the DB it's supposed to offload. Staleness is the price of the speedup.
- **Latency target:** sub-ms to low single-digit ms reads — the entire value proposition versus hitting the DB directly.
- **Memory-bound, not just throughput-bound.** The real ceiling is usually "does the working set fit in aggregate RAM," not raw ops/sec.

---

## I — Incentives & Trade-offs
- **Exact global LRU vs. per-node approximate eviction:** accept a small hit-ratio precision loss to avoid a cluster-wide synchronization bottleneck. (There's no single "overall capacity" to track once sharded — each leader owns its own slice of keyspace and its own memory bound; a global LRU structure has no coherent referent.)
- **Hot-key handling:** replicate a hot key onto extra nodes at the cost of slightly staler reads on just that key, rather than slowing down the whole shard.
- **Recency (LRU) vs. frequency (LFU) as the eviction signal:** LFU alone has an aging problem — a key popular yesterday but cold today still "wins" on raw count — unless paired with decay.

---

## D — Decisions (Architectural Blueprint)

### D1. Sharding
Consistent hashing ring, ~100-200 virtual nodes per physical node. Bounds key movement to ~1/N of keyspace on node join/leave, instead of reshuffling everything (the failure mode of plain `hash % N`).

### D2. Replication
Each shard = 1 leader + M async followers, for read scaling and failover.

### D3. Eviction
Per-node, approximate LRU: sample K random keys, evict the oldest of the sample. No synchronized ordering structure to maintain — contrast with *exact* LRU (hashmap + doubly-linked list, O(1) get/evict, but every read mutates shared structure). Sampling trades exactness for keeping the hot path (reads) free of contention; pay the cost on the cold path (eviction) instead.

### D4. TTL
Lazy expiry on read (treat as a miss if stale) + background sweep per node. Tolerate small clock-skew slop (NTP-synced local clocks) rather than chasing exact expiry across nodes.

### D5. Routing
Ring metadata (vnode → owning node) is replicated to every client, or to a fleet of stateless proxies — the hash+lookup is parallelized across many independent processes rather than centralized in one router, which would recreate the single-bottleneck problem sharding was meant to solve. Topology propagates via a coordination service or gossip; staleness self-heals via redirect-on-wrong-node.

---

## Failure Modes

1. **Shard leader failure.** Followers detect missed heartbeats → promote a follower via leader election (Raft-style term/epoch, or an external coordinator). The dangerous case isn't a clean crash — it's a leader that's merely *partitioned*, not dead, and keeps accepting writes during election. Fenced off via epoch numbers, so the old leader's writes are rejected once it reconnects. This is the CAP trade-off recurring concretely: block writes during election (consistency cost) vs. let the old leader keep serving (availability cost, risk of conflicting writes).

2. **Node join/leave triggers a cache stampede.** Consistent hashing bounds the blast radius to ~1/N of keys, but those keys become simultaneous misses — if they all hit the backing DB at once, the cache can take down the thing it was protecting. Mitigated with request coalescing/single-flight at the gateway (one in-flight DB request per key) and gradual, not all-at-once, ring rebalancing.

---

## Metrics & Debugging

| Metric | What it tells you | Safe operating region | Collected how |
|---|---|---|---|
| Hit rate (per shard, **node-side**) | Is the cache doing its job | Watch the trend, not the absolute number — falling at flat QPS signals a problem | Counter at the node — it already knows hit-vs-miss as part of serving the request; per-shard breakdown is what catches an isolated problem (e.g. one follower not getting writes) that a client-side aggregate would blur away |
| Latency p50/p99/p999 | User-facing experience | p99 within low single-digit ms | Per-node histogram — every request's (end − start) duration increments a bucket (0-1ms, 1-2ms, ...) live; buckets are **scraped** every N minutes (continuous collection, periodic reporting). Percentiles are read off cumulative bucket counts; never average pre-computed percentiles across nodes — merge histograms first |
| QPS per shard | Scale-out signal + leading indicator of a *traffic*-hot shard | ~50-70% of the ~1M ops/sec/node ladder anchor, leaving failover headroom | Counter, rate computed per interval — a time series, not itself a histogram (though percentiles *of* that series, e.g. p99 of 1-min QPS over a week, are a useful secondary capacity-planning number) |
| Memory utilization % | Capacity pressure | <80% | Gauge |
| Eviction rate | Capacity/**cardinality** pressure — too many distinct keys for this node. A different failure than QPS-hot: reads alone don't evict, only new keys arriving at a full node do | Tracks input write rate at steady state; a spike with flat write rate signals TTL misconfig or cardinality growth | Counter |
| Replication lag | Failover readiness + follower read staleness | Single-digit ms | Leader stamps writes with offset/timestamp; follower reports the delta |
| Skew ratio (heaviest node ÷ median node) — run separately on QPS, memory, and eviction | Distinguishes traffic-hot vs. memory-hot vs. cardinality-hot shards — these are different failures with different fixes | Close to 1.0 | Computed from per-node metrics, not collected directly |
| Redirect rate | Stale routing metadata across the client/proxy fleet | Near zero outside an active rebalance | Counter, incremented when a node returns a "wrong node, try X" redirect |

**Collection architecture:** pull-based (Prometheus-style) — each node exposes current counter/gauge/histogram state; a collector scrapes on an interval. Decoupled, so a slow collector never backs up the cache's hot path. (Push-based would recreate the exact "don't centralize the hot path" mistake routing was designed to avoid — just for metrics instead of cache traffic.)

**Debugging pattern:** almost no single metric identifies a root cause alone — each one narrows the search, you read 2-3 together:
- p99 latency up, hit rate flat → check memory/eviction on the specific node (churn, not a workload problem).
- Hit rate down, latency flat → check working-set size vs. capacity math, or TTL config.
- One shard's QPS or latency far above the rest → check the skew ratio — a hot-key/hot-shard signature, not a capacity problem; adding nodes won't fix it.
- Redirect rate elevated past a rebalance window → propagation to clients/proxies is lagging.
- Replication lag climbing → leader's write rate has outpaced async replication; early warning for data loss on failover.

---

## See also
- [capacity_benchmarks.md](../concepts/capacity_benchmarks.md) — the ÷10 ladder mnemonic used in the Approximations section above.

## Open items / things to revisit
- LFU-with-decay as an alternative/complement to LRU — flagged as a trade-off, not designed in detail.
- Cross-region replication — explicitly scoped out of v1.
- The eviction-pool refinement over one-shot sampling (keeping the best candidates seen across multiple sampling rounds, rather than discarding them after one round) — mentioned but not worked through.
