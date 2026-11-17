# ADR-0003: Use a layered monolith as the measured baseline

**Date:** 2026-11-10

## Status

Accepted (baseline only). The target architecture is defined in [ADR-0004](combine-patterns-into-final-architecture.md).

## Context

The project method is *build the baseline → load-test it → identify the bottleneck → change one architectural decision → test again* (SRS §6.2, C-5). To know which bottleneck (B1–B6) fails first we need a simple, typical implementation to measure — evidence, not assumptions.

Options considered:

| Option | Assessment |
|---|---|
| Start directly with the hybrid architecture | No reference point; cannot show what each change improves |
| **Layered monolith (Spring Boot: controller → service → repository → PostgreSQL)** | The usual default; quick to build; reproduces the expected bottlenecks |
| Microservices baseline | Network and operational complexity would hide the real bottlenecks |

## Decision

The baseline is one Spring Boot application with closed layers (presentation, business, persistence, database). A purchase checks stock, creates the order, calls the simulated payment service (with configurable delay and failure rate, AS-2) and confirms the order synchronously, using PostgreSQL.

The baseline already records the metrics required by FR-15–FR-18 (latency, error rate, DB connections, lock waits, payment time, oversold and duplicate counts), so the same measurements can be compared after every change.

Tests T-1 to T-6 are run on the baseline first and the results are committed as the reference.

## Consequences

- Expected observations: lock waits on the shared stock row (B1), possible overselling if check and claim are separate (B2), connection pool exhaustion during slow payment (B3, B4), and growing retries (B5).
- The layered structure is easy to understand and test, and is reused as the internal structure of each service later.
- Its known limits — one deployable unit and a synchronous flow — motivate ADR-0004.
