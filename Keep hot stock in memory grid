# ADR-0007: Keep hot stock in a space-based in-memory layer

**Date:** 2026-11-10

## Status

Accepted. Concrete technology is decided in [ADR-0013](choose-in-memory-data-grid-technology.md).

## Context

Even with an admission gate and an atomic claim, every claim in the baseline updates the same PostgreSQL stock row and needs a database connection. Under burst load this row is expected to be the main point of contention (B1) and a driver of connection usage (B3). Space-Based Architecture targets exactly this situation: remove the central database from the high-concurrency transactional path and keep the hot data in memory, writing to the database asynchronously.

Options considered:

| Option | Assessment |
|---|---|
| Keep stock in PostgreSQL only | Simple and durable; contention remains |
| Split stock into several rows (sharding) | Less contention; "last unit" and FIFO handling become complex |
| **Hot stock state in a distributed in-memory component** | Microsecond atomic operations; no DB connection on the claim path |

## Decision

Apply the space-based pattern only to the **hot flash-sale inventory state** (SI-2, C-7):

1. **Warm-up.** Before the sale starts, the initial quantity is loaded from PostgreSQL into the in-memory layer.
2. **Claim in memory.** The Inventory Service performs the atomic claim (ADR-0006) against the in-memory stock. No PostgreSQL access is needed to accept or reject a claim, which also supports early sold-out answers (FR-4).
3. **Single owner per stock key.** Each sale's stock value has one primary owner (replicas only for failover), so replicas cannot apply conflicting updates.
4. **Asynchronous persistence.** Each successful claim produces a `StockReserved` event; reservations and orders are written to PostgreSQL by the services that consume it (ADR-0009). PostgreSQL stays the durable system of record (ADR-0002).
5. **Release.** Failed or expired reservations return the unit to the in-memory stock, exactly once (BR-4, SAF-3).
6. **Reconciliation.** After each test run, in-memory totals, PostgreSQL totals and initial stock are compared (FR-18).

## Consequences

- The shared stock row is no longer touched by every request, reducing lock waits (B1) and DB connection usage (B3). This is measured by comparing T-1 against the baseline.
- New risk: data held in memory can be lost on failure. Mitigation: replication and persistence settings of the chosen product (ADR-0013), durable events, and reconciliation.
- Only small hot data (stock, tickets, request identifiers) lives in memory; everything else stays in PostgreSQL.
- Failover and burst tests are harder to reproduce locally (lower testability).
