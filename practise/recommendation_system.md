# Recommendation System & Newsfeed Architecture
# (Derived from a Socratic design session — use as context for follow-up work)

---

## Overall Architecture Summary

Two parallel problems:
1. Fan-out on write — posts from people you follow
2. Candidate generation + ranking — "you may also like" content

These merge at serve time via a lightweight CPU re-scorer.

---

## Write Side

### Follow-Graph Fanout
- Post created → Kafka topic
- Fanout worker reads follower list → writes post to per-user timeline cache (sharded by user_id)
- Also writes to persistent storage (Cassandra or similar)
- Celebrity problem: accounts with 50M+ followers don't fan out on write — pulled at read time instead (hybrid)

### Video/Post Engagement Metrics
- Every view, like, share, reaction → Kafka topic (separate from post fanout)
- Multiple consumers reading from same Kafka topic independently (each tracks its own offset):
  - Consumer 1: Raw log writer → S3/HDFS (ground truth for offline ML training)
  - Consumer 2: Flink aggregator (15-min tumbling window) → item feature store (engagement stats)
  - Consumer 3: Top-K dashboard (real-time analytics)
  - Consumer 4: Abuse/anomaly detection
  - Consumer 5: Creator analytics (delayed batch)
- Kafka retention = safety net for consumer crashes / recovery within retention window
- Consumers use at-least-once semantics; downstream handles dedup

### Item Feature Store
- Keyed by item_id
- Contains: embedding (precomputed offline, updated slowly), engagement_rate_15m, like_rate, share_velocity, quality_score, topic_vector
- Core embeddings populated from offline training pipeline
- Engagement stats updated every ~15 min via Flink → feature store write
- Virality is captured via engagement features, NOT embeddings — embedding doesn't change when video goes viral, but ranker sees spiking engagement stats and boosts it
- Trending pool: separate cron job scans for items crossing engagement velocity threshold → adds to global trending candidate pool (how viral content escapes personalization)

---

## Candidate Generation ("You May Also Like")

### Two-Tower Model
- Two separate neural encoders trained jointly with shared contrastive loss:
  - User Tower: [user_id, history, demographics, ...] → 128-dim embedding
  - Item Tower: [item_id, content features, stats, ...] → 128-dim embedding
- Loss: dot(user_emb, positive_item_emb) should be HIGH, dot(user_emb, negative_item_emb) should be LOW
- Both towers co-trained → output spaces are geometrically compatible
- Key insight: item embeddings precomputed offline; user embedding computed once per request → ANN retrieval is O(log N) not O(N)
- Cannot swap in independently pretrained item embeddings — they live in a different vector space

### ANN Retrieval (HNSW)
- Item embeddings indexed offline in HNSW (Hierarchical Navigable Small World) structure
- Multi-layer graph: sparse at top, dense at bottom
- Query: start at top layer, traverse graph, drop layers, return top-K at layer 0
- Complexity: O(log N) vs O(N) for brute force
- Index lives in RAM on dedicated ANN servers (memory-heavy — ~500GB+ for 1B items at 128-dim float32)
- Sharded across machines; query fans out to all shards in parallel, results merged
- Cosine similarity = dot product on L2-normalized vectors (normalize at index build time)
- Fresh index: new items (<1hr old) go into small brute-force-searched index rebuilt every ~5min; merged with main index at query time

### Candidate Sources Merged
1. ANN retrieval on user long-term embedding (~2000 items)
2. Follow-graph timeline cache (posts from people you follow)
3. Trending pools (globally/regionally viral content)
4. Session-vector ANN query (if session has drifted significantly)

---

## Prefetch Pipeline (Async Daemon)

Runs ahead of user need — triggered at session start and periodically during session.

```
Read: user long-term embedding (user feature store, stale ~1hr OK)
Read: session context vector (in-memory, fresh)
ANN query on item embedding index → 2000 candidates
Merge with: follow-graph cache + trending pools
Lightweight ranker (CPU) → top 200
Deep ranker (GPU, batched) → top 50
Write → prefetch cache (keyed by user_id, with TTL)
```

GPU batching: prefetch scheduler collects N users due for refresh → single GPU job scores N×100 candidates together → write back per user. This is the efficiency win vs real-time GPU serving.

Prefetch cache TTL is a tunable knob:
- Short TTL (5 min): fresher, more GPU cycles
- Long TTL (30 min): staler, cheaper
- Tuned per user segment (active users get shorter TTL)

