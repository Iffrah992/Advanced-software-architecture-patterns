# Split into microservices

**Date:** 2026-10-26
**Step:** 6 · **Pattern:** Microservices · **Requirements:** AV-2, DC-2, MA-2, PO-1

## Status

Accepted. Supersedes the single-deployable part of [Use layered architecture as baseline](use-layered-architecture-as-baseline.md).

## Context

After Steps 1–5 the logic is correct and asynchronous, but everything is still one deployable. The parts have very different load profiles:

| Part | Load at 12:00 |
|---|---|
| Gate | The whole crowd (10,000+ requests per second) |
| Inventory | ~120 claims (only admitted tickets) |
| Payment | Limited by the bank's rate |
| Order status | Continuous polling |

In a monolith we must scale everything to handle the Gate's load, and a bug or redeploy in notifications can take down purchases.

Topologies considered:

| Topology | Assessment |
|---|---|
| API REST-based (fine-grained services via API gateway) | Good for the buyer-facing edge |
| Application REST-based | Not needed; single UI |
| Centralized messaging (broker between services) | Fits the event-driven design from Step 4 |

## Decision

Split into five services, using an **API gateway at the edge** and **centralized messaging (RabbitMQ)** between services:

| Service | Owns (own database/keyspace) | Scaling |
|---|---|---|
| Gate | Tickets, rate limits, waitlist (Redis) | Scale out heavily before 12:00 |
| Inventory | Stock, reservations | Single writer per SKU |
| Order | Orders, purchase status (read endpoint) | Moderate |
| Payment | Payment attempts | Bounded worker pool |
| Notification | Outgoing messages | Any; failure does not block purchases |

**Avoid dependencies and orchestration:**

1. No synchronous service-to-service calls on the purchase path (DC-2); services react to events.
2. Database per service; no shared tables.
3. Choreography, not orchestration — no central service directs the others.
4. Data needed elsewhere travels inside events (e.g. price snapshot in `StockReserved`), accepting small duplication over coupling.
5. Each service keeps a layered internal structure.

## Consequences

- The Gate scales independently to absorb the burst; Payment stays bounded; failures are isolated.
- Each service can be deployed and tested alone; event contracts need contract tests.
- New costs: network latency, partial failures, distributed tracing (OB-1), more deployables (Docker Compose locally).
- Eventual consistency: the status page may show "processing" briefly.
- The Inventory service still depends on one hot DB row; addressed in [Keep hot stock in an in-memory grid](keep-hot-stock-in-memory-grid.md).
