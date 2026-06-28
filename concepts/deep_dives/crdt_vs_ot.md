# CRDTs vs. Operational Transformation (Real-Time Collaborative Editing)

**The problem:** multiple people editing the same document concurrently (Google Docs-style). Unlike the Dropbox "two uploads race" problem (resolved by accepting silent last-write-wins, or upgrading to an optimistic-lock version check — see `practise/dropbox.md`), a collaborative editor can't accept "one edit wins, the other vanishes" — every keystroke from every participant has to survive. That requirement is what forces a fundamentally different mechanism than whole-file last-write-wins.

There are two real, independent answers to this. Both guarantee **no data loss** and **every replica converges to the identical result** — neither guarantees the merged result is *semantically sensible*. That second part is not solvable by a smarter algorithm: if two humans blindly type unrelated content into the same spot, some garbling is unavoidable. Low latency (edits streamed live over a persistent connection, typically visible within ~50-200ms) is what makes true blind collisions rare in practice — the algorithms don't prevent them, they just guarantee a clean, lossless outcome *when* they happen.

## CRDT (Conflict-free Replicated Data Type)

No central authority required — any two replicas can merge directly.

**Data model** (the RGA / causal-tree style, as opposed to fractional-index variants like Logoot/LSEQ): each character is a node — `{id, char, is_deleted, prev, next}`. Typing a character creates a new node linked after the node you were just at; deleting sets `is_deleted` (a **tombstone**) rather than removing the node — a tombstone still serves as a valid anchor point for future inserts.

**ID generation** — this is the key mechanism, and it's why a separate ID is needed at all even though `prev`/`next` already define structure: in a single-machine linked list you wouldn't need IDs, pointers would suffice. But each replica holds its own *separate* copy of the structure — A's pointer (a memory address) means nothing on B's machine. The ID is a network-transmissible, globally-stable name for a node, generated **entirely locally, with zero coordination**:

`ID = (site_id, local_counter)` — `site_id` is a random tag assigned once per client session; `local_counter` is just "how many nodes have I personally created so far." The pair is guaranteed unique across the whole system with no network round-trip.

**Tie-break (the one real collision case):** two replicas inserting after the *same* `prev` node concurrently produce two competing "next" candidates. Resolved by comparing `site_id` — same ID used for addressing also serves as a ready-made deterministic ordering key. No data is ever dropped here; the tie-break only decides *order*, e.g. two people both appending after the same point get **both** insertions, concatenated in whichever order the tie-break picks — not one overwriting the other.

**Why an entire offline editing session collapses into one block on merge:** every character after the first one in a continuous typing session anchors to *your own* immediately-preceding character, not back to the original shared anchor. So the only real collision between two independent (e.g. offline) editing sessions is their two *first* characters — everything after that is internally self-consistent. Result: two long offline sessions merge as two large intact blocks placed side by side, not interleaved — which can be a large, nonsensical concatenation if the two sessions diverged for a long time. This is a known, real weakness, not a contrived edge case.

## Operational Transformation (OT)

Requires a single central sequencer per document — this is what Google Docs has historically used.

**Data model:** positions are plain, mutable array indices — `Insert(position=5, "X")`, `Delete(position=5)` — not stable IDs.

**The transform function:** if Client A generates `Insert(5, "System Design ")` against the original text, and Client B concurrently generates `Insert(2, "really ")`, applying A's operation literally after B's has already landed would target the wrong location (B's insertion shifted everything after position 2). The server **transforms** A's operation against B's already-applied edit — adjusting A's position by however much B's edit shifted it — before applying and rebroadcasting it. A transform function exists for every pair of op types (insert/insert, insert/delete, delete/delete); that machinery is the bulk of an OT engine.

**Why this needs one server:** transforming correctly requires an agreed-upon canonical order of "what already happened." Without one authority, two replicas could compute different transforms and diverge. So: client generates an op locally → sends to the single server for this doc → server transforms it against everything since the client last synced → applies, then rebroadcasts the corrected op to everyone else.

**Not the same thing as Git's merge**, despite both reconciling concurrent edits with a central party: Git diffs whole-file **snapshots** line-by-line at discrete commit points, and can produce an unresolved conflict marker for a human. OT operates on a continuous stream of fine-grained ops and the transform function is defined for *every* possible pair of concurrent operations — it always produces an automatic result, never a conflict marker. Same architectural shape (central authority), different algorithm entirely.

## Comparison

| | CRDT | OT |
|---|---|---|
| Central authority | Not required — peer-to-peer merge possible | Required — single sequencer per doc |
| Identifiers | Permanent, globally unique, never reinterpreted | Plain mutable indices, reinterpreted via transform |
| Offline / multi-leader | Natural fit | Awkward — no canonical order without a server |
| Complexity cost | Per-data-type design (tombstones, ID schemes); memory grows with tombstones | Transform function must be correct for every op-type pair |

## site_id vs. user_id — two separate layers

`site_id` is purely mechanical, generated fresh per **editing session** (per open tab/device), with no notion of human identity — your Mac and phone editing the same doc are two separate sessions with two different site_ids, indistinguishable at the CRDT layer from two different people. `user_id` is your authenticated account identity, used for permissions/sharing/display. The two are tied together by a `site_id ↔ user_id` mapping maintained by the **session/auth layer**, sitting above the CRDT engine — established at connection time (you log in, then your session's site_id gets recorded against your user_id). The core conflict-resolution algorithm never needs to know what a "user" is; this also means per-character attribution ("edited by X") falls out almost for free, since the ID already encodes who created each character.

## The honest wall

Both algorithms guarantee no lost data and identical convergence everywhere — neither guarantees a sensible result when two people genuinely edit the same spot with conflicting intent. Real products acknowledge this: past a large divergence (e.g. long offline edits reconciling), some systems (Notion is a public example) stop trusting the automatic character/operation-level merge and fall back to a coarser, human-supervised "pick a version" UI — the same kind of explicit "name the wall, don't silently push past it" move as everywhere else in these case studies.

First surfaced in: a tangent off the Dropbox case study (`practise/dropbox.md`), starting from "we never solved two people uploading at the same time" → generalizing to real-time collaborative editing as the next-level-deep version of the same conflict-resolution problem.

## See also
- [capacity_benchmarks.md](capacity_benchmarks.md) — the same "name the wall, don't drill past it" instinct generalized as a depth-ceiling heuristic for any subtopic in a case study.
