# Release the database before payment

**Date:** 2026-10-25
**Step:** 4 · **Pattern:** Event-Driven · **Problem:** P3 Slow payments · **Requirements:** FR-9 – FR-14, DC-1, PR-5, PR-6

## Status

Accepted

## Context

In the baseline, the payment call happens inside the purchase transaction. The bank takes ~5 s (up to 30 s). During that time the DB connection stays checked out. With a pool of e.g. 50 connections, only 50 purchases can be in progress; everyone else cannot even start. CPU is fine — the **connection pool** is the real limit. Slow pages then trigger retries (P4).

Options considered:

| Option | Assessment |
|---|---|
| Bigger connection pool | DB has a hard connection limit; more connections also increase lock contention |
| Shorter payment timeout | Real payments still take seconds; more false failures |
| Commit stock + order, call payment synchronously afterwards (same request) | Frees the DB, but the HTTP thread still waits ~5 s and the client may retry |
| **Reserve → commit → queue → pay asynchronously, with reservation TTL** | DB busy only during the reserve step; payment load becomes steady |

## Decision

The purchase is split into a short transactional part and an asynchronous part:

1. **Reserve** (one short transaction): atomic claim, insert reservation with `expires_at = now + 5 min`, insert order `PENDING_PAYMENT`, insert `StockReserved` into the outbox. Commit. *The DB connection is released here.*
2. **Queue:** the outbox relay publishes the event; payment workers (bounded pool) consume it.
3. **Pay:** a worker calls the gateway with the order ID as idempotency key. No DB connection is held during the call.
4. **Confirm or release:**
   - success → order `CONFIRMED`;
   - failure → order `FAILED`, unit returned to stock exactly once;
   - no answer before `expires_at` → a scheduler publishes `ReservationExpired` and the unit is returned.
5. The HTTP request returns `202 Accepted` right after step 1; the page polls for the result.

**Rule (DC-1):** no transaction may span a call to an external system.

## Consequences

- Connection hold time per purchase drops from ~5 s to milliseconds (PR-6); the pool no longer runs out (PR-5).
- Stock is never lost to abandoned or stuck payments.
- Buyers see a two-phase experience: *Reserved — pay within 5:00*, then *Confirmed*.
- A late payment success after expiry must be handled: the payment is refunded automatically and the order stays `FAILED` (rare; logged and alerted).
- Requires an event infrastructure — decided in [Use event-driven broker topology](use-event-driven-broker-topology.md).
