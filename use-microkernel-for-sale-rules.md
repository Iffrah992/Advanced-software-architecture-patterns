# Use microkernel for sale rules

**Date:** 2026-10-25
**Step:** 5 · **Pattern:** Microkernel · **Problem:** P5 Fairness (policy flexibility) · **Requirements:** FR-20, FR-21, MA-1, FR-22

## Status

Accepted

## Context

Step 3 introduced strict FIFO by server acceptance order. It is predictable, but it still favours buyers closer to the server. The project lists **randomized queues** and **lottery-based allocation** as alternatives "if fairness requirements change". Other things also vary per sale: payment provider, per-user limits, anti-bot checks.

If each variation is an `if/else` in the purchase code, the most correctness-critical code changes for every new sale rule, which risks breaking P2/P3 fixes.

Options considered:

| Option | Assessment |
|---|---|
| `if/else` / `switch` on a config value | Fast to write; core changes and re-testing for each new rule |
| Microkernel as the whole system architecture | Core would be a single deployable; does not scale the hot path |
| **Microkernel inside the Gate and Payment components** | Stable core, rules as plug-ins with a contract, unit-testable |

## Decision

Apply the microkernel pattern **internally**:

- **Core (fixed):** ticketing, admission, atomic claim, reservation lifecycle.
- **Plug-in contracts:**
  - `AllocationPolicy` — `FIFO` (default), `RANDOMIZED_QUEUE`, `LOTTERY`;
  - `PaymentProvider` — one adapter per gateway;
  - `EntryCheck` — per-user limit, anti-bot, region rules.
- **Registry:** Spring collects all implementations into a map by name; the sale configuration selects which ones are active. Configuration is frozen when the sale starts (FR-22).
- A `LOTTERY` sale collects entries for a fixed window (e.g. 60 s), then draws winners with `SecureRandom`; the seed is logged so the draw can be audited.
- Plug-ins never call each other; they only use the core contract.

## Consequences

- A new fairness policy is a new class plus tests; the core claim and payment code is untouched.
- Fairness rules can be compared experimentally (FIFO vs lottery) using test T-7.
- Contract design must be stable; breaking changes to `AllocationPolicy` require a new ADR.
- Adds a small amount of indirection and configuration validation (unknown policy names must fail at configuration time, not at 12:00).
