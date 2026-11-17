# ADR-0011: Use idempotency keys

**Date:** 2026-11-10

## Status

Accepted

## Context

When the system is slow, buyers click again and clients retry (B5). A repeated request can create a second reservation, a second order, or a second charge, and every repeat adds load. Retries cannot be prevented completely, so they must be made harmless (FR-13, FR-14, PR-2, SAF-2, BR-5).

Options considered:

| Option | Assessment |
|---|---|
| Disable the Buy button only | Helps humans (UI-3) but not network retries, refreshes or scripts |
| Deduplicate by user ID only | Blocks a legitimate new attempt after a failed payment |
| **Unique request identifier per logical purchase attempt** | Standard, precise, works across instances and services |

## Decision

1. The client creates a unique request identifier (UUID) for each logical purchase attempt and sends it with every retry of that attempt.
2. The server stores the identifier with the user ID and the outcome. The first request inserts it and is processed; any later request with the same identifier returns the stored outcome (or *processing*) without doing new work.
3. Identifiers are bound to the user; another user's identifier is rejected.
4. In PostgreSQL the identifier is unique (DB-3); during the sale a fast copy may be kept in the in-memory layer so repeats are answered without database work.
5. Every payment request uses the **order ID** as the payment idempotency key, so internal retries cannot double-charge.
6. All event consumers are idempotent (ADR-0009).
7. The UI disables the Buy button and shows status after the first click (UI-3).

## Consequences

- Retries cannot create extra reservations, orders or charges; verified by T-4 (PR-2 = 0).
- Repeats are answered quickly, which reduces retry amplification.
- Requires the client to send the identifier and the server to keep identifiers for a retention period.
