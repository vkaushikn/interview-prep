# Case Study: Social Media News Feed (Reverse-Chronological)

## Problem Statement
Design the system that shows a user their home feed: posts from people they follow, in reverse-chronological order.

---

## A — Assumptions & Approximations

### Assumptions (scope)
- **Text-only status posts** — no images/video/recommendation systems.
- **Purely reverse-chronological** — no "For You" / ranking algorithm.
- **Only posts from people you follow** — no discovery feed.
- **Show the last N (~20) posts initially**; deeper history is "infinite scroll," handled as a separate concern (see D5).
- **Write/posting path is out of scope** — assume new posts are "magically" available in storage the moment they're created. This explicitly defers the **read-your-own-writes** problem (see [concepts/read_your_own_writes.md](../concepts/read_your_own_writes.md)).

### Approximations (sizing)
| Quantity | Estimate |
|---|---|
| Daily Active Users (DAU) | 200M (2e8) |
| Avg. follows per user (= avg. followers, by definition) | 200 |
| Avg. posts per active user / day | 2 |
| Avg. feed views per user / day | 10 |
| Feed entry size (denormalized, incl. post content + metadata) | ~500B–1KB |

**Derived (using N_users × per-user-rate / 1e5 sec/day):**
- Writes: 200M × 2 / 1e5 ≈ **4K posts/sec**
- Feed-view requests: 200M × 10 / 1e5 ≈ **20K req/sec**
- **Read:write ratio ≈ 5:1**

**Follower distribution:** long-tail/power-law. Most users have well under ~1,000 followers; a tiny fraction ("celebrities") have millions. A "celebrity" threshold (~10K–100K followers) is assumed, to be tuned empirically against real data.

---

## C — Constraints

- **Latency:** target P99 ≈ 200-300ms for a feed load (human-perception threshold; see [concepts/latency_numbers.md](../concepts/latency_numbers.md)).
- **Geography:** cross-region round trips (~150-300ms) can dominate the entire latency budget on their own — data placement near the reader is not optional.
- **Freshness:** feed contents fresh to within ~5 minutes — this bounds how "behind" the creation pipeline (daemon + propagation to edges) is allowed to be. It is *not* the size of any buffer (see D3).
- **Per-node physical bounds** (see [concepts/capacity_benchmarks.md](../concepts/capacity_benchmarks.md)):
  - In-memory cache: ~100K ops/sec/node, ~O(100GB)/node
  - Disk-backed DB, indexed query: ~1K-10K qps/node
  - Network: ~1-10 Gbps/node (~125MB-1.25GB/s)

---

## I — Incentives & Trade-offs

1. **Fan-out-on-write (push) vs. fan-out-on-read (pull).** Pull recomputes each feed at view-time (cheap writes, expensive reads); push precomputes feeds when posts are written (expensive writes, cheap reads). With read:write ≈ 5:1, and the feasibility numbers below, push wins for the common case — but push cost scales with **followers-per-poster**.

2. **The celebrity exception.** Push cost is O(followers). A normal user (~200 followers) is cheap; a celebrity (millions of followers) turns *one post* into millions of fan-out writes — breaking the push model's budget on its own. Needs a hybrid.

3. **Infinite scroll vs. precomputation cost.** Precomputing/storing deep history for all 200M users is wasteful if most users never scroll past page 1. Tradeoff: precompute more (uniformly fast, costly upfront) vs. precompute less + compute the tail on demand (cheap upfront, slower only for users who actually go deep).

4. **Freshness vs. daemon cost.** More frequent daemon runs = fresher feeds, more compute. The 5-minute SLA is the chosen operating point, not a law of physics.

---

## D — Decisions (Architectural Blueprint)

### D1. Serve/Create decoupling
- **Serving layer:** geo-distributed regional edge caches — a KV store keyed by user ID, holding each user's precomputed feed, placed near the reader. (Explicitly *not* a CDN — feeds are personalized, not shared content; see [concepts/cdn_vs_edge_cache.md](../concepts/cdn_vs_edge_cache.md).) One O(1) read per feed-view.
- **Creation layer:** a periodic daemon. The 5-minute freshness SLA is the budget for (daemon lag + propagation to edges) end-to-end — not a literal buffer size.

