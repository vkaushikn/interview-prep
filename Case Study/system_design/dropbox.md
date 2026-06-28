# Case Study: Dropbox (Cloud File Storage & Sharing)

## Problem Statement
Design a cloud file storage service: users upload files, access them from multiple devices, and share individual files with other users.

(Original working draft, kept as-is: `practise/dropbox.md`. This is the cleaned-up version after working through the math and a few architectural gaps.)

---

## A — Assumptions & Approximations

### Assumptions (scope)
1. **Multi-device access** — upload from one device, view/download from any other.
2. **Per-file sharing** — no folder-level permissions (folders/directory structure deferred entirely).
3. **Offline sync deferred** — not solving "is the local folder in sync with the cloud" for v1.
4. **No versioning, no merge.** Two concurrent uploads to the same file: **silent last-write-wins**, decided by server-assigned commit time (not client clock — client clocks can skew across devices and would make the wrong upload "win"). This is a deliberate, explicit scoping choice, not an oversight — see Incentives below for the upgrade path we're naming but not building.
5. Geography/multi-region ignored for v1.

### Approximations (sizing)
| Quantity | Estimate |
|---|---|
| DAU | 10M |
| Files uploaded / user / day | 5 |
| Max file size | 1GB (high end); 100MB (low end, for a range) |
| Reads vs. writes | ~10:1 (30-50 file accesses/day vs. 5 uploads/day) |

**Storage** (method: see [capacity_benchmarks.md](../concepts/capacity_benchmarks.md)):
- Files/day = 10M × 5 = 5×10⁷
- High end (1GB avg): 5×10⁷ × 1×10⁹ bytes = 5×10¹⁶ bytes/day = **50 PB/day ≈ 18 EB/year**
- Low end (100MB avg): 5×10¹⁵ bytes/day = **5 PB/day ≈ 1.8 EB/year**
- Range: **~1.8–18 EB/year**, depending on average file size. (Earlier pass mislabeled this as PB instead of EB — same digits, wrong unit by 1000x. Worth double-checking unit labels, not just the arithmetic, when the answer crosses a ladder boundary.)

**QPS:**
- Writes: 5×10⁷ / 10⁵ sec/day ≈ **500/sec**
- Reads: ~10× writes ≈ **5,000/sec**
- These are *averages*. No peak multiplier applied yet — if pushed, assume 3-5x for daily traffic skew before concluding "no cache needed" on the read path.

**Metadata DB sizing** (method: row-size buckets in [capacity_benchmarks.md](../concepts/capacity_benchmarks.md)):
- **File Metadata** (`File Id, File Name, File Type, Blob Key`): ~91B rows over 5 years (5×10⁷/day × 365 × 5), ~300 bytes/row effective → **~27TB**. Within a single Aurora cluster's hard ceiling (~50-100TB).
- **User MetaData / sharing ACL** (`User Id, File Id, is_owner`): grows multiplicatively with sharing fan-out (assume ~2-3x file count) → ~270B rows, ~11TB raw — but the **index** on `User Id` (needed for "all files for user X") is ~4.3TB by itself. That's the real constraint: a multi-TB *index* needing to live in RAM for fast lookups exceeds a realistic single-node RAM budget (~0.5-4TB), well before disk capacity does.
- **Conclusion:** disk was never the binding constraint — RAM for the sharing table's index is. This is why the sharing table is split onto its own Aurora cluster (see D-decisions) rather than co-located with File Metadata: they have different growth curves and different bottlenecks, and bundling them would force File Metadata to shard/resize on a schedule it doesn't need.

---

## C — Constraints
1. **Use industry-standard blob storage** (S3-class) for file bytes — inherit its durability/replication; budget is the constraint on going further (e.g. multi-cloud blob replication).
2. **Low latency within a working session** — e.g. upload-on-device-A, immediately-read-on-device-B, or a download→modify→upload loop.
3. **Consistency** — a read should return the most recently *committed* version of a file. Not "the most recent upload attempt," but "the most recent one that successfully finished."

---

## I — Incentives & Trade-offs

