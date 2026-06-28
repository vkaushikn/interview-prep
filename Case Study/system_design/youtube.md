# Case Study: YouTube (Video Upload & Adaptive Streaming)

## Problem Statement
Design a video-sharing platform: users upload video, the platform makes it watchable at adaptive quality to a massive, highly skewed viewer base.

This reuses most of the Dropbox blueprint (`Case Study/dropbox_case_study.md`) for the upload path and metadata split — the deltas are: video needs **transcoding before it's watchable**, and the CDN is **core infrastructure**, not an edge-case optimization, because the read:write ratio here is orders of magnitude more skewed than file storage.

---

## A — Assumptions & Approximations

### Assumptions (scope)
1. **VOD only** — live streaming deferred (a live manifest is a sliding window that has to be re-fetched as new segments appear; a finished upload's manifest can be generated once, in full — meaningfully different problem, out of scope here).
2. **Search/recommendation excluded** — same scoping move as the News Feed case study excluding ranking; candidate generation/serving stays the same regardless of what eventually orders results.
3. **Comments/social graph excluded** — orthogonal to the storage/streaming problem this case study is about.
4. Upload mechanics (presigned URL, multipart/chunked, server-stamped completion) are **reused as-is from Dropbox D2** — no new design needed there.

### Approximations (sizing)
| Quantity | Estimate |
|---|---|
| Daily active viewers | ~1B |
| Avg. video plays / viewer / day | ~5 |
| Video uploaded / day | ~500K hours (rough public-figure order of magnitude) |
| Avg. video length | ~10 min |

**Derived:**
- Plays/day ≈ 1×10⁹ × 5 = **5×10⁹/day**
- Uploads/day ≈ 500,000 hr ÷ (10/60 hr) ≈ **3×10⁶ videos/day**
- **Read:write ratio ≈ 1,700:1** — vs. Dropbox's ~10:1. This single number is *why* the CDN graduates from "optional add-on for a hot minority" (Dropbox D4) to "load-bearing core" here: at this ratio, nearly every served byte is a repeat read of something already-cached, not a 1-reader file.

**Storage multiplier from transcoding:** each upload produces several renditions (240p…4K), not one file. Rather than computing each rendition's exact size, use a rough multiplier on the raw upload size — call it **2-3x** raw size for the full rendition set (lower-res renditions are individually small but there are several; the multiplier is intentionally an order-of-magnitude placeholder, not a precise figure) — apply this on top of whatever the Dropbox-style EB/year storage math would give for raw uploads alone.

---

## C — Constraints
1. Reuse Dropbox's blob-storage constraint (C1) as-is.
2. **Time-to-first-frame** is the analogous UX-critical latency target to the News Feed's feed-load latency — viewers expect playback to start in a second or two, not after a multi-second stall.
3. **Adaptive quality under varying network conditions** — a constraint Dropbox never had: file downloads either finish or fail, they don't need to gracefully degrade mid-transfer the way a video stream does.

---

## I — Incentives & Trade-offs
- **Transcode-before-serve vs. serve-raw:** pay a one-time processing cost (and a delay before the video is fully watchable in all qualities) in exchange for adaptive streaming and far lower bandwidth cost per view. Same "pay once at write-time, benefit on every read" shape as denormalization/CDN caching — the cost is paid once per upload; the benefit is collected on every one of millions of plays.
- **Manifest + client-side adaptive logic vs. a server that's asked for "the next chunk":** push the bitrate decision entirely into the player, keep the CDN/origin completely unaware that adaptive streaming is even happening — same "smart edge, dumb-but-fast middle" shape as CRDT pushing merge logic to the client instead of requiring a server in the loop for every operation. The win: chunk URLs are deterministic and content-addressable, which is exactly the shape a cache wants.
- **CDN as core, not optional (vs. Dropbox D4):** Dropbox's personal files are mostly 1-reader, so caching almost everything would be pure overhead for the common case. Here the read:write ratio (≈1,700:1) means the overwhelming majority of objects clear the "worth caching" bar — the *mechanism* (pull-through + eviction, see [cdn_vs_edge_cache.md](../concepts/cdn_vs_edge_cache.md)) is unchanged from Dropbox, only the base rate of what qualifies as hot changes, and that shift is large enough to flip the architecture's center of gravity.

---

## D — Decisions (Architectural Blueprint)

### D1. APIs
`Upload (initiate + complete)` — identical to Dropbox D2, reused without modification. `Watch(video_id)`, `Delete`. Search/recommend explicitly excluded (A2).

### D2. Transcoding pipeline
1. Raw upload lands in an ingest blob bucket via the reused Dropbox upload path.
2. Upload-complete event (same webhook pattern as Dropbox D2) enqueues a transcode job onto a **message queue** (the "~100K-1M msgs/sec/broker" anchor from [capacity_benchmarks.md](../concepts/capacity_benchmarks.md)) — fully decoupled from the upload request; the uploader's client has nothing further to do.
3. A worker pool consumes jobs, producing multiple renditions (resolution × bitrate combinations) in parallel, since each rendition is an independent job.
4. **Renditions are prioritized, not produced all-at-once-then-released**: the lowest-resolution rendition is cheap and fast, so the video can go watchable at low quality almost immediately, with higher-resolution renditions swapped into the manifest as they finish in the background — avoids gating "watchable at all" on every rendition completing.
5. Status field needs more granularity than Dropbox's `pending`/`committed`: `uploaded → processing → partially_ready → ready`.

### D3. Manifest-driven adaptive streaming (the core mechanic)
1. Once at least one rendition exists, a **manifest** (HLS `.m3u8` / DASH `.mpd`) is generated — a small file listing every available rendition and a URL *template* per rendition (e.g. `.../video_id/{quality}/segment_{n}.ts`), not an explicit enumeration that needs regenerating per request.
2. The player fetches this manifest **once**, at the start of playback (itself just a small, cacheable static object — no DB involved).
3. From then on, **the player constructs every subsequent chunk URL locally** from the template — there is no "what's the next chunk" round-trip to any server, ever. A chunk request is an ordinary GET against a predictable URL, indistinguishable from requesting any other static asset.
4. **Bitrate adaptation is entirely client-side**: the player tracks recent segment download time vs. playback duration, and current buffer depth. Based on those two signals alone, it picks which quality folder to request the *next* segment from. The CDN and origin have no awareness that adaptive streaming is occurring — they only ever see deterministic GETs.
5. **Live-stream contrast (out of scope per A1, noted for completeness):** a live manifest can't list all future segments, so it's a sliding window the player re-fetches periodically — a meaningfully different mechanism from VOD's "generate once, fully, after transcoding completes."

### D4. CDN (core serving layer, not an add-on)
Same pull-through + eviction mechanism as Dropbox D4 and the distributed cache case study — no pre-placement, popularity discovered purely by request pattern. What's different here: caching is at **chunk/segment granularity**, not whole-file, because every viewer re-requests the same small set of segments (including viewers who join partway through or seek). Given the read:write ratio from Approximations, this is load-bearing infrastructure, not an optional addition gated on "if read cost becomes a problem."

### D5. Storage tiering
Reuse Dropbox D7's mechanism (recency-driven, background daemon, intermediate archive tiers to avoid retrieval-latency surprises) — same design, applied to a pattern that's likely *more* pronounced here: most views on a given video cluster shortly after upload, then taper sharply, making the recency signal even stronger than Dropbox's general file-access pattern.

### D6. Metadata DB
- **Video Metadata** (own cluster, same functional-split reasoning as Dropbox D3): `Video Id, Uploader Id, Status, manifest_location, available_renditions, upload_time, last_accessed_time`.
- **Engagement/view-count table** (separate cluster): view counts update at far higher write QPS than core metadata ever does, and have a different consistency requirement (approximate, eventually-consistent counters are fine for a view counter; they are not fine for "does this file exist") — same reasoning as splitting Dropbox's File Metadata from its sharing ACL table: different growth curve, different bottleneck, split rather than bundled.

---

## Failure Modes
1. **Transcode worker crashes mid-job.** Job needs to be safely re-queued (idempotent transcode — re-running shouldn't produce a corrupt duplicate rendition); partial output from the crashed attempt needs cleanup, same shape as Dropbox's abandoned-multipart-upload cleanup.
2. **CDN cache stampede on a suddenly-viral video.** Direct reuse of the distributed cache case study's stampede failure mode — a video crossing from cold to extremely hot faster than cache population can keep up causes a burst of origin requests; mitigated the same way (request coalescing at the edge).
3. **Player requests a rendition that isn't ready yet** (still mid-transcode, D2 step 4). Falls back to the lowest available rendition rather than failing — the same "graceful degradation" instinct as the manifest only ever listing renditions that actually exist yet.

---

## Metrics and Monitoring

| Metric | What it tells you | Collected how |
|---|---|---|
| Rebuffering rate | The core streaming UX metric — direct measure of whether ABR is keeping up with real network conditions | Player-reported event, aggregated |
| Time-to-first-frame | Startup latency, the streaming analog of feed-load latency | Per-session timer |
| Transcode queue depth/lag | Is upload-to-watchable latency under control | Queue depth gauge |
| CDN hit rate (chunk-level) | Whether the core serving layer (D4) is actually absorbing the read:write skew it exists for | Per-edge counter, same as cache case study |
| Per-rendition request distribution | Skewed toward low-res = many viewers on poor networks (or D2's prioritization isn't surfacing higher quality fast enough) | Counter per rendition |
| Storage cost by tier | Whether D5's tiering is moving the needle, same as Dropbox | Periodic gauge |

---

## See also
- [Case Study: Dropbox](dropbox_case_study.md) — the reused upload path, metadata-split reasoning, and storage tiering mechanism.
- [cdn_vs_edge_cache.md](../concepts/cdn_vs_edge_cache.md) — why CDN amplification is the dominant mechanism here vs. the edge-case role it played for Dropbox.
- [distributed_cache_case_study.md](distributed_cache_case_study.md) — cache stampede failure mode, reused as-is.
- [capacity_benchmarks.md](../concepts/capacity_benchmarks.md) — message-queue throughput anchor used for the transcode job queue.

## Open items / things to revisit
- Live streaming (sliding-window manifest) — explicitly out of scope, noted as a meaningfully different mechanism rather than an extension of VOD.
- Exact rendition set / bitrate ladder — named conceptually, not tuned with real numbers.
- Recommendation/search — excluded by assumption, same move as News Feed's ranking exclusion.
