# STAMM Protocol

[![All Rights Reserved](https://img.shields.io/badge/License-Proprietary-red.svg)](LICENSE)
[![GitHub Issues](https://img.shields.io/github/issues/LiquiHog/hogs-STAMM-v0.1)](https://github.com/LiquiHog/hogs-STAMM-v0.1/issues)

**Stratified Automated Market Maker — by HOG**

STAMM is a multi-tier constant-product AMM built on Algorand. Instead of a single liquidity pool per asset pair, STAMM stratifies liquidity across 8 independent tiers — each with its own reserves, fee rate, and LP token — all managed atomically within a single smart contract.

---

## Why STAMM?

Traditional AMMs force liquidity providers into a one-size-fits-all fee structure. High fees protect LPs from impermanent loss but repel traders. Low fees attract volume but expose LPs to adverse selection. There's no right answer because different LPs have different risk appetites.

STAMM solves this by letting liquidity self-organize across fee tiers. Passive LPs can park in higher-fee tiers for more protection. Active LPs can compete in lower-fee tiers for more volume. The market decides where liquidity goes.

## How It Works

Each STAMM pool contains **8 stratified tiers**, each functioning as an independent constant-product market:

- Every tier has its own reserves, fee rate, and LP token
- Traders choose which tier to swap through
- LPs choose which tier to provide liquidity to
- Protocol fees are extracted and redistributed inline on every swap
- Tiers auto-activate when they receive their first real deposit and auto-deactivate when all LP is withdrawn

All tiers live inside one smart contract, enabling atomic operations that aren't possible with separate pool contracts — like inline fee redistribution and aggregate price oracles.

## Key Features

| Feature | Description |
|---|---|
| **8 Stratified Tiers** | 8 independent fee tiers per asset pair, all created at bootstrap |
| **Tier P** | Protocol-managed backstop tier with 0.0001% fees |
| **Native ALGO Pools** | Full support for ALGO/ASA pairs — ALGO as asset A |
| **Two-Sided Fees** | Fee split across input and output for inline redistribution |
| **Inline Spill** | Protocol fees redistributed to weak tiers on every swap |
| **Pull Model Treasury** | Treasury claims stored in pool state, withdrawn on demand |
| **TWAP Oracle** | Inline time-weighted average price oracle across all active tiers |
| **Auto Tier Management** | Tiers auto-activate on first mint, auto-deactivate on full withdrawal |
| **Permissionless Pools** | Anyone can create a pool for any asset pair |
| **Factory Architecture** | Pools deployed and governed through a factory contract |
| **Reverse LP Registry** | Any LP token can be resolved to its pool, pair, and tier via factory box lookup |

## Tiers

All 8 tiers are created during bootstrap. Four default tiers are seeded during the initial liquidity deposit (`seed_and_mint`). The remaining four can be seeded individually via `seed_tier`.

| Tier | Index | Fee Rate | Seeding |
|---|---|---|---|
| Tier 0 | 0 | 0.01% (1 bps) | `seed_tier` |
| Tier 1 | 1 | 0.03% (3 bps) | `seed_tier` |
| Tier 2 | 2 | 0.1% (10 bps) | Default (`seed_and_mint`) |
| Tier 3 | 3 | 0.3% (30 bps) | Default (`seed_and_mint`) |
| Tier 4 | 4 | 1% (100 bps) | Default (`seed_and_mint`) |
| Tier 5 | 5 | 3% (300 bps) | `seed_tier` |
| Tier 6 | 6 | 5% (500 bps) | `seed_tier` |
| Tier P | 7 | 0.0001% (1 ppm) | Default (`seed_and_mint`) |

Tier P is a protocol-managed backstop tier with near-zero fees. It is not open to public LPs — liquidity is built through the inline spill mechanism.

## Architecture Overview

```
                    PoolFactory
                   /     |     \
              Pool A   Pool B   Pool C
             /  |  \    \   \
    Tier0  Tier1  ...  Tier6  TierP
    0.01%  0.03%       5%    0.0001%
```

Each pool is a standalone `TieredAMM` contract deployed by the `PoolFactory`. The factory remains governor of all pools. Treasury claims are stored in pool state and withdrawn by the admin via factory proxy methods.

## Pool Creation Flow

```
1. Factory.create_pool(asset_a, asset_b)
   -> Deploys pool, funds MBR, bootstraps (creates 8 LP tokens)
   -> Registers pair in factory box storage

2. Factory.register_pool_lps(pool)
   -> Writes 8 reverse LP lookup boxes on factory
   -> Marks pool as registered

3. Pool.seed_and_mint(amounts, tier)
   -> Seeds default 4 tiers (P, 2, 3, 4) with 1 micro each
   -> Adds real liquidity to the chosen tier

4. (Optional) Pool.seed_tier(tier_index)
   -> Seeds a non-default tier (0, 1, 5, or 6) with 1 micro each
```

## Documentation

Comprehensive documentation organized by topic. See [docs/README.md](docs/README.md) for full navigation.

### Core Concepts
- **[Architecture](docs/core/architecture.md)** — System design, contract roles, and deployment flow
- **[Tiers](docs/core/tiers.md)** — Tier mechanics, Tier P, and how liquidity stratification works

### Features
- **[Swaps & Liquidity](docs/features/swaps-and-liquidity.md)** — Swap execution, minting modes, and burn mechanics
- **[Fee Engine](docs/features/fee-engine.md)** — Two-sided fees, inline spill, and treasury claims
- **[Oracle](docs/features/oracle.md)** — TWAP price oracle design and consumption

### Reference
- **[Indexing & Lookups](docs/reference/indexing.md)** — Pool, pair, and LP token resolution via factory registries
- **[Security](docs/reference/security.md)** — Safety checks, invariants, and protections
- **[Glossary](docs/reference/glossary.md)** — Key terms and definitions

### Testing
- **[Tests Overview](docs/tests/README.md)** — Integration and fuzz testing methodology and results

## Status

STAMM is under active development. The core contracts compile and are functional. Interfaces, parameters, and mechanics are subject to change.

## Built With

- [Algorand](https://algorand.co) — L1 blockchain
- [PuyaPy](https://github.com/algorandfoundation/puya) — Algorand Python smart contract compiler
- [AlgoKit](https://developer.algorand.org/algokit/) — Development toolkit

## Notice

**This repository is proprietary and for viewing purposes only.**

This code is not currently open-source, though we may consider open-sourcing it in the future. See [CONTRIBUTING.md](CONTRIBUTING.md) for details on our policy regarding external contributions and [LICENSE](LICENSE) for copyright information.

If you discover bugs or have questions, please open an issue.

### Early Access to Development Tools

If you would like access to **hog_py_sdk** for testing or development purposes prior to our TestNet launch, please contact us:

- **Email**: [LiquiHog@gmail.com](mailto:LiquiHog@gmail.com)
- **X (Twitter)**: [@LiquiHog](https://x.com/LiquiHog)

---

**Copyright © 2026 HOG. All Rights Reserved.**

*STAMM is a component of the LiquiHog project.*
