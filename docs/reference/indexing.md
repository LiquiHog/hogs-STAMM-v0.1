# Indexing & Lookups

STAMM's registry contract provides permanent on-chain registries for resolving pools, assets, and LP tokens. Two box-storage registries cover both directions: finding a pool from an asset pair, and finding a pool from an LP token. The registry is a standalone contract that survives factory replacement, ensuring lookup data is never lost.

---

## Registries

### Pair Registry — "I have two assets, which pool trades them?"

Every pool is registered by its asset pair. Given two asset IDs, a single box read returns the pool app ID.

```
Key:   "p" + itob(min_asset_id) + itob(max_asset_id)    (17 bytes)
Value: itob(pool_app_id)                                  (8 bytes)
```

Asset IDs are sorted (lower first) so the lookup is canonical regardless of which order the caller provides them.

Written during `create_pool` (factory forwards MBR to the registry and calls `register_pair`). One entry per pool. Only one pool per asset pair is allowed.

### Reverse LP Registry — "I have an LP token, what pool/pair/tier is it?"

Every LP token ASA is registered by its asset ID. Given an LP asset ID, a single box read returns the full context.

```
Key:   "l" + itob(lp_asset_id)                            (9 bytes)
Value: itob(pool_app_id)                                   \
     + itob(asset_a_id)                                     |  32 bytes
     + itob(asset_b_id)                                     |
     + itob(tier_index)                                    /
```

All values are big-endian uint64. The tier index is 0-5.

Written during `register_pool_lps` (factory forwards MBR to the registry and calls `register_lps`). Six entries per pool (one per tier).

---

## Lookup Flows

### Asset Pair → Pool

```
Input:  asset_a_id, asset_b_id
Method: get_pool(uint64, uint64) → uint64
Result: pool_app_id
```

Or via direct box read with key `"p" + itob(min(a,b)) + itob(max(a,b))`.

### Pool → LP Tokens

Each pool stores 6 LP asset IDs in global state under keys `t{c}_la` where `{c}` is the tier character (`0`-`4`, `p`):

```
Pool global state:
  t0_la → LP asset ID for tier 0
  t1_la → LP asset ID for tier 1
  t2_la → LP asset ID for tier 2
  t3_la → LP asset ID for tier 3
  t4_la → LP asset ID for tier 4
  tp_la → LP asset ID for Tier P
```

Read these directly from the pool's global state via algod.

### LP Token → Pool, Pair, Tier

```
Input:  lp_asset_id
Method: get_lp_info(uint64) → byte[]
Result: 32 bytes → pool_app_id(8) + asset_a_id(8) + asset_b_id(8) + tier_index(8)
```

Or via direct box read with key `"l" + itob(lp_asset_id)`.

### Complete Resolution Example

A wallet holding an unknown LP token can resolve everything in two reads:

1. **LP → context**: Read reverse LP box → get pool_app_id, asset_a_id, asset_b_id, tier_index
2. **Asset info**: Fetch asset params for asset_a_id and asset_b_id from algod

No pool scanning required.

---

## Pool State

Each pool tracks whether its LP tokens have been registered:

```
Key:   "registered"
Type:  uint64
Value: 0 = not registered, 1 = registered
```

This flag is set during `register_pool_lps` and checked by `seed_and_mint` to ensure the registry is populated before liquidity enters.

---

## Why Separate Registration?

The reverse LP boxes cannot be written during `create_pool` because LP asset IDs are created by inner transactions during bootstrap — their IDs are not known at transaction construction time and cannot be included as box references in the same call.

This is solved by splitting registration from deployment:

```
Step 1: create_pool
  → Deploys pool, bootstraps (creates 6 LP tokens)
  → Forwards pair-box MBR to registry, calls register_pair
  → Pool's "registered" flag = 0

Step 2: register_pool_lps
  → Reads 6 LP asset IDs from pool state
  → Forwards LP-box MBR to registry, calls register_lps
  → Sets pool's "registered" flag = 1
  → Transfers governor authority to admin contract

Step 3: seed_and_mint
  → Requires registered == 1
  → Seeds default tiers + adds initial liquidity
```

