# ADR-0001: Record architecture decisions

**Date:** 2026-11-10

## Status

Accepted

## Context

The project starts from a baseline, measures its bottlenecks, and then changes one architectural decision at a time (SRS §6.2). Constraint C-6 requires major decisions to be recorded. The supervisor reviews progress on GitHub and needs to see **why** each change was made, not only what changed.

Options considered:

| Option | Assessment |
|---|---|
| Explain decisions only in the SRS | The SRS states requirements; mixing in design rationale makes it hard to read and to change |
| Code comments | Scattered; explain code, not architecture |
| **ADRs in the repository** | One small file per decision, versioned with the code, easy to review |

## Decision

Every architecturally significant decision is recorded as an ADR in `/adr`, using the Michael Nygard template and the naming conventions of the architecture-decision-record project. Each ADR links to the SRS requirements and bottlenecks (B1–B6) it addresses. Technology choices that are still open in SRS Appendix C are recorded as **Proposed** until confirmed.

## Consequences

- Requirements (SRS) and decisions (ADRs) are traceable in both directions (SRS §6.3, §6.4).
- The commit history shows the step-by-step evolution together with its reasoning.
- Accepted ADRs are not edited; changes are made by a dated amendment or a superseding ADR.
