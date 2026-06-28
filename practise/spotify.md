# Design Spotify
---
App to explore and listen to songs

## Assumptions
(Use this to scope down the problem)
Note: We should pick just one "scope" in the interview -- the idea here is to explore as much as I want to. 

User specific workflow
1. User can create multiple playlist. Each playlist can have 1 or more songs.
2. User can search a song by exact specification -- Artist, Album, Song (Arbitrary search is Out of scope) and add it to playlist
3. User can remove song from playlist
4. User can share playlist with friends (deprioritizing for now)
5. User can listen to playlist
    - listent exactly to playlist;
    - no random songs
    - no monetization (ads)
6. User can like /dislike song 

Song specific workflow
7. Show Today's top songs by number of listen
    - arbitrary time horizon
    - Top songs among my friends
8. Show "Similar" Songs
    - Blend in User Data (# of songs, thubms up, Repeat listens)
    - Blend in Song Data (Genre, Artist, etc)
    - Suggest a new song
9. Play Ads
    - Ads Inventory
    - Ads Matching
    - Metrics??

10. (Came back here) -- Not tracking "state" of the song being played -- can the user just pause and pick up later? Maybe it is trivial but I punted here.


## Approximations
(Use this to understand the scale of the problem)
0. Total number of users: 1B
1. Number of DAU: 500M 
2. Number of Songs: 2B (say 10MB per song) - 2e9 * 1e7 = 2e16 = 20 PB of Songs
    - Songs database consists of things like song_id, artist_id, genre_id, song_url
    - 2e9 * (8 + 8 +8 + 300 bytes) = 2e9 * 5e2 bytes = 1e12 ~ 2 to 3 GB
3. Number of Songs listened to per day per User: 15 (roughly 1 hour of listening to music)
    - Throughput is 500 e6 * 15 / 1e5 = 75 e3 = 75,000 requests per second (lets make it 100k)
    - Peak is say 5x or 500k requests / second
4. We can assume that playlist reading will be more common than modifying playlist -- so the reads there might be the constraining factor.
5. However, for the "writes" to the Song DB (listen count, like count etc) 
    - Every listen needs to be logged (also important because we might pay back to the Artist based on accurate listen counts) -- so 500k / requests/second writing 
    - Worst case, we can assume every time a song is played, we get a "like"/ "do not like" signal - so 500k requests /second writing into Likes Database

Take aways:
 - Storage is not the bottleneck. The songs themselves will be in some blob storage, and the Songs DB is likely much smaller (2 to 3 GB)
 - Different parts of my system are dominated by Read and Write throughput (so depending on what we focus on, we might have to think about the design)
    - With 500k read/sec (assuming DB can do 10k read/sec -- we need 50 node DB cluster to serve reading)
    - We need 500 node DB cluster to serve writing


## Constraints
(Use this to list the explicit and implict constraints on the system)
1. Latency -- Users should be able to load their playlist and listen to songs instantly.
2. Consistency - Users should be able to fetch their latest playlists -- Song like / dislike statistics should be up-to-date
3. Users should have access to a near real time song listen count and get the top-K songs 


## Incentives & Trade-offs
(Use this to list the trade-offs you want to make based on the constraints, also some generic trade-offs)
1. Choose Availability -- I want to serve the user the most recent numbers about song like etc -even if it is not the "latest" number. 
2. Choose Availability -- Billing/ Tracking/ Data analytics on songs are not dependent on real time -- the "analytics warehouse" can be built after eventual consistency
3. We likely have to shard our DBs - not because of storage but because there is a potential for heavy reads and writes
    - Potentially have to think about better strategies to batch writing -seems fishy to have such large clusters, but might be okay (I genuinely do not know)

## Design 
(Overview of the design)

### Data
UserDB: user_id X playlist_id X song_id --> like_count, dislike_count, played_count 
  - Tracking like and dislike count separately -- I have surely liked and disliked the same song on Spotify
  - Primary key: User Id
  - Secondary key - Playlist Id 
  -  1e9 X 2e1 (20 playlist / user) X 5e1 (50 Songs / Playlsit)   = 1e12 Rows in the table 
  -  1e12 X 5e1 X 2 = 1e13  ~ 10TB (User DB)
  -  Can fit in one cluster, but logging the data might be the issue (see how I am thinking about MetricDB)
  -  Options:
       - 500 node shards, Sharded by {user_id} -- to manage the write throughput
       - MetricDB takes care of all the metrics -- and we display the user statistics from a Cache and this table gets updated periodically by a daemon (Batching the writes to the table). This way, the UserDB need not be sharded 
         (Leaning toward this solution as complexity of maintaining a sharded database is localized)
SongDB: 
 - We have identified that the SongDB is likely small (2 to 3 GB) -- so no sharding 
 - Similar to the UserDB, I think we can make the table bigger (to keep track of song level statistics if needed) but that can be done periodically by a daeomon 
 - Should we have to shard this table for Production -- I am leaning toward no, because by design the data on this table is "static", so we can serve from a cache that might still have to shared by {song_id}, but only 5 nodes (instead of 50) as cache can get throughput of uptp 100k/sec

(Separating the song metrics DB from the Song DB because the metrics is write heavy and songs is read heavy - Intuitively seems like the right thing to do because the pressures on the two DBs are different)
MetricsDB: This is the doozy one, because we need to keep this correct as well as serve the top_K request across many different buckets. I am leaning toward the design where we keep the Current Day's Metrics DB active, while the previous day etc are actually summarized 
(i.e., the full cardinality song X user id) is kept only for a few hours on disk, and when the daemon succesfully writes the data back to our primary UserDB and SongsDB, we can "reset" this database. This helps us to keep the DB size under-control (if needed). 

The top_K metrics can be served from Caches where we just pre-populate the top_K for a bunch of commonly used groupings of the data -- And the esoteric ones can be done by some creative querying between the MetricsDB and the SongDB as required. 

user_id X song_id X timestamp  -->is_played, is_liked, is_not_liked
- We have to write 500k/sec onto this -- so we really need 500 shards to make that possible
- Shard by time -- will not solve because all the writes go to the same one, song_id --> Could very well have a "hot song", so most natural is to shard by {user_id X song_id} (some combined hash)

We will also have CDN to make sure that we do not overwhelm anything here and to satisfy latency Gods.

### Plumbing
The user facing plumbing is 
 - fetch the entire Playlist that they select from the Cache onto the phone / laptop (check if BW is constraining, otherwise we can chunk it on the client side)
 - download the songs from the blob store (do we need pre-signed keys etc -- maybe can happen in the first step)
 - every time a song is being played - send a write request to the metrics service (at the end of the song)
 - Should be easy to add the AI stuff here - m10n etc -- are just calls made from the client when being used 

The top_K is also - when the write request to the metrics service is done -- just give a call to the SortedSets (the common ones that we maintain - last N minutes, last N days, last N minutes/Genre etc)  [I will need more practise on this -- From my last study, this was a much more harder problem]. The data is kept sorted, so we can just pick the top N. 

The hard engineering problems are punted to this deamon service that dumps the metrics from the MetricsDB back onto the UserDB and the SongDB and how to do that correctly. I think we need to have a last_updated in the UserDB and SongsDB. That tells us when the metrics got syncked and the deamon can look at the delta to syhnch back. There is probably some computation done to say which timestamp and earlier can be marked as safe to delete from the Metrics DB based on signals from the UserDB and SongDB 


### Failure Modes
 Too bored - let us discuss this


## Metrics, Measurements and Debugging
Too bored - let us discuss this