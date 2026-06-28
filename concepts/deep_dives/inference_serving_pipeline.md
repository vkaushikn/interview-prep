# Inference Serving Pipeline: From Request to Streamed Token

A request entering an LLM serving system passes through several decision points before the first output token streams back, and several more after each turn ends. Each decision point is a capacity-allocation trade-off wearing a different costume — the same shapes that show up in [newsvendor_shortage_overage.md](newsvendor_shortage_overage.md) and the capacity case studies, just at GPU-cluster timescales (milliseconds, not hours).

## The pipeline, end to end

1. **Classifier.** A lightweight pass over the incoming request — cheap to run, used to inform routing (e.g., expected length, which model/tier this needs). The highest-value signal it can carry, though, is **which session this belongs to** — because that determines whether a KV cache for this conversation already exists somewhere.

2. **Router.** Sends the request to a specific node/cluster. Two signals compete here:
   - **Affinity** — send it back to the node holding this session's KV cache (from [kv_cache.md](kv_cache.md)'s prefix-caching idea), so only the *new* tokens need prefilling.
   - **Load** — that node might be busy; sending more work to it queues behind everything else already there.

   Pure affinity → hot-spotting (long-running or popular sessions pile onto one node). Pure load-balancing → destroys locality (every turn pays a full re-prefill on a cold node). Production routers hybridize: prefer the affine node unless it's over some load threshold, then fall back to a cold node and eat the re-prefill cost. This is the **same holding-cost-vs-recompute-cost trade-off** as KV cache eviction policy (below) — both are really asking "where does this session's cache live, and is keeping it there worth what it costs elsewhere?"

3. **Queue (host RAM).** The request waits here before being admitted to the GPU. How expensive it'll be once admitted depends on what state its KV cache is in: still resident in GPU HBM (cheap), swapped to CPU RAM (a swap-in cost), or fully dropped (a full prefill). The queue itself doesn't pay this cost — it's set by whatever happened to this session's cache after its *previous* turn ended (see step 5).

4. **Scheduler admission (continuous batching).** At each tick, the scheduler decides which queued requests join the running batch — based on whether they need prefill or decode, whether their KV cache is hot, and **where the current batch sits on the roofline** (next section). This is the per-tick version of the shortage/overage trade-off: admitting more work raises aggregate throughput but can blow the latency budget of requests already mid-decode.

5. **Decode loop.** Each tick, every active sequence in the batch produces one token, streamed back to its client. This continues until a sequence hits its stop condition for the turn.

6. **Exit / park-or-evict.** When a sequence stops generating, its *compute slot* frees immediately — but its KV cache doesn't have to vanish with it. The serving system chooses to keep it resident (GPU HBM), demote it (CPU RAM swap), or drop it (text-only, full re-prefill on return), based on expected idle time vs. the opportunity cost of holding that memory. This decision is what step 2's "affinity" check is testing on the *next* request from this session — the pipeline is a loop, not a line.

## The Roofline Model: prefill vs. decode

The **roofline model** plots achievable performance against **arithmetic intensity** (FLOPs performed per byte moved from memory). Every workload sits in one of two regimes:
- **Compute-bound**: enough work per byte loaded that the GPU's FLOP throughput is the bottleneck — memory bandwidth sits idle waiting for compute.
- **Memory-bandwidth-bound**: so little work per byte that the GPU spends most of its time waiting on HBM reads — compute units sit idle waiting for data.

**Prefill** (processing the prompt) has *high* arithmetic intensity — many tokens are run through the same weight matrices in parallel, so each byte of weights loaded gets reused across the whole prompt. Prefill tends to be **compute-bound**.

**Decode** (generating one token at a time) has *low* arithmetic intensity — one token's worth of computation, but it still has to read the *entire* KV cache (which only grows) plus the model weights from HBM. Decode tends to be **memory-bandwidth-bound**.

**Why this matters for step 4 (admission):** a batch of decode-only requests is memory-bound — the GPU's compute units are sitting partially idle. Admitting a new request's prefill (compute-heavy) into that tick can be *free* in the sense that it fills otherwise-wasted compute — good for aggregate throughput. But if the batch is already compute-saturated (e.g., many simultaneous prefills, or a very large batch), adding more prefill work steals cycles from the decode steps of in-flight requests, **increasing their inter-token latency** — a tail-latency SLA cost paid by users who were already mid-conversation.

So "where we are on the roofline" isn't a static fact about the model — it's a **per-tick property of the current batch composition**, and it's the input the scheduler needs to answer: *does admitting this request help (fills an idle resource) or hurt (contends for the scarce one)?* Same shortage/overage shape as every other capacity decision in this repo — here the two "costs" are wasted compute (overage, if we don't admit) vs. degraded tail latency for running requests (shortage, if we admit too aggressively).

## See also
- [kv_cache.md](kv_cache.md) — the per-sequence memory cost that step 6's park-or-evict decision is managing, and the prefix-caching mechanism step 2's affinity routing exploits.
- [newsvendor_shortage_overage.md](newsvendor_shortage_overage.md) — the general shortage/overage shape that steps 2, 4, and 6 are all instances of.

First surfaced in: LLM Inference Serving case study (A — sizing the KV-cache memory constraint; this pipeline is the system-level context that constraint sits inside).
