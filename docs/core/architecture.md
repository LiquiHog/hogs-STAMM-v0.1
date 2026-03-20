# Architecture

STAMM uses four ARC-4 contracts on Algorand:

- `PoolFactory` for deployment and registration
- `AdminContract` for governor operations and treasury withdrawals
- `RegistryContract` for pair/LP lookup data
- `STAMM` pool contracts (one pool per asset pair)

All contracts use value encoding (`uint64` asset IDs, `address` accounts).

## Contract Roles

### PoolFactory

Factory responsibilities:

- Deploy new `STAMM` pool apps from template boxes (`tpl`, `tpc`)
- Bootstrap pools and create all 6 LP assets
- Register pair and LP lookups via `RegistryContract`
- Transfer pool governor to `AdminContract`
- Manage creation mode (`paused`, `admin-only`, `permissionless`)
- Manage protocol references (admin contract and registry app IDs)
- Maintain cached box MBR rates via `set_box_mbr_rates`

Operational controls:

- Governance changes use 7-day timelocks.
- `freeze_factory` blocks template management (`init_template`, `upload_template`) until unfreeze.
- Update/delete are permanently blocked by `on_delete_or_update -> op.err()`.

### AdminContract

Admin contract responsibilities:

- Permanent pool governor after registration
- Timelocked admin/treasury/governor-migration operations
- Treasury withdrawals through proxy calls to pool governor methods
- Guardian-assisted cancellation of queued governance actions

Operational controls:

- 7-day timelocks for admin transfer, treasury transfer, and governor migration.
- Roles: `admin`, `treasury`, `guardian`.
- Update/delete are permanently blocked by `on_delete_or_update -> op.err()`.

### RegistryContract

Registry responsibilities:

- Store pair lookup boxes (`p + min/max asset IDs` -> pool app ID)
- Store reverse LP lookup boxes (`l + lp_asset_id` -> pool/pair/tier)
- Enforce writer-only registration
- Validate MBR payments passed to write methods
- Maintain cached box MBR rates via `set_box_mbr_rates`

Operational controls:

- Single writer (`writer`) is expected to be the factory app address.
- Admin transfer uses a 72-hour timelock.
- No freeze mode.

### STAMM Pool

Pool responsibilities:

- Swap, mint, burn, and routed execution across 6 tiers
- Per-tier reserves, LP supply, LP asset IDs, treasury LP claims
- Two-sided fee processing and inline spill redistribution
- TWAP and cumulative volume tracking
- Treasury claim accounting (`tr_a`, `tr_b`, `t{c}_tl`)

Pool admin model:

- Governor-only methods: `set_registered`, `transfer_governor`, `withdraw_lp`, `withdraw_assets`, `bootstrap`.
- Governor is factory during setup and admin contract after `register_pool_lps`.

## Deployment Flow

```text
0. Deploy RegistryContract, AdminContract, and PoolFactory
   - Set registry writer to the factory app address
   - Configure factory with registry app ID and admin contract address

1. Initialize template in factory
   - init_template(seed, approval_size, clear_program)
   - upload_template(offset, chunk) x N

2. Create pool
   - Factory.create_pool(seed, asset_a, asset_b)
   - Factory deploys STAMM pool, computes dynamic MBR funding, bootstraps pool
   - Factory calls Registry.register_pair(pay, a, b, pool_id)

3. Register pool LP boxes
   - Factory.register_pool_lps(seed, pool_app)
   - Factory calls Registry.register_lps(pay, pool_app)
   - Pool registered flag is set
   - Governor transferred to admin contract

4. Seed and launch liquidity
   - Pool.seed_and_mint(...)
   - Optional: Pool.seed_tier(...) for tiers 0 or 4
```

## State Layout

### Factory State

| Key | Type | Description |
|---|---|---|
| `admin` | bytes | Factory admin |
| `pending_admin` | bytes | Proposed admin |
| `admin_propose_ts` | uint64 | Admin proposal timestamp |
| `creation_mode` | uint64 | 0 paused, 1 admin-only, 2 permissionless |
| `admin_contract_addr` | bytes | Admin contract app address |
| `pending_ac_addr` | bytes | Proposed admin contract address |
| `ac_propose_ts` | uint64 | Admin-contract proposal timestamp |
| `registry_app_id` | uint64 | Registry app ID |
| `pending_registry_id` | uint64 | Proposed registry app ID |
| `registry_propose_ts` | uint64 | Registry proposal timestamp |
| `frozen` | uint64 | Factory template-management freeze flag |
| `unfreeze_ts` | uint64 | Unfreeze proposal timestamp |
| `box_flat_mbr` | uint64 | Cached box base MBR |
| `box_byte_mbr` | uint64 | Cached box per-byte MBR |
| Box `tpl` | bytes | STAMM approval template |
| Box `tpc` | bytes | STAMM clear template |

### Admin Contract State

See [Admin Contract - State](../reference/admin-contract.md#state).

### Registry State

See [Registry Contract - State](../reference/registry-contract.md#state).

### Pool State (per tier)

Per-tier keys (`t{c}_*`, where `c` is `0-4` or `p`):

- `t{c}_ra`, `t{c}_rb`, `t{c}_lp`, `t{c}_la`, `t{c}_tl`

### Pool State (global)

Global pool keys:

- `governor`, `asset_a`, `asset_b`, `tr_a`, `tr_b`
- `twap_ca_3..0`, `twap_cb_3..0`, `twap_ts`
- `vol_a_hi`, `vol_a_lo`, `vol_b_hi`, `vol_b_lo`
- `agg_ra`, `agg_rb`, `tier_mask`, `registered`, `version`

Factory deploys pool schema as `global_uints=52`, `global_bytes=1`.

## ABI Methods (Summary)

### Pool (STAMM)

- Trading: `swap`, `swap_limit`, `swap_smart`, `swap_routed`
- Liquidity: `mint`, `burn`, `seed_and_mint`, `seed_tier`
- Governor: `bootstrap`, `set_registered`, `transfer_governor`, `withdraw_lp`, `withdraw_assets`

### Factory

- Setup: `init_template`, `upload_template`, `set_box_mbr_rates`
- Pool ops: `create_pool`, `register_pool_lps`
- Governance: `propose_admin/accept/cancel`, `propose_admin_contract/confirm/cancel`, `propose_registry/confirm/cancel`, `freeze_factory`, `propose_unfreeze/confirm/cancel`, `set_creation_mode`

### Admin Contract

- Governance: `propose_admin/accept/cancel`, `propose_treasury/confirm/cancel`, `set_guardian`, `propose_migration/confirm_pool_migration/cancel`
- Treasury: `withdraw_pool_lp`, `withdraw_pool_assets`

### Registry

- Writes: `register_pair(pay,...)`, `register_lps(pay,...)`
- Reads: `get_pool`, `get_lp_info`
- Admin: `set_writer`, `propose_admin/accept/cancel`, `set_box_mbr_rates`

## Design Notes

- Two-step creation flow (`create_pool` then `register_pool_lps`) is required because LP asset IDs are only known after bootstrap inner transactions.
- Factory and registry compute and validate box MBR dynamically using cached protocol rates (`box_flat_mbr`, `box_byte_mbr`).
- Treasury uses a pull model: claims stay in pool state and are withdrawn by admin contract treasury role.
- Pool governor migration is timelocked at the admin-contract layer, while pool `transfer_governor` itself remains immediate for the current governor.
