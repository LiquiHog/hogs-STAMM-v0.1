# Architecture

STAMM uses a four-contract architecture: a **Factory**, an **Admin Contract**, a **Registry**, and **Pools (TieredAMM)**. All are deployed on Algorand as ARC-4 compliant smart contracts using value encoding (asset params as `uint64`, account params as `address`).

## Contract Roles

### PoolFactory

The factory is a lightweight deployment tool for the protocol. It handles:

- **Pool deployment** — compiles and deploys new `TieredAMM` instances for asset pairs
- **Registry forwarding** — registers pairs and LP lookups in the separate RegistryContract (forwarding MBR)
- **Governor handoff** — transfers governor authority to the admin contract after LP registration
- **Admin contract management** — tracks and updates the admin contract address (7-day timelock)
- **Registry management** — tracks and updates the registry contract reference (7-day timelock)
- **Freeze system** — instant factory freeze blocks template/update/delete; unfreeze requires 7-day timelock

Pool creation access is controlled by a `creation_mode` setting: paused (no new pools), admin-only, or permissionless. Only one pool per asset pair is allowed (enforced by the registry). Admin transfer uses a two-step process (`propose_admin` → `accept_admin`) with a 7-day timelock and cancellation option.

The factory holds no pair or LP box storage — all registries live in the RegistryContract. The factory cannot call governor-protected methods on pools after the handoff.

### AdminContract

The admin contract is the permanent governor of all registered pools. It handles:

