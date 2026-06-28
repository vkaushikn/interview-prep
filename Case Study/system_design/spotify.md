# Spotify: Canonical Design Reference

## A — Assumptions & Approximations

### Assumptions (scope)
- User can create playlists, add/remove songs (exact search by Artist/Album/Song name — no fuzzy/arbitrary search)
- User can listen to a playlist start-to-finish (no shuffle, no ads)
- User can like/dislike songs
- Show top-K songs globally, by genre, by artist
- No social features (friends feed, shared playlists) — scoped out
- No offline mode / DRM

### Approximations
| Quantity | Estimate |
|---|---|
| Total users | 1B |
| DAU | 500M |
| Songs in catalog | 2B |
| Song file size | ~10MB |
| Song metadata row size | ~300 bytes (IDs + URL) |
| Playlists per user | 20 |
| Songs per playlist | 50 |
| Songs played per user per day | 15 (~1 hr listening) |

**Derived:**
- Blob storage: 2B × 10MB = **20 PB**
- Song metadata DB: 2B × 300B ≈ **600 GB ~ 1 TB** (fits in cache)
- UserDB rows: 1B × 20 × 50 = **1e12 rows** × ~100B ≈ **100 TB**
- Play throughput: 500M × 15 / 1e5 ≈ **75k plays/sec avg, ~500k peak**
- Per-node bounds: disk-backed DB ~1k writes/sec, ~10k reads/sec; Redis ~100k ops/sec (see [capacity_benchmarks](../../../concepts/approximations/capacity_benchmarks.md))

**Takeaway:** storage is not the bottleneck. Write throughput to MetricsDB (500k/sec peak) and read throughput for playlist serving are the binding constraints.

---

## C — Constraints

- **Latency:** playlist loads and song starts < 200ms (human perception threshold)
- **Consistency:** playlist edits must be immediately visible (strong); play/like counts can lag minutes (eventual)
- **Availability:** serve playlists from cache even if primary DB is degraded
- **Throughput:** 500k peak play events/sec must be durably recorded (artist royalty payments depend on accurate counts)

---

## I — Incentives & Trade-offs

1. **Separate write-heavy metrics from read-heavy metadata.** Recording play/like events at 500k/sec against the same DB that serves playlist reads would require a single ~500-node cluster. Splitting into MetricsDB (write-optimized) and UserDB/SongDB (read-optimized) lets each scale independently.

2. **Batch aggregation over synchronous writes.** Writing aggregate stats (total plays, total likes) synchronously on every event is expensive and unnecessary — stats can lag by minutes. A daemon processing closed time buckets is cheaper and correct (see [batch_aggregation_daemon](../../../concepts/system_design/batch_aggregation_daemon.md)).

3. **Cache SongDB, don't shard it.** Song metadata is ~1 TB and static. A cache covering the top ~1% of songs (by Zipf distribution) handles ~90% of reads. Remaining reads hit a small SongDB cluster (~5 nodes). No sharding needed.

4. **Precomputed sorted sets for top-K, scatter-gather for esoteric queries.** Redis sorted sets serve common top-K cuts in O(log N). Arbitrary cuts (non-precomputed) fall back to the aggregate DB via scatter-gather.

5. **Fan-in at read time for friends top-K** (if added). Play events are too frequent (15/day × 100 friends = 1,500 write fan-outs/day per user) to maintain personalized sorted sets. Fetching 100 friends' activity lists at query time and aggregating in memory is cheaper (see [fan_out_vs_fan_in](../../../concepts/system_design/fan_out_vs_fan_in.md)).

---

## D — Decisions

### D1. Data model

| Store | Schema | Notes |
|---|---|---|
| **SongDB** | song_id, title, artist_id, album_id, genre_id, song_url | Static metadata, ~1 TB, served from cache |
| **UserDB** | user_id, playlist_id, song_id → is_liked, is_disliked | Like is a boolean per user per song, not a count |
| **MetricsDB** | user_id, song_id, timestamp → is_played, is_liked, is_disliked | Raw events, write-optimized, sharded by hash(user_id, song_id) |
| **SongStatsDB** | song_id → total_plays, total_likes (per time bucket) | Populated by daemon from MetricsDB |
| **Blob storage** | song files (20 PB) | S3-equivalent behind CDN |

Aggregate stats (total plays, likes) live in SongStatsDB, not SongDB — keeps static metadata separate from write-heavy aggregates.

### D2. Audio delivery

Progressive download: song files (~10MB) are small enough to download entirely at play time. Client starts playing after a 1-2s initial buffer; rest downloads in background.

- Files served via CDN (pre-signed URLs, 15-min TTL)
- Top 1% of songs proactively pushed to CDN PoPs; long-tail lazy-pulled on first regional request
- Playback resume position checkpointed server-side on pause/stop (enables cross-device resume)

See [media_delivery](../../../concepts/deep_dives/media_delivery.md).

### D3. Metrics pipeline (batch aggregation daemon)

At end of each song play, client writes event to MetricsDB (sharded by hash(user_id, song_id), ~50 shards at 1k writes/sec/node).

Daemon runs on closed hourly time buckets:
1. Fans out to all shards in parallel; each shard aggregates its rows for the closed bucket
2. Dispatches absolute-value updates to SongStatsDB and UserDB
3. Coordinator marks bucket `processed = true` once all shards ack
4. Safety margin: only process buckets where `current_time > bucket_end + 5 min` (straggler writes)

Idempotent by design — writes are absolute values, so crash-and-rerun is safe.

See [batch_aggregation_daemon](../../../concepts/system_design/batch_aggregation_daemon.md).

### D4. Top-K leaderboard

Workers consume play events from a queue, aggregate locally per batch, write `ZINCRBY` to Redis sorted sets (well under 100k ops/sec/node after batching).

One sorted set per time bucket per cut (e.g. `top_songs_genre_pop_hour_14`). Multi-hour windows use `ZUNIONSTORE` across buckets, run on a schedule, result stored in a precomputed set for reads.

Which cuts to maintain: product decision based on query log analysis. Each additional cut multiplies worker write load.

Non-precomputed cuts: scatter-gather across SongStatsDB shards (local top-K per shard → coordinator re-aggregates → global top-K).

See [top_k_leaderboard](../../../concepts/system_design/top_k_leaderboard.md).

### D5. Search

Exact-match search by song title, artist, album. Inverted index (title/artist/album → list of song_ids). Small enough (~1 TB) to serve from a search cluster without sharding. Out of scope for deeper design here.

### D6. Playlist serving

Playlists fetched from a regional cache (user_id → playlist metadata + song list). Cache populated on login, invalidated on edit. UserDB is the source of truth; edits write through to both DB and cache for strong consistency on mutations.

---

## Failure Modes

- **MetricsDB shard failure:** unprocessed events for that shard are held in the queue; daemon retries once shard recovers. Play counts lag but are not lost.
- **Daemon crash mid-bucket:** restarts from last unprocessed bucket; idempotent writes make rerun safe.
- **CDN miss on song file:** request falls through to blob storage origin; latency degrades but playback is not blocked.
- **Redis sorted set node failure:** top-K reads degrade to scatter-gather on SongStatsDB until Redis recovers. Acceptable — top-K is not on the critical playback path.
