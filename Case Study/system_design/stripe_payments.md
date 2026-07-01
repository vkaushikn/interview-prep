# Stripe / Payment Systems — System Design

## Core Problem
Process payments exactly once. Network retries mean the same request can arrive multiple times — charging a user twice is catastrophic.

## Key Concepts

### Idempotency Key
- Client generates a UUID per payment attempt, sends with every request
- Server stores `(idempotency_key → result)` 
- If key seen before → return cached result, do not recharge
- Scoped to `(user, operation)` — not just operation, otherwise users collide
- TTL on key: keep too long → storage blows up. Delete too soon → retry after expiry double-charges. Typically 24-48 hours.
- In-flight race: two requests with same key arrive simultaneously → need distributed lock or DB unique constraint on key to prevent both proceeding

### Double-Entry Bookkeeping
Every transaction creates two ledger entries summing to zero:
```
debit  account_A  -$100
credit account_B  +$100
# sum = 0, always
```

**Rule**: never write one side and wait. Write both atomically in one DB transaction.

### Webhook Pattern
- Never hold DB connection waiting for payment processor (kills connection pool)
- Flow: payment processor confirms → sends webhook → you receive event → open DB connection → write both ledger entries atomically → close connection → acknowledge webhook
- All waiting happens outside the DB
- Idempotency key on webhook too — processor may send twice on retry

### DB Connection Pool Constraint
- Typical pool: 100-500 connections
- Each write: ~0.5ms → 500 connections = ~1k writes/sec
- Holding connection for external wait (60 sec) → 500/60 = ~8 writes/sec (100x collapse)
- Rule: hold DB connection only for the duration of the write, nothing else

## Failure Modes

**Payment confirmed, DB write fails:**
- Webhook arrives again (processor retries) → idempotency key catches it → safe

**Partial ledger write:**
- One side written, crash before second → DB transaction rollback → neither side written
- On retry, both sides written together → consistent

**Double webhook:**
- Idempotency key + `UNIQUE(payment_id)` on ledger → second webhook hits constraint violation → ignored

## vs. Ticketmaster
Stripe has fungible inventory (money) — just decrement/increment counts.
Ticketmaster has unique inventory (seats) — need row-level reservation.
Both need idempotency. Both avoid holding DB connections during external waits.
