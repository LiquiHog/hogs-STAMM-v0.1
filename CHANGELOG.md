# Changelog

All notable changes to the STAMM protocol will be documented here.

---

## Unreleased

### Core
- 7-tier constant-product AMM with per-tier reserves, fees, and LP tokens
- All 7 tiers created upfront during bootstrap (no governor add/remove)
- Tier P (index 6) with 1 ppm fee and no protocol extraction
- Standard swap with slippage protection and k-invariant enforcement
- Price-limited swap (`swap_limit`) with partial execution and atomic refund
- Smart-routed swap (`swap_smart`) with waterfall routing across up to 2 tiers
- Unified `mint` supporting balanced, hybrid (swap excess + mint), and single-sided (optimal swap split via 128-bit sqrt) modes
- Unified `burn` supporting proportional and single-sided (burn + internal swap) modes via `output_asset` parameter
- Bitmask-based tier active tracking (`tier_mask`)
- Auto-activate: tier becomes active when total LP > 1 (first real mint)
- Auto-deactivate: tier becomes inactive when total LP == 1 (all user LP burned)
- Sweep-on-final-burn: excess reserves sent to treasury, tier reset to seed state (1,1,1)

### Native ALGO Support
- ALGO/ASA pools with ALGO as asset A (id 0)
- Value encoding for all ABI methods (asset params as uint64, accounts as address)
- Generic transaction type for A-side deposits (Payment for ALGO, AssetTransfer for ASAs)
- `do_asset_transfer` branches on asset id: Payment for ALGO, AssetTransfer for ASAs
- MBR guard on ALGO transfers: `bal - mbr >= amount`
- Bootstrap skips ALGO opt-in (ALGO doesn't need opt-in)
- LP token naming handles ALGO (uses "ALGO" string instead of `asset_params_get`)

### Fee Engine
- Two-sided fee model: half fee on input, half on output
- 80/20 tier-retained/protocol fee split
- Inline spill: protocol fees redistributed per-swap to Tier P (10%) + 2 weakest tiers (55% + 35%)
- Weakest tier indices computed inline per-swap via O(6) scan (no cached state)
- Self-tier exclusion: spill skips the tier being operated on to prevent state conflicts
- Pull model treasury: claims stored in pool state (`tr_a`, `tr_b`, `t{c}_tl`)
- Dust from spill (amounts too small to mint LP) goes to treasury claims
- `tier_retained` floored to 1 on every swap, guaranteeing k always grows

### Oracle
- Inline TWAP oracle updated on every reserve-modifying operation
- 256-bit accumulators (4 × uint64 words per accumulator) with 2^32 fixed-point scale
- Aggregate-reserve weighted pricing across all active tiers
- 128-bit cumulative volume counters for both assets (`vol_a_hi/lo`, `vol_b_hi/lo`)

### Architecture
- PoolFactory with configurable creation mode (paused, admin-only, or permissionless)
- One pool per asset pair (enforced via box storage registry)
- Reverse LP registry: factory box storage maps any LP asset ID to its pool, pair, and tier
- Two-step pool creation: create_pool -> register_pool_lps -> seed_and_mint
- `registered` flag gates seed_and_mint (ensures LP boxes are written before liquidity)
- Two-step admin transfer: `propose_admin` -> `accept_admin` (with `cancel_propose_admin`)
- Factory verifies pool ownership on all proxy operations
- Seed payment sender validation on pool/tier creation

### Optimizations
- Shared subroutines for swap and LP mint calculations
- Old reserves passed to state updates (eliminates redundant global reads)
- Inline bootstrap checks using already-read reserves
- Aggregate reserves used for total liquidity calculations
- No external keeper required — all fee distribution is inline
- 56 uint64 + 1 bytes global state (8 spare slots within AVM max of 64)
