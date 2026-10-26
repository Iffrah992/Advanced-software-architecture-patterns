# Architecture Decision Log

Architecture Decision Records (ADRs) for the flash-sale system, following the
[architecture-decision-record](https://github.com/architecture-decision-record/architecture-decision-record) conventions:

- **Template:** Michael Nygard — *Title, Status, Context, Decision, Consequences* (plus date and links).
- **File names:** present-tense imperative verb phrase, lowercase with dashes, `.md`.
- **One decision per ADR.** A decision is never rewritten; it is amended with a dated note or **superseded** by a new ADR.

## Log (in the order decisions were made)

| # | ADR | Status | Step | Solves |
|---|---|---|---|---|
| 1 | [Record architecture decisions](record-architecture-decisions.md) | Accepted | — | — |
| 2 | [Use layered architecture as baseline](use-layered-architecture-as-baseline.md) | Accepted, partly superseded by 9 | 0 | Baseline to measure |
| 3 | [Claim stock atomically](claim-stock-atomically.md) | Accepted, amended by 10 | 1 | P2 Overselling |
| 4 | [Use idempotency keys](use-idempotency-keys.md) | Accepted | 2 | P4 Retry storms |
| 5 | [Control entry with a server-side queue](control-entry-with-server-side-queue.md) | Accepted | 3 | P1, P4, P5 |
| 6 | [Release the database before payment](release-database-before-payment.md) | Accepted | 4 | P3 Slow payments |
| 7 | [Use event-driven broker topology](use-event-driven-broker-topology.md) | Accepted | 4 | P3, P1 |
| 8 | [Use microkernel for sale rules](use-microkernel-for-sale-rules.md) | Accepted | 5 | P5 flexibility |
| 9 | [Split into microservices](split-into-microservices.md) | Accepted | 6 | Independent scaling |
| 10 | [Keep hot stock in an in-memory grid](keep-hot-stock-in-memory-grid.md) | Accepted | 7 | P1 DB jam |
| 11 | [Combine patterns into the final architecture](combine-patterns-into-final-architecture.md) | Accepted | ✓ | P1–P5 |

## Status values

`Proposed` → `Accepted`

## How to add an ADR

```markdown
# Title 
OPTIMIZING HIGH TRAFFIC ON FLASH SALE 
**Date:** 2026-10-06

## Status

Proposed | Accepted 

## Context

What is the issue that motivates this decision? What forces and options are in play?

## Decision

What is the change that we are proposing and/or doing?

## Consequences

What becomes easier or harder because of this change? Follow-up decisions?
```
