# ADR-0004: Adopt the hybrid target architecture

**Date:** 2026-11-10

## Status

Accepted

## Context

Five architecture patterns were evaluated against the flash-sale bottlenecks (see [architecture-patterns.md](../docs/architecture-patterns.md)). Ratings follow Richards, *Software Architecture Patterns*:

| Characteristic | Layered | Event-Driven | Microkernel | Microservices | Space-Based |
|---|---|---|---|---|---|
| Overall agility | Low | High | High | High | High |
| Ease of deployment | Low | High | High | High | High |
| Testability | High | Low | High | High | Low |
| Performance | Low | High | High | Low | High |
| Scalability | Low | High | Low | High | High |
| Ease of development | High | Low | Low | High | Low |

No single pattern solves all six bottlenecks:

| Bottleneck | Best response |
|---|---|
| B1 Shared inventory contention | Space-Based (hot stock in memory) + admission gate |
| B2 Overselling | Atomic claim (works in any pattern) |
| B3 DB connection exhaustion | Event-Driven (short transactions, async payment) |
| B4 Slow payment | Event-Driven + separate Payment service |
| B5 Retry amplification | Admission gate + idempotency |
| B6 Fairness | Server-side FIFO gate |

## Decision

The target system is a **hybrid** of three patterns plus admission control (C-7, C-8):

| Element | Role | ADR |
|---|---|---|
| Server-side admission/FIFO gate | Controls the burst; defines allocation order | ADR-0005 |
| Space-Based Architecture | Holds hot inventory state in memory during the sale | ADR-0007 |
| Event-Driven Architecture (broker topology) | Decouples order and payment from the stock claim | ADR-0009 |
| Microservices | Inventory, Order and Payment deployed and scaled separately | ADR-0010 |
| PostgreSQL | Durable system of record | ADR-0002 |

Cross-cutting rules: atomic stock claim (ADR-0006), no transaction across the payment call (ADR-0008), idempotent requests and consumers (ADR-0011).

Not selected as target patterns:

- **Layered** — kept only as the baseline (ADR-0003) and as the internal structure of each service; low performance and scalability.
- **Microkernel** — evaluated for pluggable allocation rules and rejected for the current scope (ADR-0012).

## Consequences

- Every bottleneck B1–B6 has a named response and a test (SRS §6.2, §6.4).
- Accepted trade-offs: eventual consistency, event ordering, duplicate messages, more deployables and harder end-to-end testing.
- The hybrid is introduced step by step, one decision at a time, and compared against the baseline after each step; if a step does not improve the measured bottleneck, it is reconsidered in a new ADR.
