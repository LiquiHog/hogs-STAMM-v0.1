# STAMM Documentation

Multi-tier constant-product AMM for Algorand with inline fee redistribution and pull-model treasury.

---

## Documentation Structure

### Core Concepts

foundational architecture and tier mechanics

- **[Architecture](core/architecture.md)** — Contract design, state layout, deployment flow
- **[Tiers](core/tiers.md)** — 6-tier system, lifecycle, activation/deactivation

### Features

Key protocol mechanisms and operations

- **[Fee Engine](features/fee-engine.md)** — Two-sided fees, 80/20 split, inline spill
- **[Oracle](features/oracle.md)** — TWAP oracle with aggregate reserves
- **[Swaps & Liquidity](features/swaps-and-liquidity.md)** — All operation types (swap, mint, burn variants)

### Reference

Technical details and supporting documentation

- **[Glossary](reference/glossary.md)** — Term definitions and abbreviations
- **[Indexing](reference/indexing.md)** — On-chain registry lookups for pools, assets, and LP tokens
- **[Security](reference/security.md)** — Access control, invariants, attack vectors
- **[Admin Contract](reference/admin-contract.md)** — Governor lifecycle, timelocks, freeze, migration, treasury
- **[Registry Contract](reference/registry-contract.md)** — Pair/LP registries, writer authorization, permanence

### Governance

- **[Governance](core/governance.md)** — Timelocks, freeze mechanisms, admin transfer, governor migration

### Testing

Comprehensive test coverage and validation methodology

- **[Tests Overview](tests/README.md)** — Integration and fuzz testing summary

#### Integration Tests
- **[Overview](tests/integration/overview.md)** — Test environment and methodology
- **[Trading Operations](tests/integration/trading-operations.md)** — Swap tests across all tiers
- **[Liquidity Management](tests/integration/liquidity-management.md)** — Mint and burn operation tests
- **[Tier Dynamics](tests/integration/tier-dynamics.md)** — Tier lifecycle and state transition tests

#### Fuzz Tests
- **[Overview](tests/fuzz/overview.md)** — Property-based testing approach
- **[Invariants](tests/fuzz/invariants.md)** — Mathematical property validation
- **[Edge Cases](tests/fuzz/edge-cases.md)** — Boundary conditions and corner cases

---

## Quick Start

1. **Understand the basics**: Start with [Architecture](core/architecture.md) and [Tiers](core/tiers.md)
2. **Learn the mechanics**: Read [Fee Engine](features/fee-engine.md) and [Swaps & Liquidity](features/swaps-and-liquidity.md)
3. **Explore operations**: See [Swaps & Liquidity](features/swaps-and-liquidity.md) for all operation types
4. **Reference details**: Use [Glossary](reference/glossary.md) for terminology and [Security](reference/security.md) for safety properties

---

## Documentation Philosophy

- **Accuracy**: All documentation verified against contract implementation
- **Completeness**: Every feature, parameter, and edge case documented
- **Clarity**: Complex mechanisms explained with examples and diagrams
- **Verification**: Test documentation shows validation of all documented behavior

---

## License

This documentation is provided for informational purposes. The STAMM protocol is proprietary software. Contact LiquiHog@gmail.com or @LiquiHog on X for SDK access and licensing information.
