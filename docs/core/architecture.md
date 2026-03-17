# Architecture

STAMM uses a two-contract architecture: a **Factory** and a **Pool (TieredAMM)**. Both are deployed on Algorand as ARC-4 compliant smart contracts using value encoding (asset params as `uint64`, account params as `address`).

## Contract Roles

### PoolFactory

The factory is the entry point for the protocol. It handles:

- **Pool deployment** — compiles and deploys new `TieredAMM` instances for asset pairs
- **Pair registry** — maintains box-storage mapping of asset pairs to pool app IDs
- **Reverse LP registry** — maps any LP asset ID to its pool, pair, and tier via box storage
- **Governor proxy** — remains governor of all pools it creates, proxying admin operations
- **Treasury withdrawal** — admin withdraws treasury claims from pools via factory proxy methods

Pool creation access is controlled by a `creation_mode` setting: paused (no new pools), admin-only, or permissionless. Only one pool per asset pair is allowed. Admin transfer uses a two-step process (`propose_admin` → `accept_admin`) with a cancellation option.

### TieredAMM (Pool)

Each pool is an independent smart contract managing 7 fee tiers for a single asset pair. The pool handles all trading operations:

- Swaps (standard, price-limited, and smart-routed), mints (balanced, hybrid, and single-sided via unified entry point), and burns (proportional and single-sided via unified entry point)
- Per-tier reserve and LP token management
- Protocol fee extraction and inline redistribution
- TWAP oracle updates
- Treasury claim tracking (pull model)

The pool's governor is the factory contract. Direct admin access is not possible — all governance operations go through the factory.

Pools support both ASA/ASA pairs and ALGO/ASA pairs natively. For ALGO pools, asset A is ALGO (id 0), and A-side deposits use Payment transactions instead of AssetTransfer.

## Deployment Flow

```
1. Deploy PoolFactory

2. Create pool: Factory.create_pool(asset_a, asset_b)
   - Factory deploys TieredAMM instance
   - Funds pool with ALGO for MBR
   - Bootstraps pool (opts into assets, creates all 7 LP tokens)
   - Registers pair in box storage

3. Register LP boxes: Factory.register_pool_lps(pool)
   - Reads 7 LP asset IDs from pool global state
   - Writes 7 reverse LP lookup boxes on the factory
   - Marks pool as registered (sets registered = 1 on pool)

4. Seed and add liquidity: Pool.seed_and_mint(amounts, tier)
   - Requires registered = 1
   - Seeds default 4 tiers (P, 1, 2, 3) with 1 micro of each asset
   - Adds real liquidity to the chosen tier

5. (Optional) Seed additional tiers: Pool.seed_tier(tier_index)
   - Seeds a non-default tier (0, 4, or 5) with 1 micro of each asset
```

## State Layout

### Factory State

| Key | Type | Description |
|---|---|---|
| `admin` | bytes | Protocol admin address |
| `pending_admin` | bytes | Proposed new admin (zero until `propose_admin` is called) |
| `creation_mode` | uint64 | Pool creation access: 0 = paused, 1 = admin-only, 2 = permissionless |
| Box: `p` + itob(min_asset_id) + itob(max_asset_id) | bytes(8) | Pool app ID for the pair |
| Box: `l` + itob(lp_asset_id) | bytes(32) | pool_app_id + asset_a_id + asset_b_id + tier_index |

### Pool State (per tier)

Each tier `t` uses keys prefixed with its character (`0`-`5` for standard, `p` for Tier P):

| Key Pattern | Type | Description |
|---|---|---|
| `t{c}_ra` | uint64 | Reserve of Asset A |
| `t{c}_rb` | uint64 | Reserve of Asset B |
| `t{c}_lp` | uint64 | Total LP token supply |
| `t{c}_la` | uint64 | LP token ASA ID |
| `t{c}_tl` | uint64 | Treasury LP claim |

Fee rates are hardcoded in the contract (`get_tier_fee()`) — not stored in state.

### Pool State (global)

| Key | Type | Description |
|---|---|---|
| `asset_a` | uint64 | Asset A ID (0 for ALGO) |
| `asset_b` | uint64 | Asset B ID |
| `governor` | bytes | Governor address (factory) |
| `tr_a`, `tr_b` | uint64 | Treasury asset claims |
| `tier_mask` | uint64 | Bits 0-6: active tier flags |
| `agg_ra`, `agg_rb` | uint64 | Aggregate reserves across active tiers |
| `twap_ca_3`, `twap_ca_2`, `twap_ca_1`, `twap_ca_0` | uint64 | TWAP accumulator A (256-bit, 4 words: `_3` most significant → `_0` least significant) |
| `twap_cb_3`, `twap_cb_2`, `twap_cb_1`, `twap_cb_0` | uint64 | TWAP accumulator B (256-bit, 4 words) |
| `twap_ts` | uint64 | TWAP last update timestamp |
| `vol_a_hi`, `vol_a_lo` | uint64 | Cumulative volume Asset A (128-bit) |
| `vol_b_hi`, `vol_b_lo` | uint64 | Cumulative volume Asset B (128-bit) |
| `registered` | uint64 | LP box registration flag (0 or 1) |

