# ADR-0013: Choose the in-memory data-grid technology

**Date:** 2026-11-10

## Status

Proposed. To be accepted after a prototype claim test (SRS Appendix C, item 2).

## Context

ADR-0007 requires a distributed in-memory component for the hot inventory state, the admission ticket counter (ADR-0005), and optionally a fast copy of request identifiers (ADR-0011). Requirements for the product:

- one atomic operation for "check limit + check stock > 0 + decrement + record buyer";
- an atomic, strictly increasing counter for tickets;
- replication or persistence to reduce data loss on failure;
- a good Java/Spring Boot client and easy local/container setup.

| Option | Atomic claim | Ticket counter | Durability | Java / ops | Notes |
|---|---|---|---|---|---|
| **Redis** | Lua script executed atomically | `INCR` | Replication + AOF persistence | Spring Data Redis; one container | Simple; single-threaded per key gives a clear owner |
| Hazelcast | `IMap` `EntryProcessor` | `IAtomicLong` (CP subsystem) | In-memory replicas; optional persistence | Java-native, embeddable | Closest to the classic space-based processing-unit model; more cluster concepts |
| Apache Ignite | Transactions / entry processors | Atomic sequence | Native persistence | Java-native | Heavier to configure for a student project |

## Decision

**Proposed: Redis**, with:

- the claim implemented as one Lua script per request;
- tickets from `INCR` on one key per sale;
- one primary per stock key, replicas for failover, AOF persistence enabled.

Hazelcast is the fallback if a Java-native embedded grid is preferred.

The decision becomes Accepted when a prototype passes T-2 (no oversell) and shows lower claim latency than the PostgreSQL baseline in T-1.

## Consequences

- If accepted, the gate and Inventory Service share one Redis deployment for tickets and stock.
- Redis becomes critical during the sale; its availability and persistence settings must be documented in the README.
- If the prototype fails the criteria, this ADR is rejected and a new ADR selects another option.
