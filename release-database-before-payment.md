# ADR-0008: Release the database before payment

**Date:** 2026-11-10

## Status

Accepted. The reservation timeout value is still open (SRS Appendix C, item 4).

## Context

In the baseline, the payment call happens inside the purchase transaction. The payment service is external and may be slow (C-3). While it responds, the database connection stays checked out. With a fixed pool, only a few purchases can be in progress; new buyers cannot even start (B3), and the whole purchase path waits on payment (B4).

Options considered:

| Option | Assessment |
|---|---|
| Larger connection pool | PostgreSQL has a connection limit; more connections also mean more contention |
| Shorter payment timeout | Real payments still take time; more false failures |
| **Short transaction for reservation and order, then payment outside any transaction** | Database is busy only for the short step |

## Decision

No database transaction stays open while waiting for the payment service (FR-10):

1. **Reserve** — atomic claim (ADR-0006) and creation of a reservation with an expiry time.
2. **Create order** — a `PENDING` order is created after successful reservation (FR-9); the transaction commits and the connection is released.
3. **Pay** — payment is requested asynchronously (ADR-0009) using the order ID as the payment idempotency key (ADR-0011).
4. **Confirm** — on success the order becomes `CONFIRMED` (FR-11).
5. **Release** — on failure, timeout, or reservation expiry the order becomes `FAILED` and the unit returns to stock (FR-8, FR-12, BR-4, SAF-3).

The buyer receives a status straight after the reservation step and follows progress through the status endpoint (CI-3, UI-2).

## Consequences

- DB connection time per purchase drops to the short reservation/order step; measured with FR-16 during T-3.
- Stock is not lost to abandoned or stuck payments.
- The buyer experience becomes two-phase: *reserved*, then *confirmed* or *failed*.
- A payment success that arrives after the reservation expired must be handled (refund and log); this edge case is covered in tests.
