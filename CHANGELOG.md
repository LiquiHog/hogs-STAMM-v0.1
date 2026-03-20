# Changelog

All notable changes to the STAMM protocol will be documented here.

---

## Unreleased

### Core
- 6-tier constant-product AMM with per-tier reserves, fees, and LP tokens
- All 6 tiers created upfront during bootstrap (no governor add/remove)
- Tier P (index 5) with 1 ppm fee and no protocol extraction
- Standard swap with slippage protection and k-invariant enforcement
- Price-limited swap (`swap_limit`) with partial execution and atomic refund
- Smart-routed swap (`swap_smart`) with waterfall routing across up to 3 tiers
- Caller-directed routed swap (`swap_routed`) with 1-6 explicit (tier, amount) legs packed as N x 9 bytes
- Unified `mint` supporting balanced, hybrid, and single-sided modes
- Unified `burn` supporting proportional and single-sided modes via `output_asset`
- Bitmask-based tier active tracking (`tier_mask`)
- Auto-activate when total LP > 1, auto-deactivate when total LP == 1
- Sweep-on-final-burn sends excess reserves to treasury claims and resets tier to seed state

### Native ALGO Support
- ALGO/ASA pools with ALGO as `asset_a` (`id=0`)
- Value encoding for all ABI methods (asset params as uint64, accounts as address)
- Payment handling for ALGO deposits/withdrawals and AssetTransfer for ASAs
- MBR guard on outgoing ALGO transfers

### Fee Engine
- Two-sided fee model (input and output side)
- 80/20 tier-retained/protocol fee split for standard tiers
- Inline spill distribution to Tier P and weak standard tiers
- Pull-model treasury claims (`tr_a`, `tr_b`, `t{c}_tl`)

### Oracle
- Inline TWAP updates on reserve-modifying operations
- 256-bit TWAP accumulators and 128-bit volume counters
- Aggregate-reserve weighted pricing across active tiers

### Architecture

#### Contracts
- Four-contract architecture: PoolFactory, AdminContract, RegistryContract, and STAMM pools
- Factory deploys pools and hands governor to admin contract after LP registration
- Registry owns pair and LP lookup storage

#### Admin Contract
- 7-day timelocks for admin transfer, treasury transfer, and governor migration
- Dedicated `treasury` role for withdrawals and `guardian` role for cancellation veto
- Treasury withdrawal proxy methods: `withdraw_pool_lp`, `withdraw_pool_assets`
- Update/delete blocked by `on_delete_or_update -> op.err()`

#### Registry Contract
- Single writer model for pair and LP registrations
- Admin transfer timelock: 72 hours (259,200 seconds)
- Write signatures include payment args: `register_pair(pay,...)`, `register_lps(pay,...)`
- Cached box MBR rates with `set_box_mbr_rates(uint64,uint64)`

#### Factory
- 7-day timelocks for admin transfer, admin-contract changes, registry changes, and unfreeze
- Instant `freeze_factory` plus timelocked unfreeze
- Cached box MBR rates with `set_box_mbr_rates(uint64,uint64)`
- Dynamic pool/registry funding using current protocol min-balance values plus cached box MBR math
- Update/delete blocked by `on_delete_or_update -> op.err()`

#### Pool Setup
- Two-step setup: `create_pool` -> `register_pool_lps` -> `seed_and_mint`
- `registered` gate ensures reverse LP boxes are written before liquidity entry

### Optimizations
- Shared math/routing helpers and reduced redundant global reads
- No external keeper required for fee redistribution
- STAMM pool deployment target: 52 uint64 + 1 byte-slice global schema
- Routing table (RT) box keeps per-tier fee-adjusted scores for `swap_smart`
