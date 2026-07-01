# Data Center Metrics Brain

## Problem

10,000 servers, each emitting 2,000 metrics/sec (CPU, memory, disk, network, …). Build a central system that ingests these and serves a dashboard answering:

1. Given a `server_id`, return min/max/avg of all metrics within a 2-day time window at 1-minute granularity.
2. Given a `metric_id`, return min/max/avg of all servers within a 2-day window at 1-minute granularity.
3. Given a `server_id` and a time range, return min/max/avg of all metrics.
4. Given a `metric_id` and a time range, return min/max/avg of all servers.

Data retention: 1 year.

## Why it is interesting

- 10,000 × 2,000 = 20M events/sec ingest — pure write throughput problem
- Two access patterns with orthogonal primary keys (server_id vs metric_id) — schema and sharding choices matter
- 1-minute granularity over 2 days = 2,880 buckets — pre-aggregation vs raw storage tradeoff
- 1 year retention — tiered storage likely needed
- Connects to: batch aggregation daemon, time-bucketed writes, scatter-gather reads, stream processing
