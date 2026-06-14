# KV Cache: Per-Sequence Memory in LLM Inference

**The problem:** generating each output token requires the transformer to "attend to" every previous token in the sequence (prompt + everything generated so far). Recomputing the attention inputs (key/value vectors) for the whole sequence at every step would be wastefully expensive — so the model **caches** each token's key (K) and value (V) vectors, per layer, the first time they're computed. Each new token then only costs *incremental* work: compute K,V for the new token, attend against the cache.

## Why it's a *memory* problem, not just a compute optimization
The K,V vectors for a token at position *i* are computed from that token's **hidden state**, which depends on the entire preceding context (through all prior layers' attention). So:
- Two sequences with the *same token* at the same position but **different preceding context** produce **different K,V** — the cache is genuinely per-sequence.
- KV cache size is proportional to **(sequence length so far) × (number of concurrent sequences)** — unlike model weights (a fixed, one-time cost), it's a **per-request, growing-over-time** consumer of GPU memory (HBM).

This makes KV cache the **binding capacity constraint** for batching, distinct from compute: you can run out of memory (too many long-running sequences held concurrently) before you run out of compute throughput. See the LLM Inference Serving case study for how this turns "make batches bigger" into a genuine resource-allocation problem, not just a latency/throughput trade-off.

## The exception: prefix caching (reference vs. denormalization, again)
If two sequences share an **identical prefix** (same system prompt, same unchanged conversation history), the K,V for that shared prefix are **identical** by the same logic — identical preceding context → identical hidden states → identical K,V. Production systems (e.g., vLLM's prefix caching / radix-tree KV cache) detect and **share** this portion across requests instead of duplicating it.

This is the same trade-off as [reference_vs_denormalization.md](reference_vs_denormalization.md): store each sequence's KV cache fully independently (simple, no dedup bookkeeping) vs. detect shared prefixes and share that memory + the compute to produce it (more bookkeeping, real savings when many requests share a system prompt or conversation prefix).

First surfaced in: LLM Inference Serving case study (A — sizing; the KV-cache memory constraint on batch size).
