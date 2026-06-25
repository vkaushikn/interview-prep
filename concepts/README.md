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
- [CDN vs. Geo-Distributed Edge Cache](cdn_vs_edge_cache.md) — CDNs cache shared content; personalized feeds need a regional KV store keyed by user, for locality not amplification.
- [Merging a Sorted Stream Without Re-sorting](merge_sorted_stream_no_resort.md) — process new items in time order, head-push/tail-pop into affected lists — O(1), no sort.
- [Reference vs. Denormalization in a Precomputed Cache](reference_vs_denormalization.md) — store an ID (extra read, single source of truth) vs. the full value (one read, fan-out on update) — same shape as push vs. pull.
- [Newsvendor: Shortage vs. Overage Cost](newsvendor_shortage_overage.md) — the universal capacity-allocation cost structure; includes the supply-side mirror of demand-unconstraining (yield/show-up rates).
- [Time-Expanded Networks for Lead Times](time_expanded_network_lead_times.md) — model transit/lead time as arcs between (location, time) nodes — turns lead time into ordinary flow-conservation bookkeeping.
- [Countdown Chains for Deterministic Sojourn Time](countdown_chain_deterministic_sojourn.md) — track remaining commitment duration, not start time — exit is automatic when the countdown hits zero.
- [Frozen / Firm / Free Zones in Rolling-Horizon Planning](frozen_firm_free_zones.md) — decisions with execution lead time become fixed inputs to future re-solves, preventing hourly plan-thrashing.
- [KV Cache: Per-Sequence Memory in LLM Inference](kv_cache.md) — why each active request holds growing, per-sequence GPU memory, and how prefix caching shares it across requests (reference vs. denormalization, again).
- [Inference Serving Pipeline: From Request to Streamed Token](inference_serving_pipeline.md) — classifier/router/queue/scheduler/decode/eviction loop, and the roofline model (prefill = compute-bound, decode = memory-bound) driving per-tick admission decisions.
- [CRDTs vs. Operational Transformation](crdt_vs_ot.md) — two answers to real-time collaborative-edit conflicts: peer-to-peer mergeable identifiers (CRDT) vs. a central sequencer + transform function (OT); plus site_id vs. user_id layering and the offline-divergence wall.
