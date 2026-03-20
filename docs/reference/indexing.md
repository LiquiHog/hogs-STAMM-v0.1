# Indexing & Lookups

STAMM uses `RegistryContract` box storage for pair and LP lookups.

---

## Registries

### Pair Registry

Maps an asset pair to one pool app ID.

```text
Key:   "p" + itob(min_asset_id) + itob(max_asset_id)   (17 bytes)
Value: itob(pool_app_id)                                  (8 bytes)
```

### Reverse LP Registry

Maps an LP asset ID to its pool context.

```text
Key:   "l" + itob(lp_asset_id)                           (9 bytes)
Value: itob(pool_app_id)
     + itob(asset_a_id)
     + itob(asset_b_id)
     + itob(tier_index)                                   (32 bytes total)
```

---

## Lookup Flows

### Asset Pair -> Pool

```text
Input:  asset_a_id, asset_b_id
Method: get_pool(uint64,uint64) -> uint64
Result: pool_app_id
```

### LP Token -> Pool, Pair, Tier

```text
Input:  lp_asset_id
Method: get_lp_info(uint64) -> byte[]
Result: 32 bytes = pool_app_id(8) + asset_a_id(8) + asset_b_id(8) + tier_index(8)
```

### Pool -> LP Tokens

Read pool globals: `t0_la`, `t1_la`, `t2_la`, `t3_la`, `t4_la`, `tp_la`.

---

## Registration Order

`register_pool_lps` is separate from `create_pool` because LP asset IDs are created during pool bootstrap and are not known at transaction-construction time.

```text
1) create_pool
   - deploy + bootstrap pool
   - registry pair write
2) register_pool_lps
   - registry LP writes (6 boxes)
   - set pool registered=1
   - transfer governor to admin contract
3) seed_and_mint
   - requires registered == 1
```

---

## Factory Methods

### `create_pool(pay,uint64,uint64) -> uint64`

- Deploys and bootstraps a pool.
- Computes pair-box MBR with cached box rates and forwards payment to registry write call.
- Calls `register_pair(pay,uint64,uint64,uint64)`.

### `register_pool_lps(pay,uint64) -> void`

- Computes LP-box MBR with cached box rates (`6 x box_mbr(9,32)`).
- Forwards payment and calls `register_lps(pay,uint64)`.
- Sets `registered = 1` and transfers governor to admin contract.

---

## Registry Methods

### `register_pair(pay,uint64,uint64,uint64) -> void`

Writer-only. Validates embedded MBR payment and writes pair box.

### `register_lps(pay,uint64) -> void`

Writer-only. Validates embedded MBR payment, verifies pool creator equals writer, and writes 6 LP boxes.

### `get_pool(uint64,uint64) -> uint64`

Permissionless pair lookup.

### `get_lp_info(uint64) -> byte[]`

Permissionless reverse LP lookup.

---

## MBR Cost Model

Registry and factory both use cached protocol box rates:

- `box_flat_mbr` (default `2500`)
- `box_byte_mbr` (default `400`)

Box cost formula:

```text
box_mbr = box_flat_mbr + box_byte_mbr * (key_size + value_size)
```

`set_box_mbr_rates(uint64,uint64)` on factory/registry updates cached rates if protocol values change.

---

## SDK Usage

### Pair lookup

```python
from stamm.registry import get_pool_app_id

pool_id = get_pool_app_id(algod, registry_app_id, asset_a_id, asset_b_id)
```

### LP lookup

```python
from stamm.registry import get_lp_info

info = get_lp_info(algod, registry_app_id, lp_asset_id)
# {"pool_app_id": int, "asset_a_id": int, "asset_b_id": int, "tier_index": int}
```
