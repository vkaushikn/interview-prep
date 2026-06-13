# Merging a Sorted Stream into Sorted Lists Without Re-sorting

**The problem:** you maintain N sorted lists (e.g., each user's last-20-posts feed, newest-first). New items arrive that need inserting into potentially many of these lists at once (a single post fans out to many followers' feeds). Naively, each insertion either requires a full re-sort or a positional insert (O(list length)).

**The trick:** if the *incoming stream of new items* is itself processed in chronological (ascending time) order, then for each item, **push to the head of every affected list and pop the tail** — O(1) deque operations, no sort needed.

**Why it works:** processing oldest-to-newest means each pushed item is more recent than everything already in the list (the list only contains items older than the start of this batch). So the head, after all pushes for this batch, correctly holds the most recent item — by construction, not by sorting.

**Failure mode if you get the order wrong:** if you instead iterate over *posters* in arbitrary order (e.g., a `user → did_post` boolean dict, iterated in dictionary/ID order) and push each poster's latest post to their followers' heads, a follower with new posts from two different followees can end up with them in the wrong relative order — whichever poster was iterated last ends up on top, regardless of actual post time.

**Fix:** maintain a small chronological log of *individual posts* from the last batch window (poster_id, post_id, timestamp), process that log in time order, then head-push/tail-pop into each affected follower's feed.

First surfaced in: News Feed case study (daemon fan-out mechanics).
