# CDN vs. Geo-Distributed Edge Cache

**CDN (Content Delivery Network):** designed for content that is *identical across many users* — static assets, images, video segments, cacheable API responses. One cached copy at an edge location (PoP) serves every nearby user's request for that same object. The value comes from amplification: cache once, serve millions of times.

**Geo-distributed edge cache (e.g., regional Redis/Memcached clusters at PoPs/DCs):** designed for *personalized* data — every user has a different value (their own precomputed feed). Each entry is read by ~one user. The value here isn't amplification — it's **placement**: putting each user's data physically close to where that user reads it, so the network round trip doesn't dominate the latency budget (see [Latency Numbers](latency_numbers.md)).

## Why the distinction matters
"Cache the feed in a CDN" is a common interview phrase that doesn't quite hold up under a follow-up: a CDN's caching model assumes shared content. A personalized feed needs a **regional key-value store keyed by user ID**, replicated/sharded across geographic regions — same geographic-placement benefit, different mechanism (no fan-out amplification, just locality).

## Applied to the News Feed
The "serving" layer is a regional Redis-like store, one entry per user (their precomputed last-20-post feed), placed in the region closest to that user. The "creation" layer (daemon) is what keeps those entries fresh.

First surfaced in: News Feed case study (D — serve/create decoupling).

## Commercial CDN mechanics (cache key, pricing, implementation)

**Cache key is mostly the URL, but it's a configured policy, not a fixed rule.** Default key = scheme + host + path (+ query string, by default on most CDNs). Real CDNs (Akamai, Cloudflare, Fastly, CloudFront) let the origin explicitly configure what participates in the key:
- Strip incidental query params (tracking parameters like `?utm_source=`) so equivalent content isn't treated as distinct objects and needlessly missing cache.
- Extend the key via the `Vary` response header (e.g. `Vary: Accept-Encoding` keeps separate cached copies per encoding, since those really are different bytes).
- Cookies are excluded from the key by default — including them would make most responses effectively uncacheable (cookies are usually unique per visitor). Accidentally varying on a header/cookie that doesn't actually change the response is a common, silent hit-rate killer.

The practical implication: design URLs to be deterministic/content-addressable in the first place (e.g. YouTube's `segment_{n}.ts` per quality level — same URL always means same bytes, forever) rather than relying on the CDN to paper over noisy URLs after the fact.

**Pricing distinguishes edge→user from edge→origin traffic.** Edge-to-end-user bandwidth is the main billed line item, typically tiered by geography (NA/EU cheapest; APAC, etc. typically higher, reflecting real backhaul cost differences) — plus often a small per-request fee on top of bandwidth. The edge→origin leg (cache misses) is usually handled by an **origin shield**: an additional caching tier between the edge fleet and the actual origin, consolidating misses from many edges into far fewer origin-bound requests. Usually a separate paid tier, worth it because origin-side egress (e.g. your own S3/data-center bandwidth bill) is often pricier than the shield tier's own cost. Using a CDN from the same cloud provider as your origin storage (e.g. CloudFront in front of S3) commonly avoids or discounts the data-transfer charge between them, since the bytes never leave the provider's own network.

**Implementation is purpose-built HTTP cache software, not generic Redis** — same conceptual shape (key→value, TTL/eviction) as any cache, but optimized differently: HTTP-semantics-aware (`Vary`, `Cache-Control`, byte-range requests for video seeking), and backed by a RAM+disk hybrid rather than RAM-only, since an edge wants to cache a working set far larger than fits in memory alone. Historically well-known concrete example: **Varnish** (open-source HTTP cache/reverse-proxy; Fastly's product was built on a heavily modified version for years). Major commercial CDNs run their own proprietary systems — specifics aren't public, but the general shape (purpose-built, HTTP-aware, hybrid storage) holds across the category.

First surfaced in: Dropbox/YouTube case study tangent, working through how a commercial CDN actually maintains and prices the edge cache.
