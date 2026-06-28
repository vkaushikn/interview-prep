# Batch Aggregation Daemon (Write-Heavy Aggregation Pattern)

**Appears in:** Spotify, YouTube, Twitter/X (like/view counts), Uber (earnings), ad platforms (billing), games (leaderboards), rate limiting systems — any design where the write shape and read shape are fundamentally different.

## The problem it solves

You have high-throughput raw events (plays, likes, impressions) that need to be written fast, but your read queries want aggregated summaries (total plays per song, earnings per driver). Writing aggregates on every event is too slow; reading raw events at query time is too expensive.

## The pattern

```
Raw events  →  [Staging / MetricsDB]  →  [Daemon]  →  Primary DB (aggregates)
(write fast,                              (periodic,    (read-optimized,
 append-only,                              idempotent)   low cardinality)
 high shard count)
```

Three parts:

**1. Staging store (MetricsDB):** append-only, write-optimized, sharded by a combined hash of the natural key (e.g. user_id + song_id). This spreads writes evenly across shards at all times — sharding by time alone would make the current hour's shards a hotspot. Timestamp is a column within each shard, not a routing key.

**2. Time-bucketed processing:** the daemon fans out to all shards in parallel. Each shard filters its rows for the target bucket (e.g. 2–3 PM), aggregates locally, and dispatches to the primary DB. A coordinator tracks acks from all shards — the bucket is only marked done once every shard has ack'd. Never touch an open bucket: only process once `current_time > bucket_end + safety_margin` (2–5 min to let straggler writes flush).

**3. Idempotency via processed flag:** once all shards have ack'd, mark the bucket `processed = true`. If a shard crashes mid-run and reruns, its write to the primary DB is safe because writes are absolute values ("total plays = 47"), not deltas. The second write is harmless.

## Why not row-level watermarks

An alternative is to track `last_synced_at` per user/song in the primary DB and query MetricsDB for records newer than that. Breaks down at scale: 1B users × fan-out per song = enormous scan to find what needs syncing. Time buckets give you one sequential read of the whole bucket instead.

## The safety margin

Straggler writes arrive out of order across shards. A play event timestamped 2:59:55 may not land on its shard until 3:00:07 due to network lag. Without the margin you'd close the 2 PM bucket before that write arrives and miss it permanently. 2–5 minutes of slack is the standard cushion.

## Formal names for the same idea

- **CQRS + projection** (write side = events, read side = projected aggregates, projector = daemon)
- **Lambda architecture batch layer** (data engineering)
- **Materialized view refresh** (databases)
- **Summary table pattern** (data warehousing)

All the same shape: raw events → periodic aggregation → read-optimized view.

First surfaced in: Spotify case study (MetricsDB → UserDB/SongDB sync).
