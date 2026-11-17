# ADR-0002: Use PostgreSQL as the persistent system of record

**Date:** 2026-11-10

## Status

Accepted

## Context

Orders, reservations, payment attempts and sale configuration must survive restarts and be auditable after the sale (DB-2, FR-18). The stock claim in the baseline needs safe transactional behaviour (DB-4), and stock must never be negative (DB-1). Even after the hot stock moves to an in-memory layer (ADR-0007), a durable record is required for reconciliation.

Options considered:

| Option | Assessment |
|---|---|
| **PostgreSQL** | ACID transactions, row-level locking, `CHECK` constraints, `INSERT … ON CONFLICT` for idempotency, good tooling for lock and connection metrics (`pg_stat_activity`, `pg_locks`) |
| MySQL | Also suitable; no clear advantage for this project |
| NoSQL store only | Weaker multi-row transactions; harder to prove "no oversell" |

## Decision

PostgreSQL is the persistent system of record for sale, inventory, reservation, order, payment attempt and request-identifier data (C-2, SI-1).

Database rules:

- `inventory.available` has `CHECK (available >= 0)` (DB-1).
- The request identifier is a primary key or unique column, so duplicates are detectable (DB-3).
- In the improved architecture each service owns its own schema; services do not read each other's tables.

## Consequences

- Baseline correctness can rely on PostgreSQL transactions and constraints.
- PostgreSQL statistics provide the connection-usage and lock-wait measurements required by FR-16.
- PostgreSQL remains a single write point; this is why the hot stock state is moved to memory during the burst (ADR-0007) while PostgreSQL stays the durable record.