- **Consistency (C3) vs. low latency (C2):** the tension resolves cleanly once you separate the two writes. The blob write is the slow, unavoidable part (network + storage durability). The metadata write (the thing that makes a file "visible") is small and fast. As long as metadata only commits *after* the blob write succeeds, you get correctness for free — the only latency cost you can't engineer away is the blob upload itself.
- **Functional split of the metadata DB (File Metadata vs. sharing ACL), not just horizontal sharding:** these two tables have different growth curves (1:1 with files vs. multiplicative with sharing fan-out) and different bottlenecks (disk vs. RAM-for-index). Putting them on separate Aurora clusters means each scales on its own schedule. The cost: no single-query SQL join across them — acceptable, since the access pattern was already "look up by key" against each table independently, not a relational join.
- **Silent last-write-wins vs. optimistic-locking (version check, reject with 409 on conflict):** chose silent LWW for v1, consistent with the "no versioning" scope decision in Assumptions. This *makes the conflict deterministic, it does not resolve it* — a real concurrent edit can still silently discard one user's work with no signal to either party. The cheap upgrade (add a version counter, reject a stale commit instead of overwriting) is a few lines of design on top of infrastructure we already have (the same metadata row), named here as the explicit next step if pushed, not built into v1.
  - One level deeper than that — actually preserving and merging both concurrent versions — is a different, much harder problem (real-time collaborative editing: CRDTs / Operational Transformation, see [crdt_vs_ot.md](../concepts/crdt_vs_ot.md)). Explicitly out of scope; named so it's clear this is a deliberate boundary, not a gap.
- **Direct-to-blob upload (presigned URL) vs. proxying through the app server:** at up to 1GB/file, routing bytes through application compute is the textbook anti-pattern for this problem shape. Presigned URLs let the client talk straight to blob storage; the app server only ever handles metadata.
- **CDN amplification vs. personal-data locality:** a CDN's value is caching one copy and serving it to many readers — most files here have ~1 reader, so that value doesn't exist for them (see [cdn_vs_edge_cache.md](../concepts/cdn_vs_edge_cache.md)). Caching everything would be pure overhead for the common case to (maybe) help the uncommon one; instead, caching is left to emerge naturally per-object via pull-through + eviction (D4), rather than being a blanket layer in front of all reads.
- **Storage-cost reduction (tiering) vs. read-path simplicity:** the cheapest possible design serves every read identically, with no tiering at all. At EB scale that leaves a large, avoidable cost on the table. The trade: D7's tiering daemon adds an access-tracking write to D4 and a background job, in exchange for moving the (likely-majority) of rarely-read files to far cheaper storage — accepted because the write is async/off the critical path, so it doesn't cost anything on the latency budget that matters (C2).

---

## D — Decisions (Architectural Blueprint)

### D1. APIs
`Upload (initiate + complete)`, `Download`, `Delete`, `Share`.

### D2. Upload path
1. Client calls `Upload/initiate(user_id, file_name, size)`.
2. Server creates a `pending` File Metadata row, generates a **presigned PUT URL** (or, for files near the 1GB ceiling, initiates a **multipart upload** and returns one presigned URL per chunk — gives resumability for free: a failed chunk is retried individually, not the whole file).
3. Client uploads directly to blob storage. App server is out of the data path entirely.
4. Completion is detected one of two ways:
   - **Client-driven**: client calls `Upload/complete` after the PUT(s) succeed → server flips the row to `committed`. Simple; if the client crashes between PUT and this call, you get an orphaned blob (harmless, garbage-collected later) but no metadata-loss risk.
   - **Storage-driven (webhook)**: blob storage emits a "PutObject complete" event → a queue/webhook → server commits the row. More robust against a misbehaving/crashed client, small added latency before the file shows as available.
