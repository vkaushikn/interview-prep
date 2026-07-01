# Coding Backlog

Problems to do after the core two-week plan.

## LRU Cache — LC 146 (Medium)

Two implementations to know:
1. `OrderedDict` + `move_to_end` — quick, clean, but requires knowing the API
2. Doubly linked list + hashmap — what Staff interviews want. Sentinel head/tail to avoid edge cases. O(1) get, put, evict.

This is an IYKYK problem. Not reasonably derivable cold. Memorize the DLL approach.
