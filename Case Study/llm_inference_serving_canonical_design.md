# LLM Inference Serving: Canonical Design Reference

This is the "production systems" skeleton for this problem — the shape of real serving stacks (vLLM, TGI, TensorRT-LLM), each step cross-linked to where it lands in our [ACID writeup](llm_inference_serving_case_study.md) and the relevant [concepts](../concepts/README.md). As with the news feed canonical doc, the goal is to confirm our derivation *covers* the shape — and to name the two places where our writeup stayed at the conceptual level while production systems have a specific, named mechanism.

## The skeleton

| # | Canonical step | Our coverage |
|---|---|---|
| 1 | Requirements: functional (multi-turn streaming chat) + non-functional (TTFT/ITL SLAs, throughput/$) | C |
| 2 | Capacity estimation: model size → GPU memory budget → KV cache budget → max concurrent sequences | A |
| 3 | Request lifecycle: tokenize → prefill → decode loop → detokenize/stream | D — the 6-step pipeline |
| 4 | Static batching → **continuous/in-flight batching** (Orca) | A (assumed as baseline) |
| 5 | Memory management: **PagedAttention** (block-based, non-contiguous KV cache) | **diverges — see below** |
| 6 | Scheduling: admission policy, **chunked prefill**, preemption/swap under memory pressure | I-2/D — **partially diverges, see below** |
| 7 | **Prefix caching / RadixAttention** (shared KV cache across requests) | D — radix tree ✓ |
| 8 | Cache-aware load balancing / routing across replicas | I-1/D — affinity vs. load ✓ |
| 9 | Throughput levers: quantization (weights/KV cache), speculative decoding | excluded — Open items |
| 10 | Multi-GPU: tensor/pipeline parallelism for models too large for one GPU | excluded — Open items |
| 11 | Autoscaling the replica fleet | excluded — Open items (connects to Azure case study) |
| 12 | Observability: TTFT/ITL/throughput/queue-depth/GPU-util dashboards | not covered |

## Where we diverge (step 5): KV cache allocation — contiguous vs. paged

**Canonical (PagedAttention):** each sequence's KV cache is allocated in fixed-size **blocks**, like OS virtual memory pages — not one contiguous buffer sized for the sequence's *maximum possible length*. Blocks are allocated as the sequence grows and reclaimed the instant it ends.

**Ours (A's sizing):** the "~30 concurrent sequences/GPU" estimate implicitly assumed KV cache sized to *actual* (average) sequence length. A naive **contiguous, max-length-reserved** allocator would waste most of that — every sequence reserves room for the worst case it may never reach, so the *real* concurrency without paging is much lower than our headline number.

This is the same shape as **internal fragmentation in memory allocators** — pre-reserving the worst case vs. allocating incrementally on demand. PagedAttention is *why* the "~30" estimate is achievable in practice rather than a theoretical ceiling. Worth naming explicitly: our A-section number is the **paged** number, even though we derived it as if memory were contiguous.

## Where we partially diverge (step 6): admission as binary vs. chunked

**Canonical (chunked prefill):** a long prompt's prefill isn't run in one giant tick (which would stall every in-flight decode for however long that takes, blowing their ITL). Instead, it's **split into chunks** and interleaved with the running batch's decode steps — turning Trade-off I-2 (admit-or-not) into a **tunable dial** (chunk size) rather than a binary decision.

**Ours (I-2/D):** we framed the scheduler's per-tick decision as "admit this request's prefill or don't," driven by roofline position. That's the right *shape* of the trade-off, but chunked prefill is the concrete mechanism that makes it continuous rather than all-or-nothing — and it composes with **preemption/swap** (if memory runs out mid-generation, evict the lowest-priority sequence's KV cache to CPU RAM rather than failing the request — the same park-or-evict tiering from I-3, but triggered by memory pressure rather than idle time).

## Things the canonical skeleton has that we explicitly scoped out
- **Quantization / speculative decoding (step 9):** both are throughput levers that shift the roofline curve itself (quantization reduces bytes-per-token, raising effective arithmetic intensity for decode; speculative decoding turns decode into a verify-multiple-tokens-at-once compute-bound op). Listed as open items — would change the A-section numbers but not the pipeline shape.
- **Multi-GPU parallelism (step 10):** out of scope by A's "single instance, known TP degree" assumption. Would add interconnect bandwidth as a second physical bound in C.
- **Autoscaling (step 11):** the *fixed pool* assumption in C is exactly the Azure case study's reservation-lead-time framing — a natural second pass connecting the two case studies.
- **Observability (step 12):** not covered at all — the TTFT/ITL SLAs in C are the metrics that would be on this dashboard, but we never designed the dashboard/alerting layer itself.
