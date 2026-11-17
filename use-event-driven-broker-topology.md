# ADR-0014: Choose the event-broker technology

**Date:** 2026-11-10

## Status

Proposed. To be accepted after the event-flow prototype (SRS Appendix C, item 3).

## Context

ADR-0009 requires a broker for events between Inventory, Order and Payment. Requirements for the product:

- durable queues and acknowledgements (no lost events);
- competing consumers with bounded concurrency (steady payment processing);
- dead-letter handling;
- a good Spring Boot integration and easy container setup.

After the admission gate, event volume is small (hundreds of purchase events per sale, not millions).

| Option | Strengths | Weaknesses for this project |
|---|---|---|
| **RabbitMQ** | Work queues, acknowledgements, dead-letter exchanges, Spring AMQP, simple to run | No long-term event replay |
| Apache Kafka | Very high throughput, partitioned ordering, replayable log | More operational setup; capacity far beyond what is needed |
| PostgreSQL-based queue (outbox polling only) | No extra component | Adds load to the database that the project is trying to relieve |

## Decision

**Proposed: RabbitMQ**, with durable queues, manual acknowledgements, one dead-letter queue per consumer, and bounded consumer concurrency for the Payment Service. Events are published from the transactional outbox (ADR-0009).

The decision becomes Accepted when T-3 and T-5 pass on the prototype: payment delay does not affect inventory latency, and failed payments return stock correctly.

## Consequences

- If accepted, RabbitMQ is added to the container setup and its queues are documented in the README.
- Replaying past events is not supported; the outbox table and PostgreSQL records are the audit source.
- If event volume or replay needs grow, Kafka can be reconsidered in a new ADR.
