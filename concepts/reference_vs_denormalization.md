# Reference vs. Denormalization in a Precomputed Cache

**The problem:** once you've decided to precompute something per-user (e.g., a feed), each entry can either:
- **Store a reference** (e.g., post ID) and fetch the real content from a separate store at read time, or
- **Store the full denormalized content** inline, so a single read returns everything.

## The trade-off
This is the *same shape* of decision as fan-out-on-write vs. fan-out-on-read ([capacity_benchmarks.md](capacity_benchmarks.md) / News Feed I-1) — push the cost to write-time (denormalize: copy content into every feed it appears in) or to read-time (reference: one extra lookup per item, but content lives in exactly one place).

| | Reference (store ID) | Denormalized (store content) |
|---|---|---|
| Storage | O(1) copies of each post | O(followers) copies of each post |
| Read cost | N+1 lookups (feed IDs, then batch-fetch content) | 1 lookup |
| Edits/deletes to a post | Propagate automatically — only one copy exists | Must fan out the update/delete to every copy |
| Best when | content mutates often, or storage dominates | reads are by far the dominant cost and content is near-immutable once posted |

## Applied to the News Feed
The canonical textbook design stores **post IDs** in the per-user feed cache (a Redis sorted set) and fetches content from a separate post cache — minimizing storage and making edits/deletes trivial.

Our design (`Case Study/news_feed_case_study.md`, D2) chose to **denormalize full content** into each feed entry, on the grounds that posts are read far more than edited/deleted, and the 2-4TB total footprint is still "Doable" per [capacity_benchmarks.md](capacity_benchmarks.md). The cost we accepted: a post edit or delete must be fanned out to every copy — the same O(followers) fan-out problem as D3, just triggered by a different event.

**General lesson:** any time you "precompute and cache," ask whether you're caching a reference or the value itself — and check whether the *update/invalidation* path you're implicitly creating is one you've already solved (here: yes, it's the same fan-out mechanism as new-post delivery) or a new one.

First surfaced in: News Feed case study, comparing our D2 against the canonical design (see `Case Study/news_feed_canonical_design.md`).
