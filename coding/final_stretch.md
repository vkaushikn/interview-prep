# Final Stretch — Pattern Reps

Two problems per pattern. Solve cold, no hints. After each, note the key insight.

| Pattern | Problem 1 | Problem 2 |
|---------|-----------|-----------|
| **Sliding Window** | LC 3 — Longest Substring Without Repeating Characters | LC 424 — Longest Repeating Character Replacement |
| **Two Pointers** | LC 11 — Container With Most Water | LC 15 — 3Sum |
| **Binary Search** | LC 33 — Search in Rotated Sorted Array | LC 875 — Koko Eating Bananas |
| **Trees** | LC 236 — Lowest Common Ancestor | LC 124 — Binary Tree Maximum Path Sum |
| **Graphs** | LC 207 — Course Schedule (topo sort) | LC 417 — Pacific Atlantic Water Flow |
| **DP 1D** | LC 300 — Longest Increasing Subsequence | LC 139 — Word Break |
| **DP 2D** | LC 1143 — Longest Common Subsequence | LC 72 — Edit Distance |
| **Backtracking** | LC 39 — Combination Sum | LC 46 — Permutations |
| **Heap** | LC 215 — Kth Largest Element in Array | LC 347 — Top K Frequent Elements |
| **Stack** | LC 739 — Daily Temperatures | LC 84 — Largest Rectangle in Histogram |
| **Linked List** | LC 19 — Remove Nth Node From End of List | LC 143 — Reorder List |

---

## OpenAI-Specific Questions (Reported by Candidates)

These are the actual patterns candidates report seeing. OpenAI leans toward production-like problems with follow-ups that extend scope mid-interview.

| Problem | What it tests | Status |
|---------|--------------|--------|
| LRU Cache (LC 146) | DLL + hashmap, async version | Done |
| Time-based KV Store (LC 981) | Binary search floor | Done |
| Async web crawler | Protocol design, asyncio, visited set | Done |
| **Encode/Decode strings (LC 271)** | Serialization with delimiter-safe encoding (length-prefix) | Todo |
| **Resumable iterator — 1D → 2D → async** | Stateful iterator, `__next__`, then `asyncio` version | Todo |
| **Spreadsheet with cell references** | DAG cycle detection + topological sort for eval order | Todo |
| **Serialize/Deserialize Binary Tree (LC 297)** | BFS/DFS encode to string, parse back | Todo |

### Spreadsheet — the hard one
Cells reference other cells (`A1 = B1 + C2`). Two problems:
1. **Cycle detection** — if A1 = B1 and B1 = A1, that's a cycle. DFS with path set.
2. **Eval order** — topological sort of the dependency DAG. Evaluate leaves first.

This is topological sort applied to a real system. If you know topo sort cold, this is straightforward.

### Encode/Decode strings
Naive: join with delimiter `","`. Breaks if values contain `","`.
Fix: **length-prefix encoding** — `"4#word3#foo"` — prefix each string with its length + separator. Decode by reading the length, then slicing exactly that many chars.

```python
def encode(strs):
    return "".join(f"{len(s)}#{s}" for s in strs)

def decode(s):
    res, i = [], 0
    while i < len(s):
        j = s.index("#", i)
        length = int(s[i:j])
        res.append(s[j+1:j+1+length])
        i = j + 1 + length
    return res
```

---

## Key Pattern Triggers (rapid fire prep)

| Trigger phrase | Pattern |
|----------------|---------|
| "longest/shortest subarray/substring" | Sliding window |
| "sorted array, find pair/triplet" | Two pointers |
| "sorted, O(log N)" | Binary search |
| "binary search on the answer" | Binary search (Koko, capacity) |
| "any two nodes, longest path" | Tree post-order + global tracker |
| "prerequisites, ordering" | Topological sort |
| "shortest path, fewest steps" | BFS |
| "all paths, combinations, subsets" | Backtracking |
| "top K, Kth largest/smallest" | Heap |
| "next greater element" | Monotonic stack |
| "overlapping subproblems, optimal substructure" | DP |
| "two sequences, alignment" | 2D DP |
