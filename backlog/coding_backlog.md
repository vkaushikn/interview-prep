# Coding Backlog

Problems to do after the core two-week plan.

## LRU Cache — LC 146 (Medium)

Two implementations to know:
1. `OrderedDict` + `move_to_end` — quick, clean, but requires knowing the API
2. Doubly linked list + hashmap — what Staff interviews want. Sentinel head/tail to avoid edge cases. O(1) get, put, evict.

This is an IYKYK problem. Not reasonably derivable cold. Memorize the DLL approach.

---

## OpenAI Interview Priority List

### Done
- LRU Cache (LC 146) — DLL + asyncio thread-safe version
- Async web crawler — Protocol-based, VisitedUrls, asyncio.Queue, task_done/join
- Time-based Key-Value Store (LC 981) — binary search floor variant
- Flatten Nested List Iterator (LC 341) — eager flatten with recursive _flatten
- In-Memory File System (LC 588) — Trie approach, _walk helper

### Todo — Tier 1
- Connection pool — `asyncio.Semaphore(pool_size)`, acquire/release pattern
- Python generators — `yield`, `yield from`, inorder tree generator
- LFU Cache (LC 460) — three structures: `key→(val,freq)`, `freq→DLL`, `min_freq`
- LC 33 — Search in rotated sorted array
- LC 875 — Koko eating bananas (binary search on answer)

### Todo — Tier 2
- Mini SQL — in-memory dict of tables, string parsing, SELECT with WHERE
- Unix FS with symlinks — Trie + cycle detection
- IP address iterator — stateful iterator, subnet math
- Partition equal subset sum — 0/1 knapsack DP
- Edit distance / LCS — 2D DP strings
