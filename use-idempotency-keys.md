# Use idempotency keys

**Date:** 2026-10-24
**Step:** 2 · **Pattern:** Layered (business layer) · **Problem:** P4 Retry storms · **Requirements:** FR-16 – FR-19, CI-2, UI-1

## Status

Accepted

## Context

When the page is slow, buyers click "Buy" again, browsers retry, and the network may deliver a request twice. Each repeat creates more load (1× → 2× → 4×), and without protection a repeat can create a second reservation, a second order, or a **second charge**. Retries cannot be prevented entirely, so they must be made harmless.

Options considered:

| Option | Assessment |
|---|---|
| Disable the button only | Helps humans, but not network retries, refreshes or scripts |
| Deduplicate by user ID | Blocks legitimate second attempts after a failed payment |
| **Client-generated idempotency key per purchase attempt** | Standard approach; precise; works across servers |

## Decision

1. The client generates a UUID v4 when the purchase page loads and sends it as the `Idempotency-Key` header on every attempt of that purchase.
2. The server stores `key → (userId, status, response)`. It first tries to insert the key (`SET key value NX EX 86400` in Redis, or an insert into `idempotency_record` with the key as primary key).
   - Insert succeeds → process the request normally and store the final response.
   - Key already exists → return the stored response (or "processing") without doing any work.
3. The key is bound to the user ID; a key presented by another user is rejected.
4. Every payment-gateway call uses the **order ID** as the gateway's idempotency key, so our own retries to the bank never double-charge.
5. The UI disables the Buy button after the first click (UI-1).

## Consequences

- A repeated click is recognised — no double reservation, no double charge (T-4: 0 duplicates with 3× retries).
- Repeats are answered in milliseconds from Redis, which also reduces storm load.
- Requires storage of keys for 24 h and an agreed header contract with the frontend.
- All event consumers must also be idempotent (FR-16); this is applied in [Use event-driven broker topology](use-event-driven-broker-topology.md).
