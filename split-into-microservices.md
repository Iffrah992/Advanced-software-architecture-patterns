# ADR-0010: Split into Inventory, Order and Payment microservices

**Date:** 2026-11-10

## Status

Accepted. Supersedes the single-deployable structure of the baseline ([ADR-0003](use-layered-architecture-as-baseline.md)) for the target architecture.

## Context

The parts of the purchase flow have very different load profiles during a sale:

| Part | Load at sale start |
|---|---|
| Admission gate | The whole crowd |
| Inventory | Only admitted requests, but correctness-critical |
| Order | Order creation plus status reads |
| Payment | Limited by the external payment service |

In a monolith everything must be scaled for the heaviest part, and a fault in one area can stop purchases. The SRS requires inventory, order and payment to be separable and independently scalable (§5.4 Scalability, Maintainability; C-7).

Microservice topologies considered:

| Topology | Assessment |
|---|---|
| API REST-based | Suitable for the buyer-facing endpoints (purchase, status) |
| Application REST-based | Not needed; the UI is minimal |
| Centralized messaging | Fits the broker topology chosen in ADR-0009 |

## Decision

The target system is split into three services behind the admission gate, using **REST at the edge** and **centralized messaging** between services:

| Service | Responsibility | Data |
|---|---|---|
| Inventory Service | Atomic claim, reservations, release | Hot stock in memory (ADR-0007); reservation schema in PostgreSQL |
| Order Service | Pending/confirmed/failed orders; purchase status for buyers | Order schema in PostgreSQL |
| Payment Service | Payment requests and results | Payment-attempt schema in PostgreSQL |

The admission gate is a separately scalable component in front of the Inventory Service (ADR-0005).

Rules to avoid a distributed monolith:

1. No synchronous service-to-service call chain on the purchase path; services react to events.
2. Each service owns its schema; no service reads another service's tables.
3. Choreography through events, not a central orchestrator.
4. Data needed by another service travels in the event (e.g. price and user ID in `StockReserved`).
5. Each service keeps a simple layered internal structure.

## Consequences

- The gate and Inventory Service can be scaled for the burst while Payment stays bounded.
- A failure in one service does not stop the others from accepting or finishing work (Reliability, §5.4).
- New costs: network latency, partial failures, more deployables, contract testing of events, and a correlation ID in logs for tracing.
- Status may briefly lag behind the real state (eventual consistency); the UI shows *processing*.
