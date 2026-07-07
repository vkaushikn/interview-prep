# Raft-Style Leader Election

**Why a leader at all:** if any node could accept writes for a given shard/resource, two nodes could accept conflicting writes with no way to reconcile them. One node is designated leader; only it accepts writes, and replicates them to followers.

## Normal operation
Leader sends periodic heartbeats to followers. Each follower has an **election timeout** that resets on every heartbeat. As long as heartbeats arrive on schedule, nobody questions who's in charge.

## The one randomized element — and why it has to be random
Each follower picks its election timeout **independently and randomly** from a range (e.g. 150-300ms), re-randomized every time it resets. This is the *only* stochastic part of the protocol — vote-granting and term increments are deterministic.

**Why it must be random:** if every follower used the same fixed timeout, a leader failure would make all of them time out simultaneously, all become candidates in the same instant, all vote for themselves, and split the vote — with the next round colliding identically, potentially forever. Randomized timeouts mean one follower's timer fires meaningfully before the others, so it gets an uncontested shot at collecting votes before any rival even starts.

**Worked example (5 nodes: A leader, B/C/D/E followers):** A fails. C's randomly-chosen timer (shortest) fires first, becomes a candidate, increments its term, votes for itself, requests votes from B/D/E before their own (longer) timers expire. They grant votes — C wins a clean majority (4/5), becomes leader. No collision.

**The failure case it's preventing — split vote:** if two followers' timers happen to fire close enough together that neither gets an uncontested shot (e.g. C and D both become candidates for the same term before either's request propagates), votes fragment — say C gets {C, B}, D gets {D, E}, neither reaches a majority of 5. No leader elected. Both candidates' timers simply restart with a **fresh** random draw; each retry is an independent draw, so the odds of colliding again shrink each round, and the system converges probabilistically rather than on a guaranteed first try.

**Precision on what "wins" means:** the earliest timer doesn't win *by definition* — it wins because it gets to request votes before any competitor exists, and with no competition, collecting a majority is close to automatic. The timer decides who gets the first uncontested attempt, not the outcome directly. (Raft also withholds a vote from a candidate whose log is missing entries the voter already has — a safety check layered on top of the timing race, not a replacement for it.)

## The dangerous failure case: partition, not crash
If the leader is merely cut off from the majority (not dead), the majority side times out and elects a new leader with a **higher term number** — but the old leader, still reachable by whatever's on its side of the partition, may keep accepting writes, unaware an election happened. This is split-brain: two nodes simultaneously believing they're in charge.

**Fencing fixes it:** every write is tagged with the term of the leader that issued it. When the partition heals, the old leader's writes carry a lower term than the current one and get rejected outright; the old leader steps down the instant it sees a higher term from anyone. Only writes from the highest-term leader ever survive across the majority.

## The CAP trade-off, concretely
While it's ambiguous whether the leader is dead or just slow, you choose: (a) block writes until certain who's in charge (consistency over availability), or (b) keep serving through whoever's reachable, accepting some writes may later be fenced/discarded (availability over consistency). No version of this avoids the trade-off — only where the cost gets absorbed.

## Reads from followers can be stale — and that's often fine, by reusing a pattern you already know
A follower may not have applied the very latest committed entries yet (replication lag). Serving a read from a follower risks staleness — but if the *real* decision (a write/claim) still has to go through the leader's authoritative state before it's accepted, a stale read only costs a wasted round-trip (attempt, get told "already taken, here's who"), not an actual correctness violation. This is the same shape as **optimistic locking**: a possibly-stale check, followed by an authoritative validation at commit time. Whether this is safe to allow depends entirely on whether the eventual write path re-validates — if it doesn't, stale reads become a real correctness bug, not just an inefficiency.

## Applied uses in this project
- **Distributed cache shard leader failure** (`Case Study/distributed_cache_case_study.md`, Failure Modes) — the original surfacing.
- **Collaborative-editor document-ownership registry** — "claim this document's live session" is the identical problem shape as "elect a shard leader," just applied to a document instead of a data shard. Real implementations of this exact pattern: ZooKeeper, etcd (both Raft/Paxos-based coordination services) — distinct in kind from a distributed *cache*, because the registry needs strong agreement on "who won," not staleness-tolerant throughput. Can still be sharded by resource ID (e.g. doc_id) for parallelism, with each shard internally strongly-consistent — different from cache-style sharding, which shards for capacity and tolerates inconsistency across replicas.

## See also
- [crdt_vs_ot.md](crdt_vs_ot.md) — OT's single-sequencer-per-document requirement is the same "exactly one authority must decide order" need that leader election satisfies for a data shard.
