# Fan-Out vs. Fan-In

Two strategies for propagating events to consumers in a distributed system.

```
Fan-out (write time)            Fan-in (read time)

event                           source 1 ──┐
  ├── consumer 1                           ├── aggregate → answer
  ├── consumer 2                source 2 ──┤
  └── consumer 3                source 3 ──┘
```

**Fan-out:** one event → write to many places immediately. Reads are O(1) — fetch from precomputed cache.

**Fan-in:** one event → one write (your own cache only). Reads query multiple sources and aggregate on demand.

## The decision

Write amplification = **event frequency × follower count**. Whichever side of this is cheaper wins.

| | Low follower count | High follower count |
|---|---|---|
| **Infrequent events** | Either works | Fan-out is okay (normal user posts) |
| **Frequent events** | Fan-in on read (friends top-K) | Fan-in on read (celebrity feed, song plays) |

- Fan-out wins when events are infrequent and reads are frequent — pre-computation pays off.
- Fan-in wins when events are frequent or follower count is very high — write amplification cost exceeds read-time aggregation cost.
- Fan-in also requires that read-time aggregation is cheap — bounded by follower count (F=100 is trivial, F=10M is not).

## The hybrid

Used when follower count is bimodal (normal users + celebrities):

1. Fan-out all normal user events to followers' caches at write time
2. At read time, fan-in celebrity posts live and merge with precomputed cache

The merge is cheap — both are sorted chronologically, so it is a simple merge of two sorted lists (see [[merge_sorted_stream_no_resort]]).

## Where this pattern appears

| System | Event | Fan-out or Fan-in | Why |
|---|---|---|---|
| News feed (normal users) | Post | Fan-out | Posts infrequent, reads frequent |
| News feed (celebrities) | Post | Fan-in at read | Follower count too high |
| Friends top-K songs | Song play | Fan-in at read | Plays too frequent (15/day × 100 friends) |
| Notifications (one recipient) | Like/comment | Fan-out | Trivial — one write |
| Notify all followers of live event | Go live | Hybrid | Depends on follower count |
| Game leaderboard (friends) | Score update | Fan-in at read | Fetch 100 scores in parallel, sort in memory |

First surfaced in: News Feed case study (fan-out for normal users, hybrid for celebrities). Revisited in Spotify case study (friends top-K as fan-in).
