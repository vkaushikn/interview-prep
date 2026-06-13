# CDN vs. Geo-Distributed Edge Cache

**CDN (Content Delivery Network):** designed for content that is *identical across many users* — static assets, images, video segments, cacheable API responses. One cached copy at an edge location (PoP) serves every nearby user's request for that same object. The value comes from amplification: cache once, serve millions of times.

**Geo-distributed edge cache (e.g., regional Redis/Memcached clusters at PoPs/DCs):** designed for *personalized* data — every user has a different value (their own precomputed feed). Each entry is read by ~one user. The value here isn't amplification — it's **placement**: putting each user's data physically close to where that user reads it, so the network round trip doesn't dominate the latency budget (see [Latency Numbers](latency_numbers.md)).

## Why the distinction matters
"Cache the feed in a CDN" is a common interview phrase that doesn't quite hold up under a follow-up: a CDN's caching model assumes shared content. A personalized feed needs a **regional key-value store keyed by user ID**, replicated/sharded across geographic regions — same geographic-placement benefit, different mechanism (no fan-out amplification, just locality).

## Applied to the News Feed
The "serving" layer is a regional Redis-like store, one entry per user (their precomputed last-20-post feed), placed in the region closest to that user. The "creation" layer (daemon) is what keeps those entries fresh.

First surfaced in: News Feed case study (D — serve/create decoupling).
