# Keep hot stock in an in-memory grid

**Date:** 2026-10-26
**Step:** 7 · **Pattern:** Space-Based · **Problem:** P1 Database jam · **Requirements:** FR-6, FR-7, FR-24, PR-2, PR-3, PR-4

## Status

Accepted. Amends [Claim stock atomically](claim-stock-atomically.md): the atomic rule is kept, its location moves from SQL to Redis.

## Context

Even with the gate and microservices, every stock claim is a row-level update on the same DB record and needs a DB connection. Under burst tests this is the last measurable bottleneck: claim latency is dominated by lock waits, and the DB remains a single point that limits scaling. The space-based pattern addresses exactly this: remove the central database from the transactional path.

Options considered:

| Option | Assessment |
|---|---|
| Stock sharding (split 100 units into 10 rows of 10) | Reduces contention; complicates "last unit" handling and FIFO across shards |
| Hazelcast IMap with `EntryProcessor` | Java-native data grid, true processing-unit model; extra cluster to operate |
| **Redis with an atomic Lua script** | Single-threaded per key → atomic; already used for tickets and idempotency keys; microsecond operations |

## Decision

Apply the space-based pattern to the **hot path only** (Gate + stock claim):

1. **Warm-up:** before the sale, load `stock:{saleId}` from the DB into Redis; scale processing units (Gate instances) ahead of 12:00.
2. **Claim in memory** with one Lua script that checks the per-user limit, checks stock > 0, decrements, and records the buyer — all atomically. No DB access, no DB connection.
3. **Single owner per key:** the stock and ticket keys live on one Redis primary (replicas only for failover), so there are no replication data collisions and FIFO ticket order stays consistent.
4. **Write-behind:** the claim publishes `StockReserved`; a data writer persists reservations/orders to the DB at a steady rate. The DB stays the system of record.
5. **Release:** `StockReleased` → guarded `INCR` (once per reservation).
6. **Durability:** Redis replication + AOF (`appendfsync everysec`); events in RabbitMQ are durable; reconciliation job after the sale (FR-24) compares grid totals, DB totals and initial stock.

## Consequences

- Claim latency drops from DB lock time (ms, growing with load) to Redis operation time (sub-ms); the DB is off the request path, so adding Gate instances now actually increases capacity.
- Oversell protection is preserved (atomic script) and re-verified by T-2.
- A Redis crash could lose up to ~1 s of claims; mitigated by replicas, durable events and reconciliation, and accepted as a documented risk.
- Realistic tests of failover and burst need extra infrastructure (testability is lower).
- Only small, hot data lives in memory; everything else stays relational.
