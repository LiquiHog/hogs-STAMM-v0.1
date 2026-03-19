# Registry Contract

The RegistryContract is a permanent on-chain directory for pool and LP discovery. It maps asset pairs to pool app IDs and LP asset IDs to their pool, pair, and tier — surviving factory replacement or deletion.

---

## Role

The registry stores all pair and LP lookup data in box storage. Only the authorized writer (the factory's application address) can register new data. The factory forwards MBR to the registry before each write. Read methods are available to anyone and remain functional even when the registry is frozen.

## State

| Key | Type | Description |
|---|---|---|
| `admin` | bytes | Registry admin address |
| `writer` | bytes | Authorized writer address (factory app address) |
| `frozen` | uint64 | Permanent freeze flag: blocks writes and writer changes |

### Box Storage

| Box Key | Value | Description |
|---|---|---|
| `p` + itob(min_asset_id) + itob(max_asset_id) | bytes(8) | Pool app ID for the pair |
| `l` + itob(lp_asset_id) | bytes(32) | pool_app_id + asset_a_id + asset_b_id + tier_index |

Pair boxes use the canonical ordering (min, max) of the two asset IDs. LP boxes store 32 bytes: the pool app ID, both asset IDs, and the tier index. This enables a single box read to resolve any LP token to its full context.

## ABI Methods

### Write Operations (writer only)

| Method | Description |
|---|---|
| `register_pair(uint64,uint64,uint64)void` | Register a new pair → pool mapping |
| `register_lps(application)void` | Register 6 reverse LP lookup boxes for a pool |

`register_pair` enforces one-pool-per-pair — attempting to register a duplicate pair fails. `register_lps` verifies the pool's creator matches the writer, preventing registration of rogue contracts.

### Read Operations (permissionless)

| Method | Description |
|---|---|
| `get_pool(uint64,uint64)uint64` | Look up pool app ID for an asset pair |
| `get_lp_info(uint64)byte[]` | Look up pool info (app ID, pair, tier) for an LP asset ID |

### Admin Operations

| Method | Description |
|---|---|
| `set_writer(address)void` | Set the authorized writer address (admin only, blocked when frozen) |
| `set_admin(address)void` | Set registry admin (admin only, works when frozen) |
| `freeze_registry()void` | Permanently freeze the registry — blocks all writes (admin only, irreversible) |

## Security Properties

- **Writer authorization** — only the factory can register new pair and LP data
- **Creator verification** — `register_lps` verifies the pool's creator matches the writer
- **One pool per pair** — enforced by box storage; duplicate registrations fail
- **Irreversible freeze** — `freeze_registry` permanently blocks all writes; reads remain functional
- **Admin independence** — admin transfer works even when frozen, preventing the contract from being orphaned
- **Permanence** — registry data survives factory replacement or deletion; pool discovery is always available