5. Metadata commit uses the **server's** timestamp as `upload_time` — never the client's clock — so two racing uploads have a well-defined, skew-proof winner. (The DB's own row-level commit ordering already gives last-write-wins on a blind overwrite; `upload_time`/`uploaded_by` are for audit/attribution, not for producing the winning behavior itself.)

### D3. Databases (two separate Aurora clusters — see Incentives)
- **File Metadata** (own cluster): `File Id, File Name, File Type, Blob Key, Size, Checksum, Owner Id, Status (pending/committed), upload_time, last_accessed_time, access_count, storage_tier` — indexed on `File Id`. The last three fields exist purely to drive D7 (storage tiering).
- **User MetaData / sharing ACL** (own cluster): `User Id, File Id, is_owner` — indexed on `User Id` (the "all files for this user" access pattern).
- **User table**: `User Id → Email` — trivial size (~10M rows), no special handling needed.
- **Session/Attribution table** (if/when collaborative features exist): out of scope for plain Dropbox; see [crdt_vs_ot.md](../concepts/crdt_vs_ot.md) for why this would be a separate table from the above if ever added.

### D4. Download path
Server issues a **presigned GET URL**; client pulls bytes directly from blob storage. That server call is also where `last_accessed_time` and `access_count` on the File Metadata row get bumped (async, fire-and-forget — not on the latency-critical path) purely as a side effect, since the app server is already in that one step of the path. This is the access signal D7 runs on.

A CDN in front of blob storage is **not** a blanket addition — most files are personal data with ~1 reader, and a CDN's whole value (cache once, serve many) doesn't apply to them; they're served straight from blob storage on every read, same as if no cache existed. A CDN only helps the minority of genuinely shared/hot files, and the right model for *which* files end up cached is **pull-through, not pre-placement**: the first request for an object at a given edge is a forced miss back to blob storage, cached at that edge, and subsequent requests at that same edge hit the cache until normal eviction. Nobody decides in advance "this file is popular in this region" — popularity is discovered purely by request pattern, the same way the cache case study's eviction policy discovers hot keys. Not built into v1; flagged as the first addition if read latency/cost on the shared-file minority becomes a problem.

### D5. Delete
Soft-delete: mark the File Metadata row deleted, leave the blob in place. A background daemon physically deletes the row + blob after a grace window (e.g. 10 days), which incidentally gives a free "restore" feature.

### D6. Share
Add a row to the sharing ACL table for the target email. If the email has no existing `User Id`, create one (pending-account) and send an invite. **Known, accepted risk**: this is an email-enumeration / unsolicited-mail vector — flagged, not mitigated, in v1.

### D7. Storage tiering (hot → cold)
At 1.8–18 EB/year, storage cost is one of the largest levers in this design. A background daemon (same shape as D5's delete daemon) periodically scans File Metadata's `last_accessed_time`/`access_count` and transitions a file's blob to a cheaper storage class once it crosses a staleness threshold (e.g., no read in 90 days).

- **Signal: recency dominates, frequency is a tiebreaker** — the same LRU-vs-LFU tension as the distributed cache's eviction policy. Age-since-*upload* is the wrong metric (a 3-year-old file opened yesterday is still hot); age-since-*last-access* is the right one. Pure frequency has the same aging problem noted in the cache case study — a file popular two years ago but untouched since still "wins" on raw count unless decayed.
- **Custom daemon vs. native cloud lifecycle rules:** cloud blob stores offer built-in age-based tiering for free, but those only see the blob's own age, not actual read pattern — fine as a cheap default, but blind to a file that's old-but-still-read. The custom daemon is more accurate because it's driven by the access signal from D4, at the cost of being infrastructure you build and run.
- **Tier is a spectrum, not hot/cold binary**, because true archive tiers carry real **retrieval latency** (minutes to hours) and a per-retrieval fee — a user clicking "download" on a 2-year-old file and waiting hours is a bad regression for a product whose value is instant access. The threshold should map to an intermediate "instant-retrieval archive" tier for most aged-out files, reserving the slowest/cheapest deep-archive tier only for files aged out far enough that the wait is an acceptable trade for the cost savings.

---

## Failure Modes

1. **Blob write succeeds, metadata commit never happens** (client crash, or webhook delivery failure). Result: an orphaned blob with no metadata row — harmless, found and reclaimed by a periodic GC sweep over blob storage cross-referenced against committed File Metadata.
2. **Two devices upload to the same file concurrently.** Resolved by design (D2/Incentives) as silent last-write-wins via server-stamped commit order. The explicitly accepted cost: the losing upload's content is gone with no signal to its user. Named upgrade: optimistic version check → 409 instead of silent overwrite.
3. **Abandoned multipart upload** (client disappears mid-upload, never calls `complete`). A `pending`-status row with no activity past a TTL is cleaned up by the same background daemon as D5, along with any partial blob chunks.

---

## Metrics and Monitoring

| Metric | What it tells you | Collected how |
|---|---|---|
| Upload funnel: `initiate` count vs. `complete` count | Drop-off rate — flaky networks, client bugs, abandoned uploads | Counters at each API step |
| Upload / download p50/p99 latency | User-facing experience; large-file tail latency specifically | Per-request histogram, scraped periodically |
| Metadata DB QPS (per cluster) | Headroom against the ~1K-10K qps/node anchor ([capacity_benchmarks.md](../concepts/capacity_benchmarks.md)) | Counter/rate per cluster |
| Storage growth rate vs. projection | Catches the EB/year estimate being wrong in either direction, early | Periodic gauge on blob storage usage |
| Delete-daemon queue depth / lag | Is the restore-window grace period being honored; is GC keeping up | Queue depth gauge |
| Conflict/overwrite rate (same File Id, two `upload_time`s within a short window) | Surfaces how often the accepted last-write-wins cost is actually being paid — the business case for ever building the optimistic-lock upgrade | Derived query over File Metadata's `upload_time` history, not a live counter |
| Orphaned-blob count (found by GC sweep) | Health of the upload-complete detection path (D2) | Counter from the GC job |
| Storage cost by tier (hot / instant-archive / deep-archive) | Whether tiering (D7) is actually moving the needle on the dominant cost line at EB scale | Periodic gauge, joined against File Metadata's `storage_tier` |
| Cold-tier retrieval rate ("rehydration" requests) | Catches a mis-tuned tiering threshold — files getting archived too aggressively, then immediately re-requested and paying the retrieval-latency/cost penalty | Counter on the retrieval-trigger path |
| CDN hit rate on the shared-file minority | Confirms the pull-through cache (D4) is actually being earned by genuinely hot files, not warming on noise | Per-edge counter, same as the cache case study's hit-rate metric |

**Debugging pattern:** upload-funnel drop-off + flat error rate → likely client/network, not server. Metadata QPS climbing with flat traffic → check for an N+1 pattern in the sharing-ACL lookups. Conflict/overwrite rate non-trivial → the strongest signal that silent LWW has a real cost and the optimistic-lock upgrade is worth building, not just a theoretical concern.

---

## See also
- [capacity_benchmarks.md](../concepts/capacity_benchmarks.md) — row-size buckets and the storage-ceiling (disk vs. RAM) distinction used in the Approximations section.
- [crdt_vs_ot.md](../concepts/crdt_vs_ot.md) — the next level of depth on conflict resolution (real-time collaborative editing), explicitly out of scope here.
- [cdn_vs_edge_cache.md](../concepts/cdn_vs_edge_cache.md) — why CDN amplification doesn't apply to the personal-data majority of this corpus (D4, Incentives).
- [distributed_cache_case_study.md](distributed_cache_case_study.md) — the LRU-vs-LFU eviction tension reused as-is for D7's tiering signal, and the hit-rate/eviction metrics reused for D4's pull-through cache.

## Open items / things to revisit
- Optimistic-locking upgrade (version check, reject-with-409) — named as the cheap improvement over silent last-write-wins, not built.
- Content-addressable storage / block-level dedup — flagged as a real cost lever at multi-EB/year scale, not designed.
- Peak-traffic multiplier on read QPS — average-only number used; revisit before concluding "no cache needed."
- Exact age/access-count threshold for D7's tiering daemon, and which intermediate archive tier each threshold maps to — named as a real decision, not yet tuned with real numbers.