---

## User Representation

### Long-Term Embedding
- Updated hourly via batch job or streaming (Kafka → Flink → user feature store)
- Captures stable interests (sports, tech, cooking)
- Slow to change intentionally — one coding session shouldn't erase sports fan identity
- Lives in user feature store, read by prefetch daemon

### Short-Term Session Context
- NOT stored in DB — lives in serving layer session memory
- Updated on every engagement event in real time
- Computed as exponentially decayed weighted average of item embeddings watched this session:
  `session_vec = sum(w_i * item_emb_i)  # w_i decays with time/position`
- Used as:
  1. Additional ANN query vector in next prefetch (finds content matching current intent)
  2. Re-scoring signal at serve time

### Session Drift Detection
- If cosine distance between session_vec and original prefetch query vector exceeds threshold → trigger full prefetch refresh with new session_vec
- Brings in fresh candidates matching current intent (e.g., coding videos) within minutes

---

## Serve Time (CPU Only, ~60-90ms total)

```
Read prefetch cache → 50 ranked items (precomputed)
Fetch fresh item features for those 50 (feature store bulk read)
Inject fresh session context vector
Lightweight re-scorer on CPU:
    final_score(item) = alpha * prefetch_score
                      + beta  * dot(session_vec, item_emb)
                      + freshness_signal
                      + diversity_penalty
Return to client
```

Alpha/beta weights: early in session alpha > beta (session signal noisy); deep in session beta increases (strong session signal).

This is how "I'm a sports fan watching coding videos right now" gets handled — deep model said sports (ran 10 min ago), but dot(session_vec, item_emb) demotes sports and promotes coding at serve time without any DB write or GPU call.

---

## Feature Store Architecture

Two categories of features, both precomputed:

### Item Features (keyed by item_id)
- embedding: 128-dim float (from item tower, updated when model retrains)
- engagement_rate_7d, engagement_rate_15m
- avg_watch_pct
- topic_vector
- quality_score
- Updated at different cadences by different workers

### User Features (keyed by user_id)
- embedding: 128-dim float (updated hourly)
- interest_topics
- avg_session_length
- recent_history_summary

At serve time: one bulk read from feature store for all 50 candidate item IDs + user → model runs purely in-memory, no further I/O.

---

## Three Update Cadences

| Cadence | What | How |
|---|---|---|
| Slow (days/weeks) | Embedding model retraining | Reads from S3 raw logs |
| Medium (minutes) | Item feature aggregations | Kafka → Flink → feature store |
| Fast (seconds) | Raw engagement counters | Written to Cassandra on every event |

Virality is captured at the medium cadence. A video going viral shows up in ranker within ~15 minutes via engagement stats, without waiting for embedding model retraining.

---

## Key Design Tensions

1. **Precompute vs real-time:** Deep GPU model runs in prefetch (efficiency, controlled batching). Lightweight CPU re-scorer runs at serve time (freshness for session context).

2. **Feature freshness vs staleness tolerance:**
   - Item engagement stats: 15min stale = acceptable
   - User long-term embedding: 1hr stale = acceptable
   - User short-term session context: must be fresh = injected at serve time
   - Breaking news / viral: 15min stale = acceptable (trending pool catches it)

3. **Write fanout vs read cost:** Fan out follow-graph on write. Pull recommendations on read (from precomputed indexes). Celebrities pulled at read time to avoid write amplification.

4. **One-tower vs two-tower:** One model concatenating user+item features is more accurate but requires O(N) inference at serving time. Two-tower enables ANN → O(log N). Architecture is an engineering constraint, not a modeling preference.

---

## Connections to existing concepts
- Fan-out/fan-in: [[fan_out_vs_fan_in]] — celebrity hybrid is the same pattern as news feed
- Batch aggregation daemon: [[batch_aggregation_daemon]] — feature store updates at medium cadence
- Top-K leaderboard: [[top_k_leaderboard]] — trending pool is top-K by engagement velocity
- Kafka multi-consumer: same durable log enabling independent consumers at different cadences

## Open Topics (Parked)
- HNSW internals and vector database implementations (Faiss, ScaNN, Weaviate)
- Exact negative sampling strategy for two-tower training
- Multi-task ranking (optimize for watch time vs likes vs shares simultaneously)
- Cold start for new users and new items
