# Read-Your-Own-Writes (RYOW) Consistency

**The textbook concept:** In a system with replication or caching, after a user writes data (e.g., posts a status), a subsequent read might be served by a stale replica or cache that hasn't yet absorbed the write — so the user might not immediately see their own post. RYOW is a consistency guarantee that a user always sees the effect of their own writes, even if the system is otherwise eventually-consistent for everyone else.

**OR/control analog:** A staleness/synchronization-window problem — similar to the lag between when an offline/hourly solve updates an operating curve and when the online controller picks up the new target (see the Azure case study). The write has happened, but the read path hasn't "seen" it yet because of a propagation delay (replication lag, cache TTL, etc.).

**Typical fixes (if it ever comes up):**
- Read-after-write from the primary/leader for a short window after a user's own write.
- Sticky sessions — route a user's reads to the same replica that served their write.
- Client-side optimistic UI — show the new post locally immediately, independent of what the backend read path returns.

**Status in this project:** Explicitly out of scope in the News Feed case study, by assumption — the design assumes the read path always sees the latest data. Good to have this ready to name if an interviewer probes the write path, even though the design itself doesn't address it.

First surfaced in: [News Feed case study](../Case%20Study/) (deferred by assumption).
