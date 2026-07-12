# Ride Sharing (Uber-like)

## Pattern

**Location-aware matching pipeline** — ephemeral driver state in Redis, durable ride state in DB, real-time bidirectional communication via WebSocket.

```
Driver App → WebSocket → Location Service → Redis Geo Index
Rider App  → WebSocket → Match Service → Redis Geo Index + Redis Lock → Ride DB → Kafka
                                       ↘ Notify Driver + Rider via WebSocket
```

---

## A — Assumptions

1. 1M active drivers globally
2. 500k ride requests per minute (~10k/sec)
3. Driver sends GPS ping every 5 seconds
4. Matching radius: 5 miles
5. Surge pricing based on demand/supply per city block
6. Payment and payout handled by separate service

---

## C — Constraints

| Constraint | Active? | Notes |
|------------|---------|-------|
| Driver location must be fresh | Yes | Stale location = wrong match |
| One driver matched to one rider at a time | Yes | Atomic claim required |
| Ride state must be durable | Yes | Financial record |
| Match notification must be real-time | Yes | WebSocket, not FCM |
| Driver location history | No | Only latest location needed |

---

## Approximations

- Driver location writes: 1M drivers ÷ 5s = **200k writes/sec** → Redis only, MySQL dies
- Ride requests: 500k/min = **~10k/sec**
- Ride DB writes: 10k/sec → manageable for MySQL/Postgres with connection pool
- Redis geo index: 1M drivers × ~100 bytes = ~100MB — fits in one Redis instance

---

## I — Incentives / Trade-offs

- **Redis over MySQL for location** — 200k writes/sec kills MySQL. Redis handles 1M ops/sec. Driver location is ephemeral — Redis going down is fine, drivers re-ping within 5 seconds.
- **WebSocket over FCM for both driver and rider** — apps are open and active during a ride. WebSocket gives bidirectional real-time comms (driver sends location, rider sends cancel, driver sends arrived). FCM is for background notifications only.
- **Atomic Redis lock for matching** — prevents two riders claiming the same driver simultaneously. `SET NX` is the primitive.
- **Kafka for event stream** — every state transition (matched, accepted, started, completed) published to Kafka. Analytics, billing, dispute resolution consume independently. Nothing lost if downstream services go down.

---

## D — Design

### Driver Location Updates

```
Driver App (GPS ping every 5s)
    → WebSocket connection to Location Service
        → GEOADD driver_id lat lng → Redis Geo Index
        → also publish to Kafka topic: driver_locations (for analytics/surge)
```

Driver location is **ephemeral** — only latest ping matters. No persistent storage. Redis TTL on each driver entry: 30 seconds (auto-expires if driver goes offline).

---

### Ride Matching

```
Rider requests ride (pickup GPS)
    → Match Service
        1. GEORADIUS pickup_lat pickup_lng 5 miles → list of nearby driver_ids
        2. Filter out unavailable drivers (check Redis status store)
        3. For nearest available driver:
               SET driver:{id}:status "matched" NX EX 60
               → if OK: claim succeeded → proceed
               → if nil: another rider got there first → try next driver
        4. Write Ride record to Ride DB (PENDING state)
        5. Publish RIDE_MATCHED event to Kafka
        6. Notify driver via WebSocket: rider details, pickup location
        7. Notify rider via WebSocket: driver details, ETA
```

**The atomic claim** — `SET NX` (set if not exists) ensures only one rider can claim a driver. Two riders racing: one gets OK, one gets nil. The one that gets nil moves to the next nearest driver. No double-booking possible.

---

### Driver Response

```
Driver accepts:
    → Update Redis: driver:{id}:status = "en_route"
    → Update Ride DB: status = ACCEPTED
    → Publish RIDE_ACCEPTED to Kafka
    → Notify rider via WebSocket: driver accepted, live location updates begin

Driver rejects / no response within 30s:
    → Redis TTL expires automatically (EX 60 on the lock)
    → Match Service retries with next available driver
    → Notify rider: "Finding another driver..."
```

---

### Active Ride

```
Driver sends GPS ping every 5s
    → Location Service updates Redis Geo Index
    → also pushes location to rider's WebSocket connection directly
         (Match Service maintains driver_id → rider WebSocket mapping)

Rider arrives at destination:
    → Driver marks ride complete in app
    → Update Ride DB: status = COMPLETED, dropoff_gps, end_time
    → Publish RIDE_COMPLETED to Kafka
    → Billing service consumes from Kafka → charges rider, queues driver payout
```

---

### Surge Pricing

- City divided into hexagonal cells (H3 library, ~500m radius each)
- Every driver ping updates cell supply count
- Every ride request updates cell demand count
- Surge multiplier = f(demand / supply) per cell
- Computed in real-time from Kafka driver_locations and ride_requests streams
- Surge price shown to rider before they confirm — baked into Ride DB at creation time

---

### Ride DB Schema

```
rides:
    ride_id         UUID primary key
    rider_id        FK
    driver_id       FK
    status          ENUM(PENDING, ACCEPTED, EN_ROUTE, COMPLETED, CANCELLED)
    pickup_gps      point
    dropoff_gps     point
    surge_multiplier float
    base_cost       decimal
    final_cost      decimal
    created_at      timestamp
    completed_at    timestamp
```

---

### Redis Data Structures

```
Geo index:     GEOADD drivers {lat} {lng} {driver_id}
               GEORADIUS drivers {lat} {lng} 5 mi → nearby drivers

Status store:  SET driver:{id}:status "matched" NX EX 60
               GET driver:{id}:status

WebSocket map: driver:{id}:ws_server → which WebSocket server holds this connection
               (used to route notifications to the right server)
```

---

## Failure Modes

**Redis goes down:**
- Driver locations lost → drivers re-ping within 5s, rebuilt automatically
- In-flight match locks lost → Ride DB is source of truth, reconcile on restart
- Mitigation: Redis Sentinel or Redis Cluster for HA

**Match Service goes down:**
- Active WebSocket connections drop → clients reconnect, re-request ride
- Ride DB preserves PENDING rides → Match Service resumes on restart

**WebSocket server goes down:**
- Clients reconnect to new server
- New server looks up `driver:{id}:ws_server` in Redis to re-establish routing
- Brief notification gap — acceptable

**Driver goes offline mid-ride:**
- GPS pings stop → rider sees "location unavailable"
- Ride DB record preserved — billing uses last known state
- Driver reconnects → resumes pings

---

## Key Numbers

| Metric | Value |
|--------|-------|
| Active drivers | 1M |
| Driver location writes/sec | 200k |
| Ride requests/sec | ~10k |
| Redis ops/sec capacity | ~1M (single instance) |
| WebSocket connections per server | ~100k |
| WebSocket servers needed | ~10M riders ÷ 100k = 100 servers |
