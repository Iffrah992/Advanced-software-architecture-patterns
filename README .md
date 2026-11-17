# Architecture Decision Log — Flash Sale Purchase System

Architecture Decision Records (ADRs) for the flash-sale project, following the
[architecture-decision-record](https://github.com/architecture-decision-record/architecture-decision-record) conventions:

- **Template:** Michael Nygard — *Status, Context, Decision, Consequences*, plus date, alternatives and links.
- **File names:** present-tense imperative verb phrase, lowercase with dashes, `.md`.
- **One decision per ADR.** An accepted ADR is not rewritten; it is amended with a dated note or superseded by a new ADR.

Requirement IDs refer to [`docs/srs.md`](../docs/srs.md) (v1.3). Bottleneck IDs **B1–B6** refer to SRS Appendix B.3:

| ID | Bottleneck (SRS B.3) |
|---|---|
| B1 | Shared inventory contention |
| B2 | Overselling |
| B3 | Database connection exhaustion |
| B4 | Slow payment dependency |
| B5 | Retry amplification |
| B6 | Fairness |

## Log

| ADR | Decision | Status | Bottlenecks | Requirements |
|---|---|---|---|---|
| [0001](record-architecture-decisions.md) | Record architecture decisions | Accepted | — | C-6 |
| [0002](use-postgresql-as-system-of-record.md) | Use PostgreSQL as the persistent system of record | Accepted | B2 | C-2, SI-1, DB-1–DB-4 |
| [0003](use-layered-architecture-as-baseline.md) | Use a layered monolith as the measured baseline | Accepted | all (measured) | C-5, FR-15–FR-18 |
| [0004](combine-patterns-into-final-architecture.md) | Adopt the hybrid target architecture | Accepted | B1–B6 | C-7, C-8 |
| [0005](control-entry-with-server-side-queue.md) | Control entry with a server-side admission/FIFO queue | Accepted | B1, B5, B6 | FR-1–FR-4, C-4, C-8 |
| [0006](claim-stock-atomically.md) | Claim stock atomically | Accepted | B2 | FR-5–FR-7, PR-1, SAF-1 |
| [0007](keep-hot-stock-in-memory-grid.md) | Keep hot stock in a space-based in-memory layer | Accepted | B1, B3 | FR-4, FR-5, SI-2 |
| [0008](release-database-before-payment.md) | Release the database before payment | Accepted | B3, B4 | FR-8–FR-12, SAF-3 |
| [0009](use-event-driven-broker-topology.md) | Use event-driven broker topology | Accepted | B4 | FR-9–FR-12, SI-3 |
| [0010](split-into-microservices.md) | Split into Inventory, Order and Payment microservices | Accepted | B1, B4 | C-7, §5.4 |
| [0011](use-idempotency-keys.md) | Use idempotency keys | Accepted | B5 | FR-13, FR-14, PR-2, SAF-2 |
| [0012](use-microkernel-for-sale-rules.md) | Use microkernel for sale rules | **Rejected** | B6 | §5.4 Fairness |
| [0013](choose-in-memory-data-grid-technology.md) | Choose the in-memory data-grid technology | **Proposed** | B1 | SI-2, App. C #2 |
| [0014](choose-event-broker-technology.md) | Choose the event-broker technology | **Proposed** | B4 | SI-3, App. C #3 |

## Status values

`Proposed` → `Accepted` → (`Deprecated` | `Superseded by …`). `Rejected` records an option that was evaluated and not taken.

## Template for new ADRs

```markdown
# ADR-NNNN: Title (imperative verb phrase)

**Date:** YYYY-MM-DD

## Status

Proposed | Accepted | Rejected | Deprecated | Superseded by [ADR-XXXX](file.md)

## Context

The problem, the forces in play, and the options considered.

## Decision

What we will do.

## Consequences

What becomes easier or harder; follow-up decisions; how it will be verified.
```
