# Glossary

| Term | Definition |
|---|---|
| **256-bit TWAP** | STAMM TWAP accumulators use 4 x uint64 words per side with carry propagation to avoid overflow over long periods |
| **Admin Contract** | Contract that becomes pool governor after registration and coordinates governance, migration, and treasury proxy calls |
| **Admin Timelock** | 7-day (604,800 second) delay on admin-contract admin transfer, treasury transfer, and governor migration proposals |
| **Aggregate Reserves** | Sum of reserves across active tiers (`agg_ra`, `agg_rb`) used by TWAP calculations |
| **ALGO Pool** | Pool where `asset_a` is ALGO (`id=0`), using `Payment` for A-side deposits |
| **ARC-4** | Algorand ABI standard for smart contract method calls |
| **ASA** | Algorand Standard Asset |
| **Auto-Activate** | Tier becomes active when `total_lp > 1` |
| **Auto-Deactivate** | Tier returns to inactive state when `total_lp == 1` and excess value is swept to treasury claims |
| **Bootstrap** | Initial seeded tier state used before normal active trading behavior |
| **Box MBR Rates** | Cached protocol costs (`box_flat_mbr`, `box_byte_mbr`) used to compute required box funding |
| **Creation Mode** | Factory pool-creation policy (`0 paused`, `1 admin-only`, `2 permissionless`) |
| **Factory Freeze** | `freeze_factory` mode that blocks template-management methods until timelocked unfreeze |
| **Factory Timelock** | 7-day delay on factory governance transitions |
| **Guardian** | Admin-contract role that can cancel pending admin/treasury/migration proposals |
| **Governor** | Address authorized for pool governor-only methods |
| **Governor Migration** | Admin-contract flow to move pools to a new governor via `propose_migration` and `confirm_pool_migration` |
| **Inline Spill** | Per-swap protocol-fee redistribution to Tier P and weak tiers |
| **k-value** | Reserve product (`reserve_a * reserve_b`) indicating depth/slippage characteristics |
| **LP Token** | ASA representing share ownership of a specific tier |
| **MBR** | Minimum balance requirement in ALGO for accounts/apps/boxes |
| **PoolFactory** | Deployment and registration contract for STAMM pools |
| **Pull Model Treasury** | Treasury claims stay in pool state and are withdrawn on demand through governor methods |
| **Registered Flag** | Pool key (`registered`) set after LP reverse-lookup boxes are written |
| **Registry Contract** | On-chain pair/LP lookup registry with writer-gated writes and timelocked admin transfer |
| **Reverse LP Registry** | Registry mapping from LP asset ID to pool app ID, pair assets, and tier index |
| **Seed State** | Baseline tier state with minimum non-zero reserves/LP supply used for safety and lifecycle transitions |
| **STAMM** | Stratified Automated Market Maker |
| **Sweep-on-Final-Burn** | Final-burn behavior that moves residual tier value to treasury claims and resets tier state |
| **Tier** | One fee band inside a pool, each with independent reserves and LP token |
| **Tier Mask** | Bitmask (`tier_mask`) tracking active tiers |
| **Tier P** | Protocol-managed low-fee tier (index 5) |
| **Treasury Operator** | Admin-contract role permitted to execute treasury withdrawal methods |
| **TWAP** | Time-weighted average price derived from cumulative oracle accumulators |
| **Two-Sided Fee** | Fee model splitting deductions across input and output sides of swaps |
| **Value Encoding** | ABI style where assets are passed as raw IDs and accounts as addresses |
| **Volume Counters** | 128-bit cumulative swap volume counters for each asset |
| **Writer** | Registry-authorized address allowed to register pair and LP lookup boxes |