- **Treasury withdrawals** — proxy methods to withdraw LP tokens and dust assets from pool state
- **Admin transfer** — two-step process with 72-hour timelock (`propose_admin` → `accept_admin`). See [Governance](governance.md#timelocks) for details
- **Freeze** — 72-hour timelock to freeze the admin contract itself; irreversible; blocks update/delete but treasury withdrawals and admin transfer remain functional
- **Governor migration** — transfer pools to a new governor address with 72-hour timelock (`propose_migration` → `confirm_pool_migration`, one pool at a time)

Even if the factory is deleted or replaced, the admin contract retains governor authority over all existing pools.

### RegistryContract

The registry is a permanent on-chain directory for pool/LP discovery. It stores:

- **Pair registry** — box-storage mapping of asset pairs to pool app IDs
- **Reverse LP registry** — maps any LP asset ID to its pool, pair, and tier

The authorized writer (factory application address) can register new pairs and LP lookups. The factory sends MBR as a separate inner payment before calling write methods.

The registry has its own freeze mechanism — once frozen, no writes or writer changes are possible. Read methods (`get_pool`, `get_lp_info`) remain functional permanently. The registry survives factory replacement or deletion.

### TieredAMM (Pool)

Each pool is an independent smart contract managing 6 fee tiers for a single asset pair. The pool handles all trading operations:

- Swaps (standard, price-limited, and smart-routed), mints (balanced, hybrid, and single-sided via unified entry point), and burns (proportional and single-sided via unified entry point)
- Per-tier reserve and LP token management
- Protocol fee extraction and inline redistribution
- TWAP oracle updates
- Treasury claim tracking (pull model)

After registration, the pool's governor is transferred from the factory to the admin contract. The admin contract is the permanent governor of all registered pools and handles treasury withdrawals and other governor-only operations.

Pools support both ASA/ASA pairs and ALGO/ASA pairs natively. For ALGO pools, asset A is ALGO (id 0), and A-side deposits use Payment transactions instead of AssetTransfer.

## Deployment Flow

```
0. Deploy RegistryContract, AdminContract, and PoolFactory
   - Set registry writer to factory address
   - Set factory's registry_app_id and admin_contract_addr

1. Initialize template:
    Factory.init_template(approval_size, clear_program)
    Factory.upload_template(offset, chunk) × N
    - Stores compiled TieredAMM approval/clear programs in factory box storage
    - Required before any pool can be created

2. Create pool: Factory.create_pool(asset_a, asset_b)
   - Factory deploys TieredAMM instance
   - Funds pool with ALGO for MBR
   - Bootstraps pool (opts into assets, creates all 6 LP tokens)
   - Forwards MBR to registry and registers pair via RegistryContract.register_pair

3. Register LP lookups: Factory.register_pool_lps(pool)
   - Forwards MBR to registry
   - Calls RegistryContract.register_lps to write 6 reverse LP lookup boxes
   - Marks pool as registered (sets registered = 1 on pool)
   - Transfers governor authority from factory to admin contract

4. Seed and add liquidity: Pool.seed_and_mint(amounts, tier) — tier must be 1, 2, or 3
   - Requires registered = 1
   - Seeds default 4 tiers (P, 1, 2, 3) with 1 micro of each asset
   - Adds real liquidity to the chosen tier

5. (Optional) Seed additional tiers: Pool.seed_tier(tier_index)
   - Seeds a non-default tier (0 or 4) with 1 micro of each asset
```

## State Layout

### Factory State

| Key | Type | Description |
|---|---|---|
| `admin` | bytes | Factory admin address |
| `pending_admin` | bytes | Proposed new admin (zero until `propose_admin` is called) |
| `admin_propose_ts` | uint64 | Admin proposal timestamp for 7-day timelock |
| `creation_mode` | uint64 | Pool creation access: 0 = paused, 1 = admin-only, 2 = permissionless |
| `admin_contract_addr` | bytes | Application address of the admin/governor contract |
| `pending_ac_addr` | bytes | Proposed new admin contract address |
| `ac_propose_ts` | uint64 | Admin contract proposal timestamp for 7-day timelock |
| `registry_app_id` | uint64 | Registry contract application ID |
| `pending_registry_id` | uint64 | Proposed new registry app ID |
| `registry_propose_ts` | uint64 | Registry proposal timestamp for 7-day timelock |
| `frozen` | uint64 | Freeze flag: blocks template/update/delete |
| `unfreeze_ts` | uint64 | Unfreeze proposal timestamp for 7-day timelock |
| Box: `tpl` | bytes | TieredAMM approval program (template) |
| Box: `tpc` | bytes | TieredAMM clear program (template) |

### Admin Contract State

See [Admin Contract — State](../reference/admin-contract.md#state) for the full state table.

### Registry State

See [Registry Contract — State](../reference/registry-contract.md#state) for the full state table (including box storage format).

### Pool State (per tier)

Each tier `t` uses keys prefixed with its character (`0`-`4` for standard, `p` for Tier P):

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
| `governor` | bytes | Governor address (admin contract after registration) |
| `tr_a`, `tr_b` | uint64 | Treasury asset claims |
| `tier_mask` | uint64 | Bits 0-5: active tier flags |
| `agg_ra`, `agg_rb` | uint64 | Aggregate reserves across active tiers |
| `twap_ca_3`, `twap_ca_2`, `twap_ca_1`, `twap_ca_0` | uint64 | TWAP accumulator A (256-bit, 4 words: `_3` most significant → `_0` least significant) |
| `twap_cb_3`, `twap_cb_2`, `twap_cb_1`, `twap_cb_0` | uint64 | TWAP accumulator B (256-bit, 4 words) |
| `twap_ts` | uint64 | TWAP last update timestamp |
| `vol_a_hi`, `vol_a_lo` | uint64 | Cumulative volume Asset A (128-bit) |
| `vol_b_hi`, `vol_b_lo` | uint64 | Cumulative volume Asset B (128-bit) |
| `registered` | uint64 | LP box registration flag (0 or 1) |

Total: 6×5 + 21 ints + 1 bytes = 51 uints + 1 bytes used. 13 slots remain below the AVM maximum of 64 uint64 globals.

## ABI Methods

### Pool — Trading

| Method | Description |
|---|---|
| `swap(txn,uint64,uint64,uint64,uint64)void` | Standard single-tier swap |
| `swap_limit(txn,uint64,uint64,uint64,uint64,uint64,uint64)void` | Price-limited partial swap with refund |
| `swap_smart(txn,uint64,uint64,uint64)void` | Auto-routed swap across up to 3 tiers via waterfall routing |
| `swap_routed(txn,uint64,uint64,byte[],uint64)void` | Caller-directed routed swap across 1-6 explicit (tier, amount) legs |
| `mint(txn,axfer,uint64,uint64,uint64,uint64,uint64)void` | Unified LP deposit (balanced, hybrid, or single-sided) |
| `burn(axfer,uint64,uint64,uint64,uint64,uint64,uint64,uint64)void` | Unified LP withdrawal (proportional or single-sided via `output_asset`) |

### Pool — Setup (permissionless)

| Method | Description |
|---|---|
| `seed_and_mint(txn,axfer,uint64,uint64,uint64,uint64,uint64)void` | Seed default tiers + add initial liquidity (tier must be 1, 2, or 3) |
| `seed_tier(txn,axfer,uint64,uint64,uint64)void` | Seed a single non-default tier |

### Pool — Admin (governor only)

| Method | Description |
|---|---|
| `bootstrap(uint64,uint64)void` | Initialize pool with assets and create 6 LP tokens |
| `set_registered()void` | Mark pool as registered (called by factory) |
| `transfer_governor(address)void` | Transfer governor role to a new address (current governor only) |
| `withdraw_lp(uint64,uint64,address,uint64)void` | Withdraw treasury LP claims |
| `withdraw_assets(uint64,uint64,uint64,uint64,address)void` | Withdraw treasury asset claims |

### Factory

| Method | Description |
|---|---|
| `init_template(pay,uint64,byte[])void` | Initialize template storage boxes (admin only) |
| `upload_template(uint64,byte[])void` | Upload approval program chunk (admin only) |
| `create_pool(pay,uint64,uint64)uint64` | Deploy and bootstrap a new pool, register pair in registry |
| `register_pool_lps(pay,uint64)void` | Register LP lookups in registry + mark pool registered + transfer governor |
| `propose_admin(address)void` | Propose a new factory admin (7-day timelock) |
| `accept_admin()void` | Accept factory admin role after timelock (pending admin only) |
| `cancel_admin_proposal()void` | Cancel a pending factory admin proposal |
| `set_creation_mode(uint64)void` | Set pool creation access mode (0=paused, 1=admin-only, 2=permissionless) |
| `propose_admin_contract(application)void` | Propose a new admin contract (7-day timelock) |
| `confirm_admin_contract()void` | Confirm admin contract change after timelock |
| `cancel_admin_contract_proposal()void` | Cancel a pending admin contract proposal |
| `propose_registry(application)void` | Propose a new registry contract (7-day timelock) |
| `confirm_registry()void` | Confirm registry change after timelock |
| `cancel_registry_proposal()void` | Cancel a pending registry proposal |
| `freeze_factory()void` | Freeze factory — blocks template/update/delete (instant) |
| `propose_unfreeze()void` | Propose unfreezing the factory (7-day timelock) |
| `confirm_unfreeze()void` | Confirm and execute factory unfreeze after timelock |
| `cancel_unfreeze()void` | Cancel a pending unfreeze proposal |

### Admin Contract

See [Admin Contract — ABI Methods](../reference/admin-contract.md#abi-methods) for the full method table.

### Registry

See [Registry Contract — ABI Methods](../reference/registry-contract.md#abi-methods) for the full method table.

## Design Decisions

**Single contract per pool**: All tiers live in one contract. This enables atomic cross-tier operations (inline spill, aggregate oracle) that would require complex group transactions if tiers were separate contracts.

**All 6 tiers at bootstrap**: Every pool starts with all 6 LP tokens created upfront. This avoids dynamic tier management complexity and ensures the reverse LP registry can be populated immediately after pool creation. Tiers start unseeded and auto-activate on first real deposit.

**Admin contract as governor**: After registration, the factory transfers each pool's governor role to the admin contract. The admin contract becomes the permanent governor of all pools, handling treasury withdrawals and governance operations. Even if the factory is deleted or replaced, the admin contract retains authority. This separates deployment (factory) from ongoing governance (admin contract).

**Pool ownership verification**: The factory verifies that the target pool was created by the factory (via `app_creator` check) before performing registration. The registry also verifies pool creator matches the writer (factory) during LP registration.

**Configurable pool creation**: Pool creation access is controlled by `creation_mode` (paused, admin-only, or permissionless). The registry enforces one-pool-per-pair. The factory handles all setup atomically.

**Separate registry contract**: Pair and LP registries live in a dedicated RegistryContract rather than in the factory. This means registries survive factory replacement or deletion — pool discovery remains functional even if the factory is upgraded. The registry can be permanently frozen to guarantee immutable lookups.

**Reverse LP registry**: Registry boxes map each LP asset ID to its pool, pair, and tier. This enables any LP token to be resolved to its context in a single box read, supporting wallets and indexers.

**Two-step creation flow**: Pool creation and LP box registration are separate steps because LP asset IDs aren't known at transaction construction time (they're created during bootstrap). The `registered` flag gates `seed_and_mint` to ensure boxes are written before liquidity flows.

**Pull model treasury**: Treasury claims are stored in pool state rather than transferred to a separate contract. The admin contract (as governor) withdraws on demand via `withdraw_pool_lp` and `withdraw_pool_assets`. This eliminates opt-in coordination and reduces inner transactions.

**7-day factory timelocks**: Factory admin transfer, admin contract changes, registry changes, and unfreezing all require a 7-day timelock (604,800 seconds). Freezing is instant to allow rapid response to emergencies.

**72-hour admin contract timelocks**: Admin contract admin transfer, freeze, and governor migration use a 72-hour timelock (259,200 seconds). The shorter timelock reflects the higher trust level of the admin contract (it's already the governor of all pools).

**Admin contract freeze**: The admin contract can be frozen after a 72-hour timelock. Freeze is irreversible — it permanently blocks updates and deletions of the admin contract code. Treasury withdrawals and admin transfer remain functional. Governor migration also works when frozen, allowing pools to be moved to a new governor if needed.

**Factory freeze system**: The factory can be frozen instantly by the admin, blocking template uploads, contract updates, and deletions. Pool creation and LP registration remain functional during a freeze. Unfreezing requires a 7-day timelock.

**Registry freeze**: The registry can be permanently frozen by its admin. Once frozen, no writes or writer changes are possible — the registry becomes an immutable on-chain directory. Read methods continue to work. Admin transfer remains functional.

**Native ALGO support**: ALGO pools use Payment transactions for A-side deposits and the contract branches on `asset.id == 0` for outgoing transfers. This avoids wrapper tokens and gives users a native experience.

**Value encoding**: All ABI methods use PuyaPy's default value encoding (asset params as `uint64`, accounts as `address`). This naturally supports ALGO (asset id 0) without foreign array limitations.
