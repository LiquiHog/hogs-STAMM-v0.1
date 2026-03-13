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

Pool creation is **permissionless** — anyone can create a pool for a new asset pair by paying the required minimum balance. Only one pool per asset pair is allowed.

### TieredAMM (Pool)

Each pool is an independent smart contract managing 8 fee tiers for a single asset pair. The pool handles all trading operations:

- Swaps (standard and price-limited), mints (standard, hybrid, single-sided), and burns (standard, single-sided)
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
   - Bootstraps pool (opts into assets, creates all 8 LP tokens with fee rates)
   - Registers pair in box storage

3. Register LP boxes: Factory.register_pool_lps(pool)
   - Reads 8 LP asset IDs from pool global state
   - Writes 8 reverse LP lookup boxes on the factory
   - Marks pool as registered (sets registered = 1 on pool)

4. Seed and add liquidity: Pool.seed_and_mint(amounts, tier)
   - Requires registered = 1
   - Seeds default 4 tiers (P, 2, 3, 4) with 1 micro of each asset
   - Adds real liquidity to the chosen tier

5. (Optional) Seed additional tiers: Pool.seed_tier(tier_index)
   - Seeds a non-default tier (0, 1, 5, or 6) with 1 micro of each asset
```

## State Layout

### Factory State

| Key | Type | Description |
|---|---|---|
| `admin` | bytes | Protocol admin address |
| Box: `p` + itob(min_asset_id) + itob(max_asset_id) | bytes(8) | Pool app ID for the pair |
| Box: `l` + itob(lp_asset_id) | bytes(32) | pool_app_id + asset_a_id + asset_b_id + tier_index |

### Pool State (per tier)

Each tier `t` uses keys prefixed with its character (`0`-`6` for standard, `p` for Tier P):

| Key Pattern | Type | Description |
|---|---|---|
| `t{c}_ra` | uint64 | Reserve of Asset A |
| `t{c}_rb` | uint64 | Reserve of Asset B |
| `t{c}_lp` | uint64 | Total LP token supply |
| `t{c}_fb` | uint64 | Fee rate in basis points |
| `t{c}_la` | uint64 | LP token ASA ID |
| `t{c}_tl` | uint64 | Treasury LP claim |

### Pool State (global)

| Key | Type | Description |
|---|---|---|
| `asset_a` | uint64 | Asset A ID (0 for ALGO) |
| `asset_b` | uint64 | Asset B ID |
| `governor` | bytes | Governor address (factory) |
| `tr_a`, `tr_b` | uint64 | Treasury asset claims |
| `tier_mask` | uint64 | Bits 0-7: active tier flags |
| `agg_ra`, `agg_rb` | uint64 | Aggregate reserves across active tiers |
| `twap_ca_hi`, `twap_ca_lo` | uint64 | TWAP accumulator A (128-bit) |
| `twap_cb_hi`, `twap_cb_lo` | uint64 | TWAP accumulator B (128-bit) |
| `twap_ts` | uint64 | TWAP last update timestamp |
| `opup_app_id` | uint64 | OpUp budget helper app ID |
| `registered` | uint64 | LP box registration flag (0 or 1) |

Total: 8x6 + 14 ints + 1 bytes = 62 uints + 1 bytes (0 spare slots within AVM max of 64).

## ABI Methods

### Pool — Trading

| Method | Description |
|---|---|
| `swap(txn,uint64,uint64,uint64,uint64)void` | Standard single-tier swap |
| `swap_limit(txn,uint64,uint64,uint64,uint64,uint64,uint64)void` | Price-limited partial swap with refund |
| `mint(txn,axfer,uint64,uint64,uint64,uint64,uint64)void` | Two-sided LP deposit |
| `mint_hybrid(txn,axfer,uint64,uint64,uint64,uint64,uint64)void` | Any-ratio LP deposit (excess swapped) |
| `mint_single(txn,uint64,uint64,uint64,uint64,uint64)void` | Single-asset LP deposit |
| `burn(axfer,uint64,uint64,uint64,uint64,uint64,uint64)void` | Two-sided LP withdrawal |
| `burn_single(axfer,uint64,uint64,uint64,uint64,uint64,uint64)void` | Single-asset LP withdrawal |

### Pool — Setup & Admin (governor only)

| Method | Description |
|---|---|
| `bootstrap(uint64,uint64)void` | Initialize pool with assets and create 8 LP tokens |
| `seed_and_mint(txn,axfer,uint64,uint64,uint64,uint64,uint64)void` | Seed default tiers + add initial liquidity |
| `seed_tier(txn,axfer,uint64,uint64,uint64)void` | Seed a single non-default tier |
| `set_opup_app(uint64)void` | Set OpUp helper app ID |
| `set_registered()void` | Mark pool as registered (called by factory) |
| `withdraw_lp(uint64,uint64,address,uint64)void` | Withdraw treasury LP claims |
| `withdraw_assets(uint64,uint64,uint64,uint64,address)void` | Withdraw treasury asset claims |

### Factory

| Method | Description |
|---|---|
| `create_pool(pay,uint64,uint64)uint64` | Deploy and bootstrap a new pool |
| `register_pool_lps(pay,uint64)void` | Write 8 reverse LP boxes + mark pool registered |
| `get_pool(uint64,uint64)uint64` | Look up pool app ID for an asset pair |
| `get_lp_info(uint64)byte[]` | Look up pool info for an LP asset ID |
| `set_admin(address)void` | Transfer factory admin |
| `set_pool_opup(uint64,uint64)void` | Set OpUp helper on a pool |
| `withdraw_pool_lp(uint64,uint64,uint64,address,uint64)void` | Withdraw treasury LP from a pool |
| `withdraw_pool_assets(uint64,uint64,uint64,uint64,uint64,address)void` | Withdraw treasury assets from a pool |

## Design Decisions

**Single contract per pool**: All tiers live in one contract. This enables atomic cross-tier operations (inline spill, aggregate oracle) that would require complex group transactions if tiers were separate contracts.

**All 8 tiers at bootstrap**: Every pool starts with all 8 LP tokens created upfront. This avoids dynamic tier management complexity and ensures the reverse LP registry can be populated immediately after pool creation. Tiers start unseeded and auto-activate on first real deposit.

**Factory as governor**: Pools cannot self-govern. This ensures consistent admin policy across all pools and prevents individual pool compromise from affecting the protocol.

**Pool ownership verification**: All factory proxy methods verify that the target pool was created by the factory, preventing interaction with rogue contracts.

**Permissionless pool creation**: Anyone can create a pool by paying the MBR. The factory enforces one-pool-per-pair and handles all setup atomically.

**Box storage for registries**: Asset pair lookups and reverse LP lookups use box storage, keeping the factory's global state minimal and allowing unlimited pool count.

**Reverse LP registry**: Factory boxes map each LP asset ID to its pool, pair, and tier. This enables any LP token to be resolved to its context in a single box read, supporting wallets and indexers.

**Two-step creation flow**: Pool creation and LP box registration are separate steps because LP asset IDs aren't known at transaction construction time (they're created during bootstrap). The `registered` flag gates `seed_and_mint` to ensure boxes are written before liquidity flows.

**Pull model treasury**: Treasury claims are stored in pool state rather than transferred to a separate contract. The admin withdraws on demand via factory proxy methods. This eliminates opt-in coordination and reduces inner transactions.

**Native ALGO support**: ALGO pools use Payment transactions for A-side deposits and the contract branches on `asset.id == 0` for outgoing transfers. This avoids wrapper tokens and gives users a native experience.

**Value encoding**: All ABI methods use PuyaPy's default value encoding (asset params as `uint64`, accounts as `address`). This naturally supports ALGO (asset id 0) without foreign array limitations.
