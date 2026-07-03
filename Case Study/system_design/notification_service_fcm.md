# Notification Service / FCM Architecture

## The Problem
Deliver push notifications to billions of devices reliably, even when devices are offline.

## A — Assumptions & Approximations

- ~3B Android devices globally, ~1B connected at peak
- Each edge server holds ~100k persistent TCP connections
- ~10,000 edge servers globally across ~100 PoPs
- ~150k notification deliveries/sec globally (rough estimate)
- Heartbeat every 15-30 min (OS-managed, battery optimized)

---

## C — Constraints

1. At-least-once delivery — missing a notification is worse than a duplicate
2. Device may be offline — must queue and deliver on reconnect
3. Multiple devices per user — fan-out to all device tokens
4. Edge server may fail — must recover without losing pending notifications
5. TTL on queued notifications — don't hold forever (typically 4 weeks)

---

## I — Trade-offs

- **Bigtable over MySQL** — 150k ACK writes/sec far exceeds MySQL's ~1k writes/sec. LSM tree + horizontal sharding handles this. Trade-off: no ACID transactions, no JOINs.
- **Redis for hot pending queue** — fast O(1) lookup on reconnect vs slow Bigtable scan. Trade-off: Redis can lose data — Bigtable is the durable source of truth.
- **At-least-once not exactly-once** — simpler delivery guarantee. App deduplicates using notification ID.

---

## D — Design

### Core Flow

```
3P App Server
    → FCM API
        → Write to Bigtable (persist first, always)
        → Fan-out to each device token for this user
            → Device Registry lookup (token → edge server ID)
            
            IF ONLINE:
                → Push to edge server's in-memory queue for that device
                → Edge server pushes down TCP connection
                → Device ACKs
                → Write ACK to Bigtable

            IF OFFLINE:
                → Write to Redis pending queue for that device token
                → On reconnect → edge server checks Redis → drains queue
                → Device ACKs
                → Write ACK to Bigtable + clear Redis entry
```

### Key Components

**Device Registry**
- `device_token → edge_server_id`
- Updated every time device connects
- Consistent hashing — same device routes to same edge server
- Entry removed / nulled when device disconnects

**Bigtable Schema**
- Key: `device_token + timestamp`
- Columns: `notification_id, payload, ack_status, expires_at`
- Co-locates all notifications for a device (range scan by token prefix)
- Source of truth — written before any delivery attempt

**Redis Pending Queue**
- Key: `device_token`
- Value: list of undelivered notification IDs
- Hot path for offline → online reconnect
- Ephemeral — can be rebuilt from Bigtable if lost

**Edge Server**
- Holds persistent TCP connections for ~100k devices
- Maintains small in-memory queue per connected device
- Drains Redis on device reconnect
- Sends heartbeat ACKs back to device registry to confirm liveness

---

## Failure Modes

**Edge server fails:**
- Devices reconnect to new edge server (device registry updated)
- New edge server checks Redis for pending notifications → drains
- For gap period while rebuilding: query Bigtable `(token_id, timerange, ack=false)`
- New notifications during rebuild go to Redis automatically

**Redis fails:**
- Rebuild pending queue from Bigtable: filter `ack_status = PENDING` per device token
- Slightly slower reconnect experience, no data loss

**Device dies before ACKing:**
- FCM retries unACKed notifications on next heartbeat automatically
- If device never comes back (uninstalled, lost) → daemon cleans up Bigtable entries past TTL
- Accept occasional missed notification — not catastrophic

**Duplicate delivery (at-least-once):**
- Each notification has a unique ID
- App deduplicates — if notification ID already seen, discard silently

---

## Multi-Device Fan-out

User has N devices → N device tokens stored in backend:
`user_id → [token_device1, token_device2, token_device3]`

FCM fans out to each token independently. Each token has its own:
- Edge server connection
- Redis pending queue entry
- Bigtable ACK record

---

## Key Numbers to Know
- 100k TCP connections per edge server
- ~10k edge servers globally
- Heartbeat: 15-30 min interval
- Notification TTL: ~4 weeks
- Bigtable: millions of writes/sec (LSM tree, horizontally sharded)
- Redis: sub-millisecond lookup for pending queue
