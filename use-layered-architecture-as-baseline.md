# Use layered architecture as baseline

**Date:** 2026-10-24
**Step:** 0 · **Pattern:** Layered · **Requirements:** C-5, §4 Verification

## Status

Accepted. The monolithic deployment is superseded by [Split into microservices](split-into-microservices.md); the layered structure remains inside each service.

## Context

Our method is *build simple → load-test → find what breaks → change one thing*. To know which bottleneck fails first (database locks, connection pool, or payment delays), we need a simple, typical implementation to measure against — evidence, not guesses.

Options considered:

| Option | Assessment |
|---|---|
| Start directly with the final distributed design | Cannot show which problem each change fixes; high effort before any measurement |
| **Layered monolith (Spring Boot: controller → service → repository → DB)** | The natural default; easy to build; makes the five problems reproducible |
| Microservices from day one | Network and ops complexity would hide the real bottlenecks |

## Decision

We build a layered monolith with closed layers (presentation, business, persistence, database), performing the five purchase steps (check, reserve, create order, pay, confirm) in one `@Transactional` method, with the payment gateway replaced by a stub that delays 5 seconds.

We then run SRS tests T-1 (burst), T-2 (oversell) and T-3 (slow payment) and record the results as the baseline.

## Consequences

- Expected results: oversold units (P2), inventory row lock waits (P1), connection pool exhaustion during the 5 s payment (P3), and growing retries (P4).
- Every later step is compared against these numbers.
- The layered structure is easy to test and becomes the internal structure of each service later.
- Its limits (one deployable, synchronous flow, low scalability) motivate Steps 4–7.
