# Registry Contract

The RegistryContract is STAMM's on-chain directory for pair and LP lookups. It stores pair->pool and LP->(pool,pair,tier) mappings in box storage.

---

## Role

- Holds canonical lookup data used by SDKs, wallets, and indexers.
- Uses a single authorized `writer` (factory app address) for registrations.
- Verifies MBR funding inline through payment arguments on write methods.

## State

| Key | Type | Description |
|---|---|---|
| `admin` | bytes | Registry admin address |
| `pending_admin` | bytes | Proposed admin address |
| `propose_ts` | uint64 | Admin-transfer proposal timestamp (72-hour timelock) |
| `box_flat_mbr` | uint64 | Cached box base MBR in uALGO (default 2500) |
| `box_byte_mbr` | uint64 | Cached per-byte MBR in uALGO (default 400) |
| `writer` | bytes | Authorized writer address |

## Box Storage

| Box Key | Value | Description |
|---|---|---|
| `p + itob(min_asset_id) + itob(max_asset_id)` | bytes(8) | Pool app ID for an asset pair |
| `l + itob(lp_asset_id)` | bytes(32) | `pool_app_id + asset_a_id + asset_b_id + tier_index` |

## ABI Methods

### Write Methods (writer only)

| Method | Description |
|---|---|
| `register_pair(pay,uint64,uint64,uint64)void` | Register pair->pool mapping with MBR payment validation |
| `register_lps(pay,uint64)void` | Register 6 reverse LP boxes for a pool with MBR payment validation |

### Read Methods

| Method | Description |
|---|---|
| `get_pool(uint64,uint64)uint64` | Resolve pair->pool |
| `get_lp_info(uint64)byte[]` | Resolve LP->pool/pair/tier |

### Admin Methods

| Method | Description |
|---|---|
| `set_writer(address)void` | Set writer address (admin only) |
| `propose_admin(address)void` | Propose admin transfer (72-hour timelock) |
| `accept_admin()void` | Pending admin accepts after timelock |
| `cancel_admin_proposal()void` | Cancel pending admin transfer |
| `set_box_mbr_rates(uint64,uint64)void` | Update cached box MBR rates |

## Security Properties

- Writer-only registration for pair and LP records.
- `register_lps` validates that the target pool creator matches `writer`.
- One-pool-per-pair enforced by pair-box uniqueness.
- No freeze mode; update/delete remain admin-gated via `on_delete_or_update`.
