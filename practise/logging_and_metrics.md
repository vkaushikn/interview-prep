# Logging and Metrics 
(FB Scuba system -- don't know if the internals are known)

# Problem
We have millions of servers, and on each server we have many VMs. Each VM is running a service that emits lots of metrics (arbitrary timing). We want to support a DB table for exact queries on those metrics, while also supporting a blazing fast approximate query on the metrics,.

(Assumptions are just coming from the question - precisely phrased)
# Approximations
1e6 server  * 4 VM / server * 1e2 metrics / sec /VM = 4e8 metrics / sec  -- Average DB is supporting 1k write/sec (even optimizing for writes - this is just mind boggling)

# Constraints
1. Highly Avaialble (the approximate query is used to debug high prioirty Sevs)
2. Eventually we want the exact data

# Design
1. Service registers metric on to a queue on the Server (service has some namespace like thing to identify topic)
2. Deaemon groups registered metric together (i think they do this to save space -- network is also a constraint at this scale)
3. Deamnon pushes metric to Queue (different topics on the queue)
4. Worker removes the metrics from the queue
   4.1 "Fast" worker - arranges the data in columnar format (Service also needs to provide some schema) and sends to a "Leaf"
   4.2 "Slow" path worker - batches the data and writes to DB (for correct query)
   4.3 "Glacier" path worker - creates even bigger batches of data and writes it Blob storage
5. User queries - the query is scattered onto all the Leafs -- the leafs return their portion of the matching result and the query server aggregates the fast metric
6. Leaf nodes have SSD - where data is written -- "extends" the time horizon for which we can query "Fast" mode (typically 3 days). Some queries are slower because we need to get data from SSD. Also, persistent means recovery is faster if Leaf dies.

# Glacier Path — Log Storage and Search

The glacier path serves double duty: long-term metric retention AND log storage.

**Log schema on fast path**: Logs also go through the columnar leaf nodes but with a fixed schema: `(namespace, service_name, timestamp, message)`. This gives fast approximate search on recent logs.

**Blob storage**: Full log text is written to blob storage in batches. A separate metadata table tracks which blob contains logs for a given `(namespace, service, hour)` — so you can answer "where are the logs for service X between 2pm and 4pm?" without scanning all blobs.

**LogGrep service**: Given `(namespace, service, hour_range)`:
1. Look up metadata table → get blob locations
2. Fetch blobs
3. Stream to browser where user does grep-like search on the raw text

This is pull-on-demand — logs are not indexed by default, just fetched and searched client-side (or server-side with streaming).

**Reverse index**: An index is built on top of blob storage for faster lookup — likely an inverted index on log message tokens, similar to how search engines work. Allows "find all logs containing ERROR in service X" without fetching all blobs for that service.

# LogGrep — Session Model and Protocol

**Debugging workflow**: A user doesn't just grep once — they grep the same logs multiple times as they follow the trail. Re-fetching the blob on every grep is wasteful.

**Session caching**: On first query, LogGrep fetches the blobs and caches them on the server. Subsequent greps on the same `(namespace, service, hour_range)` grep against the cache — no re-fetch.

**Session affinity**: Subsequent requests must hit the same LogGrep server that holds the cached blob. Two options:
- Sticky sessions via load balancer (routes by session token)
- WebSocket — persistent connection IS the session, naturally routes to the same server

**Why WebSocket over SSE here**: WebSocket is the right fit because:
1. Client sends multiple grep patterns over the lifetime of the session (bidirectional)
2. Server streams results back for each grep
3. Persistent connection keeps the blob cache alive
4. When user closes the terminal view, connection drops and cache is evicted

SSE would work for a single grep (one-directional stream), but WebSocket fits the interactive multi-grep debugging session better.

# Generalizing the Pattern — Tiered Freshness

The core pattern: **fast approximate answer now, exact answer later.** Shows up far beyond metrics:

| Use Case | Fast Path | Slow/Exact Path |
|----------|-----------|-----------------|
| Leaderboard ± N around my rank | Approx rank from scatter-aggregate | Exact rank for rewards cutoff |
| Trending topics | Approx count in last 1hr | Exact count for reporting |
| Rate limiting | Approx request count | Exact audit log |
| Ad impressions | Approx count for pacing | Exact count for billing |
| Capacity utilization | Approx for scheduling | Exact for cost allocation |
| Stock ticker | Approx last price | Exact trade record |

**Rule**: any system where "good enough now" is more valuable than "exact later" fits this architecture.

## Leaderboard ± N Around My Rank

"Show me users ranked within ±50 of my rank" — range query on a leaderboard, not top-K.

**Redis sorted set** — works if leaderboard fits in memory. `ZRANK` gives rank, `ZRANGE rank-50 rank+50` gives neighbors. Breaks at 100M+ users.

**Scatter-aggregate (this design)** — leaderboard sharded across leaf nodes by score bucket:
1. Scatter: "how many users have score > my_score?" to all leaves
2. Each leaf counts locally (fast — columnar, scan one column)
3. Aggregate counts → approximate global rank
4. Fetch users at rank ± 50 from the relevant shard(s)

Approximate because leaves have slightly stale data (seconds/minutes lag). Fine for display. Exact rank for rewards cutoffs comes from slow path.

**Key insight**: rank is never stored — computed on demand from score counts. Storing rank explicitly would require updating everyone's rank on every score change, impossible at scale.

## What Queries Does the Fast Path Support?

**Easy** — decompose cleanly across shards (local compute + combine):
- COUNT, SUM, AVG, MIN, MAX
- Simple filters: `WHERE service = 'X' AND ts > now-5min`
- GROUP BY on low-cardinality columns (service, region)
- Approximate rank, Top-K

**Hard or impossible** — require global view of data:
- Window functions (`RANK() OVER`, `LAG`, `LEAD`) — need global ordering
- JOINs — cross-shard coordination
- Exact DISTINCT — approximate with HyperLogLog, exact needs full scan
- Exact percentiles (p99) — approximate with t-digest/DDSketch, exact needs all values sorted
- Correlated subqueries

**The rule**: if an aggregation can be computed locally per shard and combined (sum of sums, max of maxes), it works on the fast path. If it needs a global view, it goes to the slow path.

This is why metrics systems expose a limited query API — not full SQL, but a fixed set of aggregation functions that decompose across shards. Prometheus, Datadog, Scuba all follow this constraint.

# Virtual Filesystem — Path to Blob Resolution

The terminal UI presents a synthetic filesystem. Paths look like:
```
~/namespace/service/2027/03/13/14_15/xyz.log
```

**What each segment maps to:**
- `namespace/service` → identifies the service
- `2027/03/13/14_15` → year/month/day/hour_minute → time bucket
- `xyz.log` → specific log file within that bucket

**On `cd`**: filters the metadata table by path prefix — no blob fetch, just a metadata query. Fast.

**On `cat` or `grep`**:
1. Parser extracts `(namespace, service, time_bucket, filename)` from the path
2. SQL query against metadata table → returns blob URL
3. Blob is fetched from blob storage (too large for memory — streamed)
4. `grep` runs server-side on the stream, matching lines sent back over WebSocket

**Caching**: blob is not held fully in memory. If the user greps the same file multiple times, the server may cache the blob on local disk (SSD) for the session duration — fast re-grep without re-fetching from blob storage. Evicted when WebSocket session closes.