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

---

## asyncio — How It Works

**Your model (correct):** There is a central event loop. When you `await` something, the current function suspends and the event loop runs something else. When the awaited thing returns, your function resumes where it left off.

**One correction:** it is not a queue of function calls. It is a scheduler of *suspended coroutines*. The event loop does not fire all coroutines at once — each one runs until it voluntarily suspends at an `await`, then hands control back. The event loop picks the next coroutine that is ready to run.

```
event loop tick:
    pick a coroutine that is ready
    run it until it hits `await <some I/O>`   ← suspends here, hands control back
    put it in "waiting" state
    pick next ready coroutine
    ...
    when I/O returns → mark that coroutine as ready again
    pick it up on the next tick
```

**Your 50 cache gets example — exactly right:**
```python
results = await asyncio.gather(*[get_from_cache(k) for k in keys])
```
`asyncio.gather` schedules all 50 coroutines. Each runs until it hits `await redis.get(...)` and suspends. The 40 hot ones come back fast — their coroutines resume immediately. The 10 cold ones go to DB — their coroutines stay suspended until the DB responds. No thread-per-request. One thread, 50 concurrent operations in flight.

**The key insight:** `await` only yields at I/O boundaries (network, disk, sleep). Pure CPU work (pointer surgery, dict lookup) never yields — it runs to completion without interruption. This is why Node methods don't need `async`: they never block on I/O.

---

## asyncio.Lock — How It Works

**Your model (correct):** whoever acquires the lock first runs; everyone else blocks until the lock is released.

**But "blocking" here means coroutine-blocking, not thread-blocking.**

```python
async def get(self, key):
    async with self.lock:       # if lock is held → this coroutine suspends
        ...                     # other coroutines run while this waits
                                # when lock is released → this resumes
```

With `threading.Lock`, a blocked thread is an OS thread doing nothing — wasted. With `asyncio.Lock`, a "blocked" coroutine just suspends — the event loop runs other coroutines in its place. One thread, still making progress.

**The cost of the lock on LRU cache:** the entire `get` or `put` is CPU-only (no I/O). So in practice the lock is held for microseconds — contention is negligible. The lock is correctness insurance, not a performance concern here.

---

## LRU Cache — Thread-Safe Version (asyncio)

```python
import asyncio

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.mapping: dict[int, Node] = {}
        self.lock = asyncio.Lock()
        self.HEAD = Node(key=-1, value=-1, predecessor=None, successor=None, is_head=True, is_tail=False)
        self.TAIL = Node(key=-1, value=-1, predecessor=None, successor=None, is_head=False, is_tail=True)
        self.HEAD.successor = self.TAIL
        self.TAIL.predecessor = self.HEAD

    async def get(self, key: int) -> int:
        async with self.lock:
            if key in self.mapping:
                node = self.mapping[key]
                Node.move_to_tail(node, self.TAIL)
                return node.value
            return -1

    async def put(self, key: int, value: int) -> None:
        async with self.lock:
            if key in self.mapping:
                node = self.mapping[key]
                node.value = value
                Node.move_to_tail(node, self.TAIL)
                return
            if len(self.mapping) < self.capacity:
                node = Node.append_tail(self.TAIL, key, value)
                self.mapping[key] = node
            else:
                evict_node = Node.remove_from_head(self.HEAD)
                self.mapping.pop(evict_node.key)
                node = Node.append_tail(self.TAIL, key, value)
                self.mapping[key] = node
```

Node methods (`append_tail`, `move_to_tail`, `remove_from_head`) stay synchronous — pure pointer surgery, no I/O, never yields.

Use `threading.Lock` + regular `def` if your server uses threads (Flask default, gunicorn workers).
Use `asyncio.Lock` + `async def` if your server uses an event loop (FastAPI, aiohttp, uvicorn).

---

## LRU vs LFU

- **LRU without DLL**: heap with lazy deletion works but O(log N) and stale entries accumulate
- **LFU without O(1) constraint**: simpler than LRU — just `min(cache, key=lambda k: freq[k])`, O(N) but trivial
- **LFU at O(1)**: hardest — three structures in sync: `key→(val,freq)`, `freq→DLL of keys`, `min_freq` tracker
