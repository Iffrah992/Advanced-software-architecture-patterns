# ADR-0012: Use microkernel for sale rules

**Date:** 2026-11-10

## Status

Rejected for the current project scope. May be reconsidered if the fairness rule must change between sales.

## Context

The fairness requirement (§5.4, B6) asks for *a defined server-side allocation rule*. ADR-0005 chooses strict FIFO by server acceptance order. Alternatives such as a randomized queue or a lottery could be fairer to buyers with slower networks.

The microkernel pattern was evaluated to make sale rules pluggable:

- **Core:** ticketing, admission, atomic claim, reservation lifecycle.
- **Plug-ins:** allocation policies (FIFO, randomized queue, lottery), per-user limit rules, payment adapters, behind a stable contract and a registry.

| Characteristic | Microkernel rating | Relevance |
|---|---|---|
| Agility | High | New policy without changing the core |
| Testability | High | Each policy is a small unit-testable class |
| Performance | High | In-process calls |
| Scalability | Low | Does not address B1 or B3 |
| Ease of development | Low | A stable plug-in contract needs careful design |

## Decision

Do **not** adopt the microkernel pattern now. The project uses one fixed allocation rule (server-side FIFO, ADR-0005) and one simulated payment service (AS-2). Pluggability adds design effort without addressing any bottleneck the project measures, and C-7 fixes the target patterns as Microservices, Event-Driven and Space-Based.

To keep the option open, the allocation rule is implemented behind a single Java interface in the gate, with FIFO as the only implementation.

## Consequences

- Less code and configuration to build and test.
- Changing to a lottery or randomized policy later means adding an implementation of the interface and recording a new ADR that supersedes this one.
- Fairness test T-6 checks FIFO only.
