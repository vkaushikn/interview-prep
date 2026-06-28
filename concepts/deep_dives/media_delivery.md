# Media Delivery (Audio & Video)

**Standard stack:** Blob storage (S3) → CDN → client. Pre-signed URLs control access. HTTP range requests (HTTP 206 Partial Content) allow seeking without re-downloading. Nothing exotic here — this part you do not need to design, just state it.

## The one structural choice: full download vs. adaptive bitrate

**Audio (Spotify-like):** files are 3–10 MB. Progressive download is sufficient — client starts playing after a small initial buffer, finishes downloading in the background. No need for adaptive bitrate. Pre-signed URL per song, short TTL (5–15 min), CDN serves the bytes.

**Video (YouTube-like):** files are 100 MB – GB+, and bandwidth varies. You must chunk. The two protocols:
- **HLS** (Apple): file split into `.ts` segments + a `.m3u8` manifest per quality tier. Best iOS/Safari support.
- **DASH** (MPEG standard): same idea, `.mp4` fragments + XML manifest. More open, better cross-platform.

Both serve multiple quality tiers (e.g. 360p / 720p / 1080p / 4K). The client's ABR algorithm picks quality per segment based on measured throughput and buffer health. URLs are content-addressable (`segment_042_720p.ts`) so CDN caching is trivial — same URL always means same bytes.

**Interview rule of thumb:** audio → progressive download is defensible. Video → you need chunked ABR, non-negotiable.

## Pre-signed URL TTL

Short TTL (5–15 min) prevents link sharing but requires renewal for long content. For audio this is a non-issue (songs < 5 min). For a 2-hour movie with a 15-min URL, the client needs ~8 renewals. Typical pattern: client requests a fresh URL from a signing service shortly before the old one expires. The signing service validates the user's session before issuing.

## CDN pre-warming

Plays follow a heavy Zipf distribution — top 1% of songs ≈ 99% of plays. Two strategies:
- **Hot content:** proactively push to all CDN PoPs when play count crosses a threshold (daemon-driven).
- **Long-tail:** lazy pull — first request to a PoP triggers a cache miss and fetches from origin; subsequent requests at that PoP hit cache.

The cutoff threshold is a tuning knob, not a fixed rule.

## Playback resume state

Where you store "how far into the song/video the user is":
- **Client-side only:** simple, zero server writes, but lost on device switch.
- **Server-side:** enables cross-device resume (Spotify's "continue on your laptop" feature). Checkpoint on *pause/stop* events only — not every second — keeps write volume low. The tradeoff is losing a few seconds of progress on an unexpected crash, which is acceptable for media.

First surfaced in: Spotify case study (audio delivery mechanics, resume state punt).
