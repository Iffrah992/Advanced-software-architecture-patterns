# Combine patterns into the final architecture

**Date:** 2026-10-26
**Step:** Final · **Problems:** P1 – P5

## Status

Accepted

## Context

The pattern analysis (see [architecture-patterns.md](../docs/architecture-patterns.md#a-pattern-analysis-summary)) shows that no single pattern is strong everywhere:

| Pattern | Strong at | Weak at |
|---|---|---|
| Layered | Testability, ease of development | Performance, scalability |
| Event-Driven | Performance, scalability | Testability, development effort |
| Microkernel | Agility, testability | Scalability |
| Microservices | Deployment, scalability | Performance (network hops) |
| Space-Based | Performance, scalability | Testability, development effort |

Choosing only one would leave at least one flash-sale problem unsolved.

## Decision

The final system is a **hybrid**, with each pattern used where its strength is needed:

| Pattern | Used for | Problem addressed |
|---|---|---|
| Space-Based | Gate + stock claim in Redis, DB written behind | P1, P2, P5 |
| Event-Driven (broker) | Reserve → queue → pay → confirm/release between services | P3, P1 |
| Microservices | Gate, Inventory, Order, Payment, Notification deployed and scaled separately | Independent scaling, isolation |
| Microkernel | `AllocationPolicy` in Gate, `PaymentProvider` in Payment | P5 flexibility |
| Layered | Internal structure of every service | Maintainability, testability |
| Idempotency keys (cross-cutting) | API, consumers, payment gateway | P4 |

End-to-end flow: **Click → Gate (ticket, admission) → Reserve (atomic, in memory) → Queue → Pay → Confirmed, or Payment failed → item back on sale.**

## Consequences

- All five problems have a named fix, a requirement and a test (SRS Appendix B).
- The design is more complex than any single pattern; complexity is concentrated where the load is (Gate, stock claim) and kept simple elsewhere.
- Future work: will compare FIFO with lottery using T-7; evaluate Hazelcast vs Redis for the grid