### D2. Core data structures
| Structure | Purpose | Size | Verdict |
|---|---|---|---|
| `user → [followers]` | fan-out graph | 200M × 200 × 8B ≈ **320GB** | Easy (~4-10 nodes) |
| `user → [last N posts authored]` | bootstrap new follows + celebrity merge | small, per-user | Trivial |
| Chronological log of new posts since last checkpoint | daemon input | ~240K posts/min × 500B ≈ **120MB/min** | Trivial |
| `user → [feed]` (precomputed, denormalized) | served artifact | 200M × 20 × ~500B-1KB ≈ **2-4TB** | Doable (~20-40 nodes) |

### D3. Daemon fan-out mechanism
- Maintain a **checkpoint / high-water-mark** (not a fixed time bucket) — process posts since the last checkpoint exactly once, advance the checkpoint after.
- Process new posts **in chronological order**. For each post from a non-celebrity author: push to the **head** of each follower's feed deque, **pop the tail** — O(1), no re-sort. Correctness depends on chronological processing order (see [concepts/merge_sorted_stream_no_resort.md](../concepts/merge_sorted_stream_no_resort.md)).
- **Celebrity posts are skipped entirely** by the daemon — they live only in the author's-recent-posts cache (D2 row 2).
- Delta propagated to edges: ~800K fanout-touches/sec × 60s ≈ 48M updated entries/min × ~500B-1KB ≈ **24-48GB/min ≈ 400-800MB/sec**. Doable against ~1-10Gbps/node, but real infrastructure (a handful of links).

### D4. Celebrity hybrid (resolves I-2)
- At **serve time** (not in the daemon): for each feed request, also fetch the small list (≤~10) of celebrities the user follows, pull their recent posts (O(1) cache reads each from D2 row 2), and merge into the returned feed.
- Cost: 20K/sec × ~10 ≈ 200K/sec against an in-memory cache (~2 nodes). Bounded and tiny — does not violate the serve/create decoupling, because the extra work is O(few), not O(followers).

### D5. Infinite-scroll hybrid (resolves I-3)
- Precompute/push only the first N (tunable — 20, 40, etc., empirically tuned against scroll-continuation rate, same flavor as the operating-curve tuning in the Azure case).
- Beyond N: a separate **pull-based pagination service** queries each of the user's ~200 followees' post tables for `post_id < cursor`, live k-way-merges the ~200 sorted result sets, returns the next page.
- Estimated load: assume ~10% of feed-views continue scrolling → 2K/sec × 200 ≈ **400K reads/sec** → ~40-400 DB nodes. An order of magnitude better than naive pull-everything (4M/sec), because it's gated by actual behavior.
- Celebrities beyond N are handled by this same pull path — no extra special-casing.

---

## Numbers & Feasibility Summary

| Quantity | Value | Per-node benchmark | Node count | Verdict |
|---|---|---|---|---|
| Feed-view requests | 20K/sec | in-memory cache, ~100K/sec | ~1 (+ redundancy) | **Easy** |
| Naive pull-everything reads | 4M/sec | disk DB, ~1K-10K qps | ~400-4,000 | **Infeasible — design problem, not capacity** |
| Push fan-out writes | 800K/sec | queue/broker, ~100K-1M/sec | ~1-10 | **Doable, real infra** |
| Follower graph storage | 320GB | cache, ~100GB/node | ~4-10 | **Easy** |
| Precomputed feed storage | 2-4TB | cache, ~100GB/node | ~20-40 | **Doable** |
| Feed delta to edges | 400-800MB/sec | network, ~1-10Gbps/node | a handful of links | **Doable, real infra** |
| Celebrity merge at serve time | 200K/sec | in-memory cache, ~100K/sec | ~2 | **Easy** |
| Infinite-scroll pull (tail) | 400K/sec | disk DB, ~1K-10K qps | ~40-400 | **Doable, real infra** |

This table is the running evidence for every architectural choice above: numbers in the "Easy"/"Doable" range validate the choice; the one "Infeasible" row (naive pull) is *why* push was chosen at all.

---

## Open items / things to revisit
- Freshness SLA reopens the read-your-own-writes question deferred in A.
- Tie-breaking for same-timestamp posts in the chronological merge.
- Daemon checkpoint bookkeeping for exactly-once processing across restarts.
- N (initial feed size) and the celebrity threshold are both empirically-tuned parameters, not fixed constants.
