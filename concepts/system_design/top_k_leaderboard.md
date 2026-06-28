# Top-K Leaderboard (Redis Sorted Sets)

**Appears in:** Spotify (trending songs), YouTube (trending videos), Twitter/X (trending topics), games (leaderboards), ad platforms (top performing ads) — any system that needs to rank items by a frequently updated count.

## The data structure

Redis sorted set: every member has a score, the set is always sorted by score. Two operations:
- `ZINCRBY set_name delta member` — increment member's score by delta. O(log N).
- `ZREVRANGE set_name 0 K-1` — fetch top-K members. O(log N + K).

N is the number of members in the set (a design choice — typically "items active in the last N days," not all items ever). K is a read-time parameter, not specified upfront.

## The write throughput problem

At 500k play events/sec and Redis at 100k ops/sec per node, you cannot write +1 per event directly to the sorted set.

**Solution:** workers consume from a queue, aggregate locally over a batch of events, write one `ZINCRBY song_abc 4523` per unique song per batch. This collapses 500k events/sec into thousands of ZINCRBY calls/sec — well within one Redis node.

```
Play events (500k/sec)
    ↓
  Queue (Kafka)
    ↓       ↓       ↓
Worker 1  Worker 2  Worker 3   ← aggregate locally per batch
    ↓       ↓       ↓
       Redis Sorted Set        ← thousands of writes/sec
```

Multiple workers writing to the same sorted set is safe — ZINCRBY is atomic (Redis is single-threaded).

## Which cuts to precompute

The sorted set only works for precomputed cuts. Product/analytics decides which dimensions to maintain (global, by genre, by artist, by region). Each cut is a separate sorted set; workers update all relevant sets per batch.

Every additional dimension multiplies write load on workers — this is the engineering constraint you feed back to product.

Esoteric queries (not precomputed) go to the aggregate DB via scatter-gather (see below).

## Time windows — tumbling buckets

A sorted set accumulates scores forever. To answer "top songs in the last 24 hours," maintain one sorted set per time bucket:

```
top_songs_hour_14  (closed)
top_songs_hour_15  (closed)
top_songs_hour_16  (currently filling)
```

To query a multi-bucket window: `ZUNIONSTORE` across the relevant buckets, summing scores. A song's total is the sum across all buckets — you cannot take top-K per bucket independently and merge, because per-bucket winners may not be global winners.

`ZUNIONSTORE` is expensive (O(N × buckets)). Run it on a schedule (e.g. every minute), store result in a precomputed `top_songs_last_24h` set, serve reads from there.

**Sliding window precision vs. cost:** smaller buckets = more precise window, more sorted sets, more writes. 1-minute buckets for "trending now" is the practical sweet spot. 1-second buckets rebuilds a raw event log — defeats the purpose.

## Non-precomputed queries — scatter-gather

For arbitrary cuts not in Redis, query the aggregate DB (e.g. SongMetricsAggDB). If the table is sharded:

1. Fan out query to all shards in parallel
2. Each shard: local GROUP BY + SUM + top-K
3. Coordinator: re-aggregate across shard results (SUM by song_id), sort, return global top-K

Each shard returns only K rows, not everything — a song outside any shard's local top-K cannot be the global winner.

## Per-friend top-K (fan-in at read time)

"Top songs among my 100 friends" cannot be precomputed per user — the write amplification is too high (every play fans out to all friends' sorted sets). Instead, compute at read time:

- Each user maintains a small recent activity cache (last 24h songs played)
- At query time: fetch 100 friends' activity lists in parallel (100 Redis GETs)
- Aggregate in memory: count by song_id across all lists (100 friends × 50 songs = 5k items — trivial)
- Sort, return top-K

Works because F=100 makes read-time aggregation cheap. See [[fan_out_vs_fan_in]] for when this flips.

First surfaced in: Spotify case study.
