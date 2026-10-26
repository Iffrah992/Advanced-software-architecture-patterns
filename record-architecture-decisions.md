# Record architecture decisions

**Date:** 2026-10-24

## Status

Accepted

## Context

The flash-sale project is solved step by step with five architecture patterns. Each step changes one thing and is validated by load testing. Our supervisor reviews progress on GitHub and needs to see **why** each change was made, not only **what** changed. Without a record, the reasons behind choices such as "why not a mediator?" are lost, and later steps may undo earlier fixes by accident.

Options considered:

- Comments in code — scattered, and they explain code rather than architecture.
- A single design document — hard to see the order and evolution of decisions.
- ADRs in the repository — one small file per decision, versioned with the code.

## Decision

We will record every architecturally significant decision as an ADR in `/adr`, using the Michael Nygard template and the file-name conventions of the architecture-decision-record project. Each ADR links to the SRS requirements and problems (P1–P5) it addresses. Each step of the evolution is committed together with its ADR.

## Consequences

- The commit history shows the step-by-step evolution with its reasoning.
- Requirements (SRS) → decisions (ADR) → patterns are traceable in both directions.
- Writing an ADR takes a little time for each decision; we accept this cost.
- Decisions are not edited after acceptance; changes require a dated amendment or a new superseding ADR.
