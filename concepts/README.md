# Concepts

A running glossary of concepts encountered while working through system design and interview problems.

---

## System Design

Core patterns for distributed system design interviews.

- [Batch Aggregation Daemon](system_design/batch_aggregation_daemon.md) — raw events → periodic closed-bucket sync → read-optimized aggregates; idempotency via processed flag (Spotify, YouTube, ads, leaderboards).
- [Fan-Out vs. Fan-In](system_design/fan_out_vs_fan_in.md) — broadcast at write time vs. aggregate at read time; decision driven by event frequency × follower count; hybrid for celebrity/normal user split.
- [Top-K Leaderboard](system_design/top_k_leaderboard.md) — Redis sorted sets, worker aggregation for write throughput, time-bucketed windows, ZUNIONSTORE, scatter-gather for non-precomputed cuts.
- [CDN vs. Geo-Distributed Edge Cache](system_design/cdn_vs_edge_cache.md) — CDNs cache shared content; personalized feeds need a regional KV store for locality not amplification.
- [Read-Your-Own-Writes (RYOW)](system_design/read_your_own_writes.md) — staleness/propagation-delay between a write and the read path seeing it.
- [Merging a Sorted Stream Without Re-sorting](system_design/merge_sorted_stream_no_resort.md) — process new items in time order, head-push/tail-pop into affected lists — O(1), no sort.

---

## Approximations

Numbers to reason with live — not to memorize precisely.

- [Order-of-Magnitude Capacity Benchmarks](approximations/capacity_benchmarks.md) — per-node throughput rules of thumb (cache/DB/queue), the ÷10 ladder, row-size estimation buckets.
- [Latency Numbers Worth Knowing](approximations/latency_numbers.md) — human perception thresholds and the memory/SSD/network access ladder.

---

## OR Concepts

Operations research patterns for DS/supply chain interviews (lower priority).

- [Newsvendor: Shortage vs. Overage Cost](or_concepts/newsvendor_shortage_overage.md) — the universal capacity-allocation cost structure.
- [Time-Expanded Networks for Lead Times](or_concepts/time_expanded_network_lead_times.md) — model transit/lead time as arcs between (location, time) nodes.
- [Countdown Chains for Deterministic Sojourn Time](or_concepts/countdown_chain_deterministic_sojourn.md) — track remaining commitment duration, not start time.
- [Frozen / Firm / Free Zones in Rolling-Horizon Planning](or_concepts/frozen_firm_free_zones.md) — decisions with execution lead time become fixed inputs to future re-solves.

---

## Deep Dives

Interesting background knowledge — good to know, not critical path for interviews.

- [Media Delivery (Audio & Video)](deep_dives/media_delivery.md) — progressive download vs. DASH/ABR; pre-signed URLs; CDN pre-warming; playback resume state.
- [CRDTs vs. Operational Transformation](deep_dives/crdt_vs_ot.md) — two answers to real-time collaborative-edit conflicts.
- [KV Cache: Per-Sequence Memory in LLM Inference](deep_dives/kv_cache.md) — why each active request holds growing GPU memory, and how prefix caching shares it.
- [Inference Serving Pipeline](deep_dives/inference_serving_pipeline.md) — classifier/router/queue/scheduler/decode/eviction loop; roofline model for prefill vs. decode.
- [Reference vs. Denormalization in a Precomputed Cache](deep_dives/reference_vs_denormalization.md) — store an ID vs. the full value — same shape as push vs. pull.
