# Ticketmaster — System Design

## A — Assumptions & Approximations

**Scope:**
- Users browse events and seat maps
- Reserve a seat, complete payment, receive ticket
- No double booking under any condition
- Flash sale scenario: popular event, massive concurrent traffic at sale open

**Scale:**
- 50k seats per large event
- 500k users attempting to buy simultaneously at sale open
- Reads (browsing) >> Writes (purchasing) — 100:1 ratio
- Peak write load concentrated in first 60 seconds of sale open
- Seat map: ~50k rows per event, small — fits in memory easily

---

## C — Constraints

1. **No double booking** — hard constraint. Overselling a seat is unacceptable.
2. **High availability** — sale opening is a known high-traffic event, must not go down.
3. **Reservation window** — seat held for 10 minutes while user completes payment. Must be released if payment not completed.
4. **Flash sale load** — 500k concurrent users is far beyond what a naive DB write path can handle.
5. **Payment atomicity** — inventory decrement and payment confirmation must be consistent. Payment failure must release the seat.

---

## I — Incentives & Trade-offs

**Speed vs. Consistency:**
- Browsing can be slightly stale (cached seat map is fine — user discovers unavailability at reservation time)
- Reservation and payment must be strongly consistent — no approximations

**Pessimistic lock vs. Reservation pattern:**
- `SELECT FOR UPDATE` on seat row is simple but holds lock for 30-60 seconds while user enters payment — kills throughput under flash sale load
- Reservation record with expiry releases the lock to milliseconds at DB level, holds "logical lock" in reservation table
- Reservation pattern wins at scale

**Thundering herd vs. Metered access:**
- Letting all 500k users hit the reservation service simultaneously → DB overwhelmed
- Virtual waiting room meters users through at controlled rate — DB sees manageable write load
- Trade-off: user experience (queue screen) vs. system stability

---

## D — Decisions

### Data Model
```
seats(seat_id, event_id, status)          -- AVAILABLE | RESERVED | SOLD
reservations(reservation_id, seat_id, user_id, expires_at, status)
  UNIQUE(seat_id) WHERE status = 'RESERVED'  -- DB enforces no double reservation
payments(payment_id, reservation_id, status, amount)
```

### Components

**1. Seat map service (read path)**
- Seat availability served from cache (Redis)
- Cache invalidated on reservation/sale events
- CDN for static event content
- Slight staleness acceptable — user finds out at reservation time

**2. Virtual waiting room (flash sale)**
- 500k users hit "buy" → all enqueued into Kafka topic
- Reservation service consumes at controlled rate (e.g. 1k/sec — tuned to DB write capacity)
- Kafka offset = queue position → "you're number 4,832 in line" is just the user's offset
- If reservation service goes down, messages stay in Kafka — no lost requests
- Consumer rate is a dial: tune up/down based on DB load. Natural backpressure.
- Important: queue only guarantees a *chance* to buy — seat may be gone by the time user reaches front. Show "seat unavailable, pick another" — this is expected behavior.

**3. Reservation service**
- User selects seat → insert reservation record
- UNIQUE constraint on `(seat_id, status=RESERVED)` — DB enforces exactly-once
- Concurrent requests race at DB level — one wins, rest get constraint violation → "seat unavailable"
- Reservation expires_at = now + 10 min

**4. Background expiry job**
- Scans reservations where `expires_at < now AND status = RESERVED`
- Sets status → AVAILABLE, invalidates cache
- Runs every 30 seconds — acceptable lag

**5. Payment service**
- User submits payment while seat is RESERVED
- Payment processor confirms → reservation CONFIRMED, seat SOLD (atomic DB transaction)
- Payment fails → reservation RELEASED, seat AVAILABLE
- Idempotency key on payment to prevent double charge on retry

### Failure Modes
- **Seat reserved, payment in-flight, service crashes** → saga pattern. Payment processor webhook confirms/denies async. On confirm → mark SOLD. On deny/timeout → release reservation.
- **Expiry job misses a reservation** → lazy check at purchase time: if reservation expired, reject even if status still shows RESERVED
- **Cache stale after reservation** → acceptable. User sees seat as available, tries to reserve, gets constraint violation. Shows "seat just taken."

### Deep Dives
- **Double booking** → unique constraint + reservation pattern (not application-level lock)
- **Flash sale** → virtual waiting room meters load
- **Payment failure** → saga / compensating transaction releases seat
- **Expiry** → background job + lazy check at purchase time
- **Read scale** → cache + CDN, invalidate on status change
