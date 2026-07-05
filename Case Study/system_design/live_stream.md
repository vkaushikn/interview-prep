# Live Stream (Social Network)

## Pattern

**Fan-out media pipeline** — one producer (streamer) broadcasting to many consumers (viewers), with a real-time messaging overlay (comments).

```
Producer → Ingest → Queue → CDN origin → CDN edge → Viewers
                                       ↘ Comment outbound → Viewers (WebSocket)
```

Appears in: YouTube Live, Twitch, Instagram Live, Discord stage channels.

Each layer has well-understood optimization space (codec, CDN placement, queue tuning, comment fan-out). This design focuses on overall topology and interface contracts, not per-component tuning.

---

## A — Assumptions

1. Discovery out of scope — handled by hybrid fan-out-on-write + fetch-at-read-time (standard news feed pattern)
2. Two separable sub-systems: **video delivery** and **live comments** — focus on influencer tier first
3. Single quality tier — slower clients buffer more (no adaptive bitrate in scope)
4. Seek-back / VOD out of scope for the live path; chunks persist for post-event processing
5. Comment ordering: arrival order (system timestamp), no client-side sort; nested replies supported
6. Two load tiers: **influencer** (~100k concurrent viewers), **mass event** (Soccer World Cup, ~500M; comments may be disabled by policy)

---

## C — Constraints

| Constraint | Active? | Notes |
|------------|---------|-------|
| Video feels live — low latency, no choppiness | Yes | HLS chunk size (2–10s) is the main latency floor |
| Comment appears near-instantly | Yes | WebSocket push, not poll |
| Exact comment ordering | No | Approximate timestamp ordering acceptable; reconcile post-event if needed |
| Viewer count accuracy | No | HyperLogLog approximation acceptable |

---

## Approximations

- 1B DAU, 5% stream on any day → **50M live streams/day** → ~500/sec sustained, **~2000/sec peak**
- 80% of streams are influencer tier with ~100k concurrent viewers
- 20% are small streams (~10 viewers) — negligible load
- **Concurrent viewers at peak**: 2000 streams × 100k viewers = **200M concurrent viewers**
- **Comment throughput**: 200M viewers × 1% commenting = 2M comments/sec total → **1000 comments/sec per stream**
  - One Kafka partition handles ~100k writes/sec → 1000 comments/sec = 1% utilization → **no sharding needed per stream**

---

## I — Incentives / Trade-offs

- **HLS over WebRTC** — HLS/DASH scales to 100k+ viewers via CDN (standard HTTP file serving). WebRTC is sub-second latency but point-to-point; does not scale to mass audiences without a media server fleet. Trade-off: HLS adds 2–10s delay.
- **CDN for video, WebSocket for comments** — different latency profiles. Video tolerates 2–10s buffering; comments must feel instant. Two separate delivery mechanisms.
- **Inbound/outbound service split for comments** — decouples write pressure from fan-out pressure. Inbound service writes to queue; outbound service drains queue and fans out per stream. Scales independently.
- **Policy cache (hot)** — AI content moderation reads the video queue async. Signals stored in `stream_id → policy_status` cache. Gate at CDN origin, not at ingest (avoids blocking the hot write path).
- **Mass event policy lever** — at Soccer World Cup scale (~500M viewers), comments may be disabled by ops policy. System design accommodates this; the comment path is optional per stream.

---

## D — Design

### Video Delivery

```
Streamer device
    → Ingest service (2000 instances, one per active stream)
        → chunks video into 2–10s HLS segments
        → writes segments to blob storage (S3 / GCS)
        → updates HLS manifest (m3u8) per stream
    → CDN origin servers pull from blob storage
    → CDN edge nodes cache latest segment globally
    → Viewers pull latest segment from nearest CDN edge (standard HTTP)
```

**Why blob storage as the handoff point**: decouples ingest rate from CDN pull rate. Ingest writes once; CDN edge replicates globally. No direct ingest-to-viewer connection.

**AI policy service**: reads segments async from blob storage, runs moderation model. Updates `stream_id → {ACTIVE | SUSPENDED}` in a hot Redis cache. CDN origin checks cache before serving; suspended streams return 403.

**Post-event**: once stream ends, segments in blob storage are handed to the standard video processing pipeline (transcoding, multi-bitrate, thumbnail extraction).

**Failure modes**:
- Ingest service crashes → client retries with exponential backoff; new ingest instance picks up; last written segment is already in blob storage
- CDN edge stale → TTL on HLS manifest is short (2–5s); edge re-fetches manifest on expiry
- Blob storage unavailable → ingest buffers locally (bounded); stream degrades to buffering on viewer side; no data loss

---

### Live Comments

```
Viewer (commenter)
    → WebSocket connection to Comment Inbound service
        → writes (stream_id, user_id, message_id, parent_message_id, body, ts) to Kafka
            → partitioned by stream_id (one partition per stream, ~1000 writes/sec = 1% utilization)
    → Comment Outbound service
        → consumes from Kafka partition for this stream_id
        → fans out to all WebSocket connections watching this stream
        → writes flush to persistent storage (Cassandra / Bigtable) async
```

**Connection registry**: `stream_id → [outbound_server_ids]`, `outbound_server_id → [connection_ids]`. Used by outbound service to know which servers hold connections for a given stream.

**Inbound / outbound split**: inbound service only writes to Kafka (write path). Outbound service only fans out (read path). Scale independently — high comment volume streams get more outbound instances.

**Ordering**: Kafka partition preserves write order within a stream. Outbound fans out in Kafka order. Good enough for live feel; exact wall-clock ordering not guaranteed across partitions (not needed per constraints).

**Viewer counter**: outbound service maintains an in-memory counter per stream. Gossip protocol aggregates across outbound servers every few seconds. Approximate count (HyperLogLog). Displayed as "~Xk watching."

**Failure modes**:
- Outbound server crashes → viewer WebSocket drops; client reconnects to new outbound server; new server re-registers in connection registry; resumes consuming from last committed Kafka offset
- Kafka lag spikes → comments appear slightly delayed; still causally consistent (Kafka order preserved); acceptable per constraints
- Mass event (500M viewers) → ops disables comment inbound service for that stream_id; outbound continues serving viewer count only

---

## Key Numbers

| Metric | Value |
|--------|-------|
| Peak concurrent live streams | ~2000 |
| Viewers per influencer stream | ~100k |
| Peak concurrent viewers | ~200M |
| Comments/sec per stream | ~1000 (1% of viewers) |
| HLS segment size | 2–10s |
| CDN edge pull frequency | every segment interval |
| Kafka partition capacity | ~100k writes/sec |
| Comment Kafka utilization per stream | ~1% |
