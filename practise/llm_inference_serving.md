# Case Study: LLM Inference Serving

## Problem Statement

> Design a system that serves chat-style LLM inference: many concurrent users send messages (often part of a longer multi-turn conversation), and the system streams generated tokens back in real time, on a fixed pool of GPUs.

This is a **System Design** problem, not an OR/business-case one — per [system_design_acid.md](../frameworks/system_design_acid.md)'s Step 0 note, the decomposition here (prefill vs. decode, per-session affinity vs. load balancing, tiered KV cache storage) **emerges through A/C/I** rather than being a visible up-front fork. No Step 0 needed.

---

## A — Assumptions & Approximations

### Assumptions (scope)
- Serving only — the model is already trained and loaded; training/fine-tuning is out of scope.
- **Chat/conversational workload**: multi-turn sessions, decoder-only transformer, autoregressive (one token at a time) generation.
- **Continuous batching** as the baseline serving strategy (industry standard — vLLM, TGI, etc.), not static/fixed batching.
- **Prefix caching** available (radix-tree-style KV cache sharing across requests with a common prefix — see [kv_cache.md](../concepts/kv_cache.md)).
- A single model instance fits on a known number of GPUs (tensor-parallel degree fixed); routing across *different models* is out of scope.

### Approximations (sizing)
| Quantity | Estimate |
|---|---|
| GPU HBM capacity | ~80GB (H100-class) |
| Model weights (13B, fp16) | ~26GB |
| KV cache per token (2 × layers × hidden_dim × 2 bytes, K+V) | ~0.8MB (40 layers, hidden 5120) |
| HBM remaining for KV cache after weights | ~50GB |
| **Derived: concurrent sequences per GPU** (50GB ÷ 0.8MB ÷ ~2000-token avg context) | **~30** |
| HBM bandwidth (H100-class) | ~3TB/s |
| **Derived: decode tick time** (weights read per step ÷ bandwidth ≈ 26GB ÷ 3TB/s) | **~9ms/tick** → aggregate ~110 tok/s across the batch, ~3-4 tok/s/sequence |
| Conversation "think time" between turns | seconds to minutes — bursty, not continuous |

The **"~30 concurrent sequences per GPU"** number is the headline finding from [kv_cache.md](../concepts/kv_cache.md): KV cache, not compute, is what runs out first. Everything in C and D is downstream of this.

---

## C — Constraints

- **Physical bounds:**
  - **HBM capacity** — binding constraint on concurrency (the ~30-sequence number above). Once weights + KV cache fill HBM, no more sequences can be admitted regardless of spare compute.
  - **HBM bandwidth** — binding constraint on *decode* speed (the roofline's memory-bound regime; see [inference_serving_pipeline.md](../concepts/inference_serving_pipeline.md)).
  - **Interconnect bandwidth** — relevant if tensor/pipeline parallelism spans multiple GPUs (out of scope for sizing here, but real for 70B+ models).
- **Operational SLAs:**
  - **TTFT (time-to-first-token), P99** — dominated by prefill (compute-bound).
  - **ITL (inter-token latency), P99** — dominated by decode (memory-bound); this is the "tok/s/sequence" number users actually perceive as reading speed.
- **Resource availability:** the GPU pool is fixed at request time — no elastic scale-up within a request's lifetime (provisioning new nodes has lead time, same as the Azure capacity case study's reservation lead times).

---

## I — Incentives & Trade-offs

1. **Routing: affinity vs. load.** Sending a request back to the node holding its session's KV cache avoids re-prefilling the whole conversation — but that node might be the busiest one. Pure affinity → hot-spotting; pure load-balancing → every turn pays full re-prefill. **Same shape as #3** (below): both are "where does this session's state live, and is keeping it there worth the cost elsewhere?"

2. **Admission/batching: throughput vs. in-flight latency.** Continuous batching wants to pack the GPU full (maximize aggregate throughput/$). But admitting a new request's prefill (compute-heavy) into a decode-heavy batch (memory-bound, idle compute) can be *free* — or, if the batch is already compute-saturated, it **steals cycles from in-flight decodes**, degrading their ITL. This is a **per-tick shortage/overage trade-off** (see [newsvendor_shortage_overage.md](../concepts/newsvendor_shortage_overage.md)): under-admit → wasted compute (overage); over-admit → SLA breach for running requests (shortage).

3. **KV cache eviction: holding cost vs. recompute cost.** After a turn ends, keeping the session's KV cache resident (GPU HBM, or CPU RAM) costs memory that an *actively waiting* request could use now; dropping it costs a full re-prefill if the user replies soon. The right policy is a **time-decaying bet** — resident for seconds, CPU-swapped for minutes, dropped after longer idle — exactly the tiered-storage pattern, now driven by *expected idle duration* rather than a fixed threshold.

4. **High utilization vs. failover/latency headroom.** Running the GPU pool near 100% utilized maximizes throughput-per-dollar, but leaves no slack to absorb a burst — every new arrival queues, degrading TTFT for *everyone*, not just the new request. This is the System Design framework's "High Cluster Utilization vs. Failover Headroom" axis, concretely instantiated.

---

## D — Decisions

### Technical blueprint
The pipeline from [inference_serving_pipeline.md](../concepts/inference_serving_pipeline.md), end to end:

**Classifier → Router (affinity/load hybrid) → Queue (host RAM) → Scheduler admission (continuous batching, roofline-aware) → Decode loop (streamed tokens) → Park-or-evict → (feeds back into Router for this session's next turn)**

### Control mechanism: the admission controller
The scheduler's per-tick admission decision is the system's **real-time control loop** (the ACID framework's "PID-style controller" pattern, here applied per-tick instead of per-hour): each tick, given the current batch's roofline position (compute-bound or memory-bound right now) and remaining KV-cache headroom, decide whether to admit queued prefill work — trading aggregate throughput against the ITL SLA of in-flight decodes (Trade-off #2).

### State & storage
- **Three-tier KV cache storage**: GPU HBM (resident, active or recently-idle) → CPU RAM (swapped, idle) → dropped (text-only, cold) — the time-decaying policy from Trade-off #3.
- **Prefix-cache radix tree**: shared KV cache across sessions/requests with common prefixes (system prompts, shared conversation history) — the "reference, not denormalization" move from [reference_vs_denormalization.md](../concepts/reference_vs_denormalization.md), applied to GPU memory instead of a database.
- **Session-affinity routing table**: maps session ID → node currently holding (or last holding) its KV cache, consulted by the router (Trade-off #1) and updated by the park-or-evict step.

---

## See also
- [inference_serving_pipeline.md](../concepts/inference_serving_pipeline.md) — the pipeline and roofline model this case study's D section is built on.
- [kv_cache.md](../concepts/kv_cache.md) — the per-sequence memory constraint underlying the A-section sizing.
- [newsvendor_shortage_overage.md](../concepts/newsvendor_shortage_overage.md) — the shortage/overage shape behind Trade-offs #2 and #3.
- [reference_vs_denormalization.md](../concepts/reference_vs_denormalization.md) — the prefix-caching mechanism in D.
- [practice_modes.md](../frameworks/practice_modes.md) — this case study was built in Teacher mode; a Critique-mode pass (e.g., "a celebrity-scale session monopolizes a node's KV cache — what breaks?") is a natural follow-up.

## Open items / things to revisit
- Multi-GPU tensor/pipeline parallelism (interconnect bandwidth as a second physical bound) for models too large for one GPU.
- Throughput levers not modeled here: speculative decoding, quantization, model distillation.
- Autoscaling the GPU pool itself has the same provisioning-lead-time shape as the Azure capacity case study — could be a second pass connecting the two.
