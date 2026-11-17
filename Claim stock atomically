# ADR-0006: Claim stock atomically

**Date:** 2026-11-10

## Status

Accepted. Applies to the baseline (in PostgreSQL) and to the target (in the in-memory layer, ADR-0007).

## Context

With one unit left, Buyer A and Buyer B both read `available = 1`, both pass the check and both decrement. Stock becomes −1 and two orders are confirmed for one item (B2). The cause is that *check* and *claim* are two separate steps that concurrent requests can interleave.

Options considered:

| Option | Assessment |
|---|---|
| `SELECT … FOR UPDATE`, then `UPDATE` | Correct, but holds a row lock across two statements and increases contention (B1) |
| Optimistic locking with a version column and retry | Under heavy concurrency most attempts fail and retry, adding load (B5) |
| In-JVM `synchronized` | Works only within one application instance |
| **Single conditional decrement** | One indivisible operation; no read-then-write gap; no retries |

## Decision

Stock is claimed in one atomic operation that succeeds only if stock is available (FR-5, FR-6):

**Baseline (PostgreSQL):**

```sql
UPDATE inventory
SET    available = available - 1
WHERE  sale_id = :saleId AND available > 0;
-- 1 row updated → reserved, 0 rows → sold out
```

The reservation row is inserted in the same short transaction. `CHECK (available >= 0)` remains as a safety net (DB-1). The per-buyer limit is enforced in the same operation or by a unique constraint on `(sale_id, user_id)` (FR-7, BR-2).

**Target (in-memory layer):** the same rule — check limit, check stock > 0, decrement, record buyer — runs as one atomic operation in the in-memory component (ADR-0007).

**Release:** a unit is returned only if the reservation is still active, so it is returned exactly once (SAF-3):

```sql
UPDATE reservation SET status = 'RELEASED' WHERE id = :id AND status = 'ACTIVE';
-- only if 1 row changed: return the unit to stock
```

## Consequences

- Overselling is prevented regardless of the number of instances (PR-1, SAF-1), verified by T-2.
- No retry traffic is created by lost races.
- All claims still meet at one stock value, so contention (B1) is handled separately by ADR-0005 and ADR-0007.
