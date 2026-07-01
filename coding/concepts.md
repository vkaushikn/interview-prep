# Coding Concepts

## Useless Things To Know (No Bearing On Actual Job)

- **LRU Cache with DLL** — in practice you use `functools.lru_cache`, `collections.OrderedDict`, or Redis. The one job where you'd implement this is writing a cache library, which is not most jobs.
- **Reversing a linked list** — no production codebase does this manually. You'd use a deque or just reverse a list with `[::-1]`.
- **malloc internals** — unless you're writing a memory allocator, which you are not.

---

## Why Doubly Linked List for LRU Cache

The hashmap gives O(1) access to a node by key. Once you have the node, you need O(1) delete.

**Singly linked list**: to delete a node you need its predecessor (`predecessor.next = node.next`). You only have the node, not its predecessor — so you traverse from head. O(N).

**Doubly linked list**: `node.prev` is right there.
```
node.prev.next = node.next
node.next.prev = node.prev
```
O(1). No traversal.

**The combination**: hashmap can't maintain order. Linked list can't find by key. Together:
- Hashmap: key → node pointer, O(1)
- DLL: given pointer, insert/delete/move O(1)

Neither solves LRU alone. Together they cover each other's weakness.

**DLL operation complexity**:
| Operation | Complexity | Condition |
|-----------|-----------|-----------|
| Access by pointer | O(1) | You have the node |
| Insert (given neighbor) | O(1) | Pointer surgery |
| Delete (given pointer) | O(1) | Pointer surgery, need prev |
| Search by value | O(N) | Must traverse |
| Access by index | O(N) | Must traverse |

## LRU vs LFU

- **LRU without DLL**: heap with lazy deletion works but O(log N) and stale entries accumulate
- **LFU without O(1) constraint**: simpler than LRU — just `min(cache, key=lambda k: freq[k])`, O(N) but trivial
- **LFU at O(1)**: hardest — three structures in sync: `key→(val,freq)`, `freq→DLL of keys`, `min_freq` tracker