Total: 7×5 + 21 ints + 1 bytes = 56 uints + 1 bytes (8 spare slots within AVM max of 64).

## ABI Methods

### Pool — Trading

| Method | Description |
|---|---|
| `swap(txn,uint64,uint64,uint64,uint64)void` | Standard single-tier swap |
| `swap_limit(txn,uint64,uint64,uint64,uint64,uint64,uint64)void` | Price-limited partial swap with refund |
| `swap_smart(txn,uint64,uint64,uint64)void` | Auto-routed swap across up to 2 tiers via waterfall routing |
| `mint(txn,axfer,uint64,uint64,uint64,uint64,uint64)void` | Unified LP deposit (balanced, hybrid, or single-sided) |
| `burn(axfer,uint64,uint64,uint64,uint64,uint64,uint64,uint64)void` | Unified LP withdrawal (proportional or single-sided via `output_asset`) |

### Pool — Setup & Admin (governor only)

| Method | Description |
|---|---|
| `bootstrap(uint64,uint64)void` | Initialize pool with assets and create 7 LP tokens |
| `seed_and_mint(txn,axfer,uint64,uint64,uint64,uint64,uint64)void` | Seed default tiers + add initial liquidity |
| `seed_tier(txn,axfer,uint64,uint64,uint64)void` | Seed a single non-default tier |
| `set_registered()void` | Mark pool as registered (called by factory) |
| `withdraw_lp(uint64,uint64,address,uint64)void` | Withdraw treasury LP claims |
| `withdraw_assets(uint64,uint64,uint64,uint64,address)void` | Withdraw treasury asset claims |

### Factory

| Method | Description |
|---|---|
| `create_pool(pay,uint64,uint64)uint64` | Deploy and bootstrap a new pool |
| `register_pool_lps(pay,uint64)void` | Write 7 reverse LP boxes + mark pool registered |
| `get_pool(uint64,uint64)uint64` | Look up pool app ID for an asset pair |
| `get_lp_info(uint64)byte[]` | Look up pool info for an LP asset ID |
| `propose_admin(address)void` | Propose a new admin (current admin only) |
| `accept_admin()void` | Accept admin role (pending admin only) |
| `cancel_admin_proposal()void` | Cancel a pending admin proposal |
| `set_creation_mode(uint64)void` | Set pool creation access mode (0=paused, 1=admin-only, 2=permissionless) |
| `withdraw_pool_lp(uint64,uint64,uint64,address,uint64)void` | Withdraw treasury LP from a pool |
| `withdraw_pool_assets(uint64,uint64,uint64,uint64,uint64,address)void` | Withdraw treasury assets from a pool |

## Design Decisions

**Single contract per pool**: All tiers live in one contract. This enables atomic cross-tier operations (inline spill, aggregate oracle) that would require complex group transactions if tiers were separate contracts.

**All 7 tiers at bootstrap**: Every pool starts with all 7 LP tokens created upfront. This avoids dynamic tier management complexity and ensures the reverse LP registry can be populated immediately after pool creation. Tiers start unseeded and auto-activate on first real deposit.

**Factory as governor**: Pools cannot self-govern. This ensures consistent admin policy across all pools and prevents individual pool compromise from affecting the protocol.

**Pool ownership verification**: All factory proxy methods verify that the target pool was created by the factory, preventing interaction with rogue contracts.

**Configurable pool creation**: Pool creation access is controlled by `creation_mode` (paused, admin-only, or permissionless). The factory enforces one-pool-per-pair and handles all setup atomically.

**Box storage for registries**: Asset pair lookups and reverse LP lookups use box storage, keeping the factory's global state minimal and allowing unlimited pool count.

**Reverse LP registry**: Factory boxes map each LP asset ID to its pool, pair, and tier. This enables any LP token to be resolved to its context in a single box read, supporting wallets and indexers.

**Two-step creation flow**: Pool creation and LP box registration are separate steps because LP asset IDs aren't known at transaction construction time (they're created during bootstrap). The `registered` flag gates `seed_and_mint` to ensure boxes are written before liquidity flows.

**Pull model treasury**: Treasury claims are stored in pool state rather than transferred to a separate contract. The admin withdraws on demand via factory proxy methods. This eliminates opt-in coordination and reduces inner transactions.

**Native ALGO support**: ALGO pools use Payment transactions for A-side deposits and the contract branches on `asset.id == 0` for outgoing transfers. This avoids wrapper tokens and gives users a native experience.

**Value encoding**: All ABI methods use PuyaPy's default value encoding (asset params as `uint64`, accounts as `address`). This naturally supports ALGO (asset id 0) without foreign array limitations.
