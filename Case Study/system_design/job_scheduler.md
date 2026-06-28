# Job Scheduler: Canonical Design Reference

## A — Assumptions & Approximations

### Assumptions (scope)
- **Task:** reusable definition — (function code, arguments, runtime environment, cron schedule, timeout)
- **Job:** one scheduled instance of a task at a specific time
- FaaS execution model — each job runs in an isolated container
- Jobs have a configurable timeout; system enforces a hard maximum
- At-least-once execution with idempotency expected from callers (exactly-once is a function of the caller's code, not the scheduler)
- Access control and geography out of scope

### Approximations
| Quantity | Estimate |
|---|---|
| Jobs per day | 10M |
| Avg throughput | 10M / 1e5 ≈ **100 jobs/sec** |
| Peak (thundering herd at midnight/top-of-hour) | **~1,000 jobs/sec** |
| Tasks (unique definitions) | 1M |
| Job status row size | ~100 bytes |
| Function + args in blob storage | varies, treat as opaque |

**Key constraint:** jobs are not uniformly distributed. Many are scheduled at midnight or top-of-hour — the scheduler must absorb 10x spikes without dropping jobs.

---

## C — Constraints

- **Timeliness:** jobs must start within seconds of their scheduled time (low jitter)
- **No double execution:** a job must not run more than once (even if a worker crashes mid-execution)
- **Timeout enforcement:** jobs running beyond their configured timeout must be killed and marked failed
- **Observability:** job status (pending / running / done / failed) must be queryable at any time
- **Durability:** a scheduler crash must not lose pending jobs

---

## I — Incentives & Trade-offs

1. **Pre-populate jobs vs. compute on demand.** When a task is created, pre-populate all future job rows into JobDB for the next N days. Alternatively, the scheduler derives upcoming jobs from cron expressions on every tick. Pre-population decouples task authoring from scheduling math, makes the hot path simple (just a DB read), and handles task modifications cleanly — but requires a background job to extend the horizon as time passes.

2. **Redis sorted set as execution buffer vs. polling JobDB directly.** Workers could poll JobDB for due jobs, but DB polling at 1k jobs/sec from many workers creates read contention. A Redis sorted set (score = scheduled timestamp) lets workers atomically claim jobs via `ZPOPMIN` — single-threaded Redis ensures only one worker gets each job, no locking needed.

3. **At-least-once vs. exactly-once.** True exactly-once across a crash boundary requires distributed transactions. Practical answer: at-least-once (claim + watchdog requeue on crash) with idempotent job functions. The scheduler's job is to ensure every job runs at least once; the function's job is to be safe to retry.

4. **Warm container pool vs. cold start.** Spinning up a fresh container per job adds seconds of latency. A warm pool of pre-initialized containers trades idle cost for low start latency. Pool sizing is a capacity tuning knob (see [capacity_benchmarks](../../../concepts/approximations/capacity_benchmarks.md)).

---

## D — Decisions

### D1. Data model

**TaskDB** (read-heavy, small):
| Column | Notes |
|---|---|
| task_id (PK) | |
| function_url | pointer to blob storage |
| args_url | pointer to blob storage |
| cron_expr | e.g. `0 * * * *` |
| timeout_sec | per-task max runtime |
| max_retries | |
| is_active | soft delete |

**JobDB** (write-heavy, large):
| Column | Notes |
|---|---|
| job_id (PK) | |
| task_id (FK) | |
| scheduled_at | indexed — primary access pattern for scheduler |
| status | pending / running / done / failed — indexed |
| started_at | set by worker on claim |
| completed_at | |
| worker_id | which worker claimed this job |
| retry_count | |

Index on `(scheduled_at, status)` — scheduler query: *"give me all pending jobs due in the next 5 minutes."*

### D2. Three-component architecture

```
TaskDB / JobDB
     ↓
  Scheduler  →  Redis Sorted Set  →  Workers (Executor pool)
                                          ↓
                                    Container pool
                                          ↓
                                    JobDB status updates + MetricsDB
     ↑
  Watchdog (runs independently)
```

### D3. Scheduler

Runs on a tick (e.g. every 30 seconds). Queries JobDB for jobs where `scheduled_at ≤ now + 5min AND status = pending`. Pushes job_ids into a Redis sorted set with score = `scheduled_at` (Unix timestamp).

**Pre-population:** when a task is created or modified, a background worker expands the cron expression and writes all job rows for the next N days into JobDB. The scheduler then just reads — no cron math at tick time. Hot path: if a new task fires within the current 5-minute window, it is added to both JobDB and the Redis sorted set immediately.

**Leader election:** run two scheduler instances (active + standby) with a distributed lock (Redis `SET NX` with TTL). Standby takes over if the active misses its heartbeat. Jobs already in the sorted set continue to be processed — no work is lost.

### D4. Worker (Executor)

Multiple workers run `ZPOPMIN sorted_set` — atomically pops the job with the lowest score (earliest scheduled time) where score ≤ now. Redis single-threaded model guarantees only one worker gets each job.

On claiming a job:
1. Write `status = running, started_at = now, worker_id = self` to JobDB
2. Fetch function code + args from blob storage (TaskDB pointer)
3. Allocate a container from the warm pool (or cold-start one)
4. Execute with timeout enforcement (kill process at timeout, mark failed)
5. On completion: write `status = done/failed, completed_at`, emit logs + metrics to MetricsDB

Workers are stateless — scale horizontally to handle peak throughput.

### D5. Watchdog

Runs on a separate tick (e.g. every minute). Two scans:

1. **Stuck running jobs:** `status = running AND started_at < now - timeout_sec`. Worker died mid-execution. Mark failed, re-enqueue if `retry_count < max_retries`.
2. **Lost claims:** job was popped from Redis (`ZPOPMIN`) but `status` was never updated to `running` (worker crashed between pop and DB write). Detected as: job not in Redis sorted set, not in running state, `scheduled_at` is in the past. Re-enqueue.

This is what gives at-least-once semantics after a crash.

### D6. Thundering herd mitigation

Many jobs fire at midnight → 1k ZPOPMIN/sec against Redis. Two mitigations:
- Workers are already parallel — natural spreading as the pool drains the sorted set
- For non-critical jobs: add small random jitter (±30s) to `scheduled_at` at pre-population time, breaking the exact-midnight spike

---

## Failure Modes

| Failure | Impact | Recovery |
|---|---|---|
| Scheduler crashes | No new jobs pushed to Redis | Standby takes over via leader election; existing sorted set continues to drain |
| Worker crashes mid-job | Job stuck in `running` | Watchdog detects timeout, re-enqueues |
| Worker crashes after ZPOPMIN, before DB write | Job silently lost | Watchdog detects via lost-claim scan, re-enqueues |
| Redis crashes | Workers can't claim new jobs | Scheduler re-reads JobDB and repopulates sorted set on Redis recovery |
| JobDB slow | Status updates lag | Execution continues; status is eventually consistent — acceptable |
