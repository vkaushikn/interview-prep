# Web Crawler (Distributed)

## Pattern

**Distributed job scheduler with self-generating work** — URLs are jobs, workers fetch and parse them, discovered URLs are new jobs. Queue feeds itself.

```
Seed URLs → URL Frontier → Fetcher Workers → Parser Workers → Blob Storage
                ↑                                    ↓
                └──────── New URLs discovered ────────┘
                                                      ↓
                                              Search Index / Data Store
```

---

## A — Assumptions

1. Crawl the entire web — ~1B URLs
2. Recrawl frequency: popular pages every few hours, others every few days
3. Respect robots.txt per domain
4. Politeness: max 1 request per domain per second
5. Output: raw HTML to blob storage + extracted text to search index

---

## C — Constraints

| Constraint | Active? | Notes |
|------------|---------|-------|
| No duplicate crawls | Yes | Bloom filter for visited URLs |
| Respect robots.txt | Yes | Cache robots.txt per domain |
| Politeness (rate limit per domain) | Yes | Domain-level queue throttling |
| Fresh content for popular pages | Yes | Priority queue by recrawl frequency |
| Handle broken/slow URLs | Yes | Timeout + retry with backoff |

---

## Approximations

- 1B URLs, average page size 100KB → 100TB raw HTML
- Crawl rate: 1000 pages/sec → 1B pages in ~12 days
- 1000 fetcher workers × 1 request/sec each = 1000 pages/sec
- Politeness: 1000 domains being crawled simultaneously, 1 req/domain/sec

---

## I — Incentives / Trade-offs

- **Bloom filter over hash set** — 1B URLs × 50 bytes = 50GB for exact hash set. Bloom filter stores same data in ~1GB with <1% false positive rate. False positive = skip a URL never seen before (miss one page). False negative never happens — never recrawl a visited URL. Acceptable trade-off.
- **Domain-level rate limiting** — without politeness, crawler hits one domain with 1000 concurrent requests and gets IP-banned. Rate limit per domain, not per worker.
- **Priority queue for URL frontier** — not all URLs equal. Popular pages (CNN homepage) recrawled hourly. Obscure blog posts recrawled monthly. Priority based on page rank + time since last crawl.
- **Separate fetcher and parser workers** — fetching is I/O bound (network), parsing is CPU bound. Scale independently.

---

## D — Design

### URL Frontier

The queue of URLs to crawl. Two levels:

```
Priority Queue (Redis sorted set)
    score = recrawl_priority (based on page rank + time since last crawl)
    → high priority URLs crawled first

Per-domain queues (one queue per domain being crawled)
    Politeness enforcer: release one URL per domain per second
    → prevents hammering a single domain
```

**Flow**: URL Scheduler pulls from priority queue → routes to correct domain queue → fetcher workers pull from domain queues.

---

### Visited URL Deduplication

```
Bloom Filter (in-memory, ~1GB for 1B URLs)
    before adding any URL to frontier:
        if url in bloom_filter: skip (already crawled or false positive)
        else: add to bloom_filter, add to frontier

Persistent backup: Bloom filter checkpointed to blob storage every hour
    on restart: reload from checkpoint, don't lose visited state
```

---

### Fetcher Workers

```
Fetcher Worker (1000 instances, I/O bound):
    1. Pull URL from domain queue
    2. Check robots.txt cache for this domain
       → if disallowed: skip, mark visited
    3. HTTP GET with timeout (5 seconds)
       → on timeout/error: retry queue with exponential backoff (max 3 retries)
    4. Write raw HTML to blob storage: blob/{domain}/{url_hash}.html
    5. Publish to Parser Queue (Kafka): {url, blob_path, crawl_timestamp}
    6. Mark URL as visited in bloom filter
```

**Robots.txt cache**: `domain → rules`, TTL 24 hours. Fetched once per domain per day, not per URL.

---

### Parser Workers

```
Parser Worker (CPU bound, scale based on parsing backlog):
    1. Consume from Parser Queue
    2. Download raw HTML from blob storage
    3. Extract:
       - Text content → search index (Elasticsearch)
       - Outgoing links → new URLs to crawl
       - Metadata (title, description, last-modified)
    4. For each new URL:
       - Normalize (remove fragments, lowercase, canonicalize)
       - Check bloom filter
       - If not visited: add to URL frontier with priority score
    5. Publish CRAWL_COMPLETE event to Kafka (for analytics, recrawl scheduling)
```

---

### Recrawl Scheduling

```
Recrawl Scheduler (background service):
    consumes CRAWL_COMPLETE events from Kafka
    computes next crawl time:
        popular pages (high page rank): now + 1 hour
        normal pages: now + 7 days
        rarely linked pages: now + 30 days
    re-inserts URL into priority queue with updated score
```

---

### Storage

```
Blob Storage (S3/GCS):
    raw HTML: blob/{domain}/{url_hash}.html
    bloom filter checkpoints: bloom/{timestamp}.bin
    robots.txt cache backup

Search Index (Elasticsearch):
    document per URL: {url, title, text_content, page_rank, last_crawled}
    indexed by text content for search queries

Crawl Metadata DB (Postgres):
    {url, last_crawled, crawl_status, page_rank, next_crawl_time}
    used by recrawl scheduler and analytics
```

---

## Failure Modes

**Fetcher worker crashes mid-crawl:**
- URL not marked visited (bloom filter not updated)
- URL stays in domain queue, gets re-fetched by another worker
- Duplicate raw HTML written to blob — acceptable, parser deduplicates by URL

**Bloom filter lost (Redis crash):**
- Reload from last checkpoint (up to 1 hour of recrawls)
- Brief window of duplicate crawls — acceptable

**Parser queue backs up:**
- Fetchers slow down automatically (Kafka back-pressure)
- Scale up parser workers horizontally
- Raw HTML safely in blob storage — no data lost

**Domain blocks crawler:**
- Fetcher gets 429 or 403
- Exponential backoff per domain
- Respect Retry-After header if provided

**Slow/unresponsive domain:**
- 5 second timeout per request
- After 3 failures: mark domain as temporarily unavailable
- Retry domain after 24 hours

---

## Key Numbers

| Metric | Value |
|--------|-------|
| Total URLs | ~1B |
| Crawl rate | ~1000 pages/sec |
| Fetcher workers | 1000 |
| Raw HTML storage | ~100TB |
| Bloom filter size | ~1GB |
| Bloom filter false positive rate | <1% |
| Robots.txt cache TTL | 24 hours |
| Politeness rate | 1 req/domain/sec |

---

## Comparison to Single-Machine Async Crawler

The async web crawler (see `coding/concepts.md`) is the single-machine version of this:

| | Single Machine | Distributed |
|--|--|--|
| Visited set | `asyncio.Lock` + Python set | Bloom filter in Redis |
| Queue | `asyncio.Queue` | Kafka + domain queues |
| Workers | `asyncio.create_task` | 1000 fetcher instances |
| Politeness | Not implemented | Per-domain rate limiter |
| Storage | In-memory | Blob + Elasticsearch |
| Scale | ~100 URLs/sec | ~1000 URLs/sec |
