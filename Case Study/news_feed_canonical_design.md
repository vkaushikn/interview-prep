# News Feed: Canonical Design Reference

This is the "textbook" (Grokking/Educative-style) skeleton for this problem, with each step cross-linked to where it lands in our [ACID writeup](news_feed_case_study.md) and the relevant [concepts](../concepts/README.md). The goal isn't to memorize this shape — it's to confirm our derivation *covers* it, and to flag the one place we deliberately diverged.

## The skeleton

| # | Canonical step | Our coverage |
|---|---|---|
| 1 | Requirements clarification (functional / non-functional) | A — Assumptions (scope) |
| 2 | Capacity estimation (DAU, QPS, storage) | A — Approximations; [capacity_benchmarks.md](../concepts/capacity_benchmarks.md) |
| 3 | API design (`postTweet`, `getFeed(cursor)`) | implicit in D5 |
| 4 | Push vs. pull vs. hybrid feed generation | I-1; resolved in D3/D4 |
| 5 | Fanout service (queue + async workers) | D3 — chronological log + checkpoint; [merge_sorted_stream_no_resort.md](../concepts/merge_sorted_stream_no_resort.md) |
| 6 | Celebrity exception (hybrid) | D4 |
| 7 | Feed cache (per-user sorted set of post **IDs**) | D1/D2 — **diverges**, see below |
| 8 | Post store + post cache (content by ID) | D2 — **diverges**, see below |
| 9 | Cursor-based pagination | D5 |
| 10 | Ranking service (pluggable, often deferred) | excluded in A (scope) |
| 11 | Media storage + CDN | excluded in A (text-only); contrast in [cdn_vs_edge_cache.md](../concepts/cdn_vs_edge_cache.md) |
| 12 | Sharding (feed table by user_id, post table by post_id) + multi-region replication | D2 (node counts); [latency_numbers.md](../concepts/latency_numbers.md) (why regional placement) |

## Where we diverge: feed cache contents (steps 7+8)

**Canonical:** the per-user feed cache stores **post IDs** (a reference); actual post content lives in a separate post-cache/store, fetched in a batch ("hydration") step at read time.

**Ours (D2):** each feed entry stores the **full denormalized post content** — one read returns the complete feed, no hydration step.

Both are the same *shape* of trade-off as I-1 (push vs. pull), one layer down — see the new concept [reference_vs_denormalization.md](../concepts/reference_vs_denormalization.md) for the general pattern and the specific cost we accepted (edits/deletes now require a fan-out, same mechanism as D3).

## Things the canonical skeleton has that we explicitly scoped out
- **Ranking (step 10):** our A explicitly assumes pure reverse-chronological. A ranking service would sit *between* D3 (candidate generation) and the served feed — candidate generation (D3/D4) stays the same, only the final ordering step changes. Worth a follow-up case study on its own.
- **Media + CDN (step 11):** excluded by the text-only assumption. If added, this *is* the canonical CDN use case (shared, immutable objects) — contrast with our personalized feed cache, which is not a CDN (see [cdn_vs_edge_cache.md](../concepts/cdn_vs_edge_cache.md)).
- **Notification service:** not part of the feed-read path; would consume the same chronological post log as D3 but fan out to a different sink (push notifications instead of feed caches).