`register_pool_lps` is permissionless — anyone can call it. The `registered` gate on `seed_and_mint` enforces ordering.

---

## Factory Methods

### `create_pool(pay, uint64, uint64) → uint64`

Deploys a new pool, forwards pair-box MBR to the registry, and calls `register_pair`.

- **Returns**: Pool app ID
- **Registry box written**: `"p" + itob(min_asset_id) + itob(max_asset_id)` → `itob(pool_app_id)`

### `register_pool_lps(pay, uint64) → void`

Forwards LP-box MBR to the registry, calls `register_lps`, and transfers governor to the admin contract. Permissionless.

- **Seed payment**: Minimum 150,000 µALGO covering box MBR
- **Transaction requirements**: Pool app in foreign apps, registry app in foreign apps
- **Checks**: Factory created the pool, pool not already registered, registry configured
- **Side effects**: Sets `registered = 1` on the pool, transfers governor to admin contract

---

## Registry Methods

### `get_pool(uint64, uint64) → uint64`

Reads the pair registry box and returns the pool app ID.

### `get_lp_info(uint64) → byte[]`

Reads a reverse LP box and returns 32 bytes of context.

### `register_pair(uint64, uint64, uint64) → void`

Writes a pair registry box. Writer-only (called by the factory via inner transaction).

### `register_lps(application) → void`

Writes 6 reverse LP boxes for a pool. Writer-only. Verifies the pool's creator matches the writer.

---

## MBR Cost

| Registry | Boxes per Pool | Cost per Box (µALGO) | Total (µALGO) |
|---|---|---|---|
| Pair | 1 | 2,500 + 400 × 25 = 12,500 | 12,500 |
| Reverse LP | 6 | 2,500 + 400 × 41 = 18,900 | 113,400 |
| **Total** | **7** | | **125,900** |

Pair box MBR is covered by `MIN_SEED_CREATE_POOL` (forwarded from factory to registry). Reverse LP box MBR is covered by `MIN_SEED_REGISTER_LPS` (150,000 µALGO, forwarded from factory to registry).

---

## SDK Usage

### Look up pool by pair

```python
from stamm.registry import get_pool_app_id

pool_id = get_pool_app_id(algod, registry_app_id, asset_a_id, asset_b_id)
# Returns int or None
```

### Look up LP token context

```python
from stamm.registry import get_lp_info

info = get_lp_info(algod, registry_app_id, lp_asset_id)
# Returns {"pool_app_id": int, "asset_a_id": int, "asset_b_id": int, "tier_index": int}
# Returns None if not registered
```

### Register LP boxes after pool creation

```python
from stamm.factory import prepare_register_pool_lps_transactions

# lp_asset_ids: list of 6 LP asset IDs read from pool global state
txn_group = prepare_register_pool_lps_transactions(
    factory_app_id, pool_app_id, lp_asset_ids, sender, sp
)
txn_group.sign_with_private_key(sender, private_key)
txn_group.submit(algod)
```

### Raw box reads

```python
import base64

# Pair lookup (read from registry contract)
pair_key = b"p" + min(a, b).to_bytes(8, "big") + max(a, b).to_bytes(8, "big")
box = algod.application_box_by_name(registry_app_id, pair_key)
pool_app_id = int.from_bytes(base64.b64decode(box["value"]), "big")

# LP lookup (read from registry contract)
lp_key = b"l" + lp_asset_id.to_bytes(8, "big")
box = algod.application_box_by_name(registry_app_id, lp_key)
value = base64.b64decode(box["value"])
pool_app_id = int.from_bytes(value[0:8], "big")
asset_a_id  = int.from_bytes(value[8:16], "big")
asset_b_id  = int.from_bytes(value[16:24], "big")
tier_index  = int.from_bytes(value[24:32], "big")
```
