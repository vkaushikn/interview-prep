Note: Typing up quicikly for feedback. 

# Problem
Design a system that shows a "Live Stream" in a social network app

# Assumptions 
1. Discovery  is out of bounds (will be handled by the hybrid fan out on write + fetch posts at service time approach)
2. Live Stream of the video and Live Reactions underneath it are two separate "services" (Ask where to focus)
3. Live Stream -- Only one "quality" tier supported. Slower clients will "buffer" more
4. Live Stream -- Only the live stream is supported (the user being to slide a timer to the past and re-watch is out of scope) (may just touch on this, not sure)
5. Live Comments -- No sorting on client side (messages appear based on recieve time -- synchronized?). Can support "nesting" of replies
6. Post event cleanup -- comments maybe re-ordered for "reply -following- original comment"
7. "Hybrid" System -- Live stream of Soccer world cup finals (watched my millions in parallel) v/s Live Stream by an Influencer (live watch / comment count is smaller)
7.a: Focus on the latter first
8. Will try and build a system for "X M watching counter" (Don't ding if it is missing in final design)

# Approximations
1B DAU, say 5% do a Live Stream on any given day --> 50M Live Streams to be supported --> 50 e6 / 1e5 = 500 Live Streams / Second  (2000 Live Streams / Second Peak)
On any Live Stream, we expect the Live Stream to be driven by the "Influencer" tier (lots of followers), so we expect say 80% of the Live Streams (40M) to have a huge follower (lets say 100k people follow/ interact)
and the bottom 20% have very few interactions (let us say 10 people follow/ interact). So number of "Live Impressions" is 40e6 * 100e3 = 4T total (math error in original: 40M × 100k = 4T not 40B). This is total across all streams per day, not per second.

Realistic comment throughput per stream: 2000 concurrent streams × 100k watchers = 200M concurrent viewers. 200M × 1% commenting = 2M comments/sec total. 2M / 2000 streams = 1000 comments/sec per stream. One queue per stream handles this easily (queue capacity is ~100k writes/sec, so 1000 comments/sec = 1% utilization). No sharding needed per stream.

The "1000 shards" concern in the design below was based on assuming all 200M viewers comment simultaneously — the 1% commenting assumption fixes this.

Note: The live watches seems to be over counted -- we estimate that 10% of our DAU is watching a live stream -- seems little too high for my gut feel. I am hoping the interviewer can adjust my estimates up or down as the think fit
or I end up designing something that is not really practical in the real world because of my numbers

# Constraints
(Active Constraint: Will address in Design)Live Stream and Live Comments --- should feel live -- low latency (subject to the users internet connection speed) for the user. This means, event streaming should not be choppy and live comments should scroll almost immediately 
as the comments are written. 

(Inactive Constraint: Will disregard in design) Number of viewers viewing this content, the sequence of the comments (time ordering) itself can be compromised to give the Live view (reconcile later, if required)

# Incentives
-- See Constraints section -- I have made the trade-off already, although can also make it here.
Some trade-off of constraints might mean that we have to spend more $ to build out the system -- but the Infra Scale can be handled with policy (who gets to create a live stream, how many -- at what point do we only support active watching, not commenting etc)

For example -- Soccer world cup, with say 500M concurrently watching -- we might turn off comments as it might be just too expensive to build that system. 

Highlight the real incentive here -- the 150B / sec of live interaction is not the main metric to drive toward -- overall it is, but what is more important is the "number of such channels" that we need to support
- 40M Live sTreams from influencer ~ 2000 Live Streams per Second
- Each of the 2000 Live Stream per Second have 100k live engagers per second - 2e3 * 1e5 = 2e8 Live engager / second/ stream. <--- This is the real constraint. If I had made assumptions such that this number was in the 100k /sec (1e5/sec) range, we can easily put
each Stream on a single Queue.  Just because I seem to have gotten myself into a jam with the scale, my design looks more complicated. 

First, Let us do 1e5/sec (so that one Queue is good enough to handle then we can increase)

# Pattern Recognition (before Design)

This system is a **fan-out media pipeline** — one producer (the streamer) broadcasting to many consumers (viewers), with a secondary real-time messaging overlay (comments).

The canonical shape:

```
Producer → Ingest → Queue → CDN (video) / Outbound service (comments) → Consumers
```

This pattern appears in: YouTube Live, Twitch, Instagram Live, Discord stage channels. Each component in this pipeline has well-understood optimization paths:

| Component | Known optimization space |
|-----------|--------------------------|
| Ingest / chunking | Adaptive bitrate (ABR), codec choice (H.264 vs AV1), keyframe intervals |
| Queue | Partition count, replication factor, consumer group lag tuning |
| CDN delivery | Edge PoP placement, cache TTLs, pre-positioning for known large events |
| Comment fan-out | Inbound/outbound service split, sorted set vs queue, client-side deduplication |
| Viewer counter | HyperLogLog approximation, gossip protocol aggregation |

**I am explicitly not optimizing any individual component.** The goal is to show that the overall architecture hangs together and meets the stated scale. A domain expert (video codec team, CDN team, Kafka team) owns each layer. My job is to define the interfaces between them so those experts can work independently.

# Design
(deliberately sketchy — focused on overall coherence, not per-component tuning)
## Live Stream
- Generator creates video -- the service (we only need like 2k of them horizontally scaled) chunks the video up in say 10s blocks and sends it a Live Stream Queue (where consumers can pull out of it). At 1e5 watches / second (the Influencer path), we need not shard the queue because we can match the read/ write throughput (fofr the socccer world cup, we shard based on user_id, the videos are already sharded by event_id so each live stream is its own shard- consistent hashing so that we can keep scaling it up ) [Edit: Came back and cleaned up the numbers, got confused with the Live chatting throughput]
- an AI policy service also reads the video queue -- and runs ML model to signal if the video is violating any policy. If so, maintain a hot cache of Stream ID: In_Policy so that consumers cannot watch it, while killing the original service which was creating the video
- **CDN for video delivery** — the consumer service does NOT serve video directly to 100k viewers. Video chunks go: Generator → Queue → **CDN edge nodes** → viewers. CDN caches the latest chunk at edge locations globally. Viewers pull from nearest CDN edge over HTTP — standard HTTP file serving, no special infrastructure. This is the standard approach (YouTube, Twitch, Instagram Live all do this). The queue feeds a small number of CDN origin servers, not individual viewers. Without CDN, serving 100k concurrent viewers from your own servers is extremely expensive and slow.

- **HLS/DASH protocol** — the standard for chunk-based video delivery. Generator chunks video into 2-10 second segments, uploads to CDN as regular HTTP files. Viewer client (HLS-compatible player) requests the latest chunk every few seconds. The "live" experience is just the client always requesting the newest available chunk. Delay = chunk size (2-10 seconds). For sub-second latency, WebRTC is used instead but is much harder to scale.

- Consumer service (scaled for the number of users watching this) -- Pull from the queue, keep track of where they are and keep buffering newer video. When a new user joins, they seek upto the latest (this gives that "seek back" thing so that the user can re-watch if they want)
- Once the video is over, chunk up and save it as a part of the normal video processing service (different rates, encoding etc)
- Main Failure modes
   -- One queue fails, we need to copy over the data to a new queue (Potential option -- Upload 10s chunks directly to Blob storage and update a table -- Nee queue downloads from Blob upon restarting)
   -- The User service fails, and a new server picks up -- does not know where to start -- I will just punt on this -- We just refill the entire pipefor the user and start at new live location -they can seek back

- Other ideas considered
   -- Similar to Whatsapp (also see Live Comments), maintain a 2 way connection pool. As soon as we write to the queue, it pushes to each active connection on the 2 way pool -- and that connection can be used to serve the next bits of chunk (the same server is feeding the
   next 10s worth of view to the customer)
   -- I think in the previous design, we might be able to do that by using some load balancing -- so that once a "user server" and a "video server" initiate contact, they stay in contact, even if one of them has to scale up. Only the service that scales up requests new connections. (and if one of them fails, then we just drain the quuee completely)


## Live Comments
- Initial idea is similar to whatsapp -- Each user service is connected to a live chat service (the websocket thing) (so we also hav a registry). Messages written by the user (event_id, message_id, parent_message_id, message) come from the one direction -- and get written into the  Messages queue (shard by even_id and user_id - to manage write and read throughput -- Note: I think my earlier numbers were wrong because every write has to be fan-out read also, we might need more shards here). Then the "chat service" pulls from all the chats (maybe this is a single thread in the chat service - it need not do it per connection) -- sorts the messages (the messages gets timestamped when the get written in the queue --some sort of system time) and sends the messages back to each acctive connection on the server.  I can even think of doing this as two separate services -- Inbound and Outbound -- The inbound could just be the user's services' server writing to the queue and the outbound just fetches sorts and writes back (relives the pressure on the inbound and outbound). Background process flush the queue to persistent storage as required. 

-- Main Failure Modes
  -- Just the scale of this system tells me that the design is somehow wrong and there is probably a better way. 1000 shards for each live stream is just ridiculous. I think I made a mistake with my numbers -- if each stream 2000 /sec needs just a few shards (so if we say each live stream has at-most 1e5 concurrent comments / sec -- this might be okay). I think the scale that I drew up, I am either unaware of technologies that exist or major systems design in such a way that they do not support these scales