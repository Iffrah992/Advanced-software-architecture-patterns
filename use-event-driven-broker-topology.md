# Use event-driven broker topology

**Date:** 2026-10-25
**Step:** 4 · **Pattern:** Event-Driven · **Problems:** P3, P1 · **Requirements:** FR-11, FR-14 – FR-16, RL-1, RL-2

## Status

Accepted

## Context

[Release the database before payment](release-database-before-payment.md) requires purchase steps to run asynchronously and to react to outcomes (payment succeeded, failed, reservation expired). The event-driven pattern offers two topologies.

| Topology | How it works | For the flash sale |
|---|---|---|
| **Mediator** | A central mediator receives each purchase and orchestrates each step through channels | Every one of the 10,000 purchases passes through one component → a new central bottleneck and single point of failure; our flow is short and linear and does not need orchestration |
| **Broker** | No central controller; each processor reacts to an event and publishes a new one through a lightweight broker | No single hot component; processors scale separately; matches a simple linear chain with one compensation path |

Broker choice: RabbitMQ (work queues with acknowledgements, dead-letter queues, simple ops) vs Kafka (log replay, very high throughput). Our event volume is small (≈ hundreds of purchase events per sale after the gate), so RabbitMQ is sufficient and simpler.

## Decision

Use the **broker topology** with RabbitMQ. Event chain:

```
PurchaseAdmitted → StockReserved | SoldOut
StockReserved    → OrderCreated → (payment workers) → PaymentSucceeded | PaymentFailed
PaymentSucceeded → OrderConfirmed → notification
PaymentFailed / ReservationExpired → StockReleased → gate admits next waitlisted ticket
```

Reliability rules:

1. **Transactional outbox:** each state change and its event are written in the same DB transaction; a relay publishes from the outbox. No "saved but not announced" or the reverse.
2. **At-least-once delivery + idempotent consumers:** each consumer records processed event IDs or checks state before acting (e.g. release only if reservation is `ACTIVE`).
3. **Compensation instead of distributed transactions:** failure is handled by `StockReleased`, not by rollback across processors.
4. **Dead-letter queues** for events that fail repeatedly, with alerts.
5. **Bounded consumers:** payment workers have a fixed concurrency, so the bank receives a steady flow.

## Consequences

- No central bottleneck in the purchase flow; payment, inventory and notifications scale independently.
- Load is smoothed: bursts are absorbed by queues (*process gradually*).
- Testing is harder: async flows and failure orders (e.g. payment success after expiry) need dedicated integration tests.
- Developers must follow the outbox and idempotent-consumer rules everywhere; we provide shared helper classes and a checklist in code review.
- The mediator topology remains an option for a future multi-step workflow (e.g. returns), recorded here as *considered, rejected for the purchase path*.
