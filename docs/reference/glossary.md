# Glossary

| Term | Definition |
|---|---|
| **256-bit TWAP** | STAMM’s TWAP accumulators use 256-bit precision (4 × uint64 words each) with full carry propagation, preventing overflow over long pool lifetimes |
| **Admin Contract** | The contract that becomes permanent governor of all pools after registration, handling treasury withdrawals, governor migration, and governance operations |
| **Admin Contract Freeze** | Irreversible freeze on the admin contract (72-hour timelock to confirm). Blocks update/delete but not treasury withdrawals, admin transfer, or governor migration |
| **Admin Timelock** | 72-hour waiting period (259,200 seconds) on the admin contract for admin transfer, freeze, and governor migration |
| **Aggregate Reserves** | The sum of reserves across all active tiers, used for the TWAP oracle’s weighted price |
| **ALGO Pool** | A pool where asset A is ALGO (id 0). A-side deposits use Payment transactions; outgoing ALGO uses Payment with MBR guard |
| **ARC-4** | Algorand standard for ABI-compatible smart contract method calls |
| **ASA** | Algorand Standard Asset — the token standard used for LP tokens and pool assets |
| **Auto-Activate** | A tier automatically becomes active when total LP > 1 (first real deposit after seeding) |
| **Auto-Deactivate** | A tier automatically becomes inactive when total LP returns to 1 (all user LP burned). Excess reserves swept to treasury |
| **Bootstrap** | A tier’s state after seeding but before its first real deposit. Auto-activates on first mint |
| **BPS (Basis Points)** | Fee unit. 1 bps = 0.01%. A 30 bps fee = 0.3% |
| **Caller-Directed Routed Swap** | Explicitly-routed swap (`swap_routed`) where the caller provides 1-6 (tier, amount) legs computed off-chain, packed as N×9 bytes. Enables custom routing strategies beyond automatic waterfall |
| **Constant Product** | The invariant `k = reserve_a * reserve_b` that governs pricing. Output is calculated so k never decreases |
| **Creation Mode** | Factory setting controlling pool creation access: 0 = paused, 1 = admin-only, 2 = permissionless. Configurable by factory admin |
| **Factory** | The lightweight deployment tool that deploys STAMM pools, forwards registry data to the registry contract, and transfers governor to the admin contract |
| **Factory Freeze** | Safety mechanism that instantly blocks template uploads, contract updates, and deletions. Unfreezing requires a 7-day timelock. Pool creation and LP registration remain functional during a freeze |
| **Factory Timelock** | 7-day waiting period (604,800 seconds) required for admin transfer, admin contract changes, registry changes, and unfreeze operations |
| **Governor** | The address authorized to perform admin operations on a pool (the admin contract after registration) |
| **Governor Migration** | Process of moving pool governor authority from one contract to another via the admin contract’s `propose_migration` → `confirm_pool_migration` flow (72-hour timelock, per-pool). Works even when the admin contract is frozen |
| **Inline Spill** | Per-swap redistribution of protocol fees to Tier P (10%), the weakest tier (55%), and the 2nd weakest tier (35%). Scan covers all active tiers (excluding the current tier), with only standard tiers (0-4) eligible as recipients |
| **k-value** | The product of a tier’s reserves (`reserve_a * reserve_b`). Higher k = deeper liquidity = less slippage |
| **LP Token** | An ASA issued per tier representing proportional ownership of that tier’s reserves |
| **MBR** | Minimum Balance Requirement — ALGO that must be held by an account to maintain its on-chain state |
| **OpUp** | An inner app call to a helper contract that increases the available opcode budget |
| **PPM (Parts Per Million)** | Fee unit for Tier P. 1 ppm = 0.0001% |
| **Protocol Fee** | 20% of the tier fee, extracted for inline spill redistribution |
| **Pull Model Treasury** | Treasury claims stored in pool state (`tr_a`, `tr_b`, `t{c}_tl`), withdrawn on demand by the admin contract (as governor) |
| **Registered** | A pool-level flag (0 or 1) indicating whether the registry contract’s reverse LP lookup boxes have been written. `seed_and_mint` requires `registered == 1` |
| **Registry Contract** | Permanent on-chain contract storing pair and LP box registries. Survives factory replacement. Controlled by a writer (the factory) and an admin. Can be permanently frozen |
| **Registry Freeze** | Irreversible freeze on the registry contract that permanently blocks all write operations. Reads remain functional. Admin transfer still works when frozen |
| **Reverse LP Registry** | Registry contract box storage mapping each LP asset ID to its pool, pair, and tier. 6 entries per pool, written during `register_pool_lps` |
| **Seed State** | A tier’s initial state: reserve_a = 1, reserve_b = 1, total_lp = 1. The 1 LP is locked permanently in the pool |
| **Smart-Routed Swap** | Auto-routed swap (`swap_smart`) that routes across up to 3 tiers using waterfall routing for optimal execution. Best tier fills first; overflow cascades to the second-best and third-best tiers via fee-adjusted proportional splitting |
| **STAMM** | Stratified Automated Market Maker — STAMM’s multi-tier AMM architecture |
| **Swap Limit** | Price-limited partial swap that executes up to a user-specified marginal price threshold, refunding unused input. Uses `sqrt128` to compute the maximum input from the constant-product formula |
| **Sweep-on-Final-Burn** | When a tier auto-deactivates, excess reserves beyond seed state (1,1) are sent to treasury claims |
| **Tier** | An independent constant-product market within a pool, with its own reserves, fee rate, and LP token |
| **Tier Mask** | A uint64 bitmask tracking which tiers are active (bit i = tier i). 6 bits used (0-5) |
| **Tier P** | The passive/backstop tier (index 5) with near-zero fees and no protocol fee extraction |
| **Tier-Retained Fee** | 80% of the tier fee that stays in the tier’s reserves, directly benefiting LPs. Floored to 1 on every swap |
| **TWAP** | Time-Weighted Average Price — a manipulation-resistant price computed from cumulative accumulators |
| **Two-Sided Fee** | Fee model where each swap is charged on both input and output sides (half the fee rate on each, each side floored to 1 microunit) |
| **Value Encoding** | ABI encoding where asset params are passed as raw uint64 IDs and account params as address bytes, rather than as indices into foreign arrays |
| **Volume Counters** | 128-bit cumulative volume counters (`vol_a_hi/lo`, `vol_b_hi/lo`) tracking total swap volume for each asset across a pool’s lifetime |
| **Wide Math** | 128-bit arithmetic operations (`mulw`, `divmodw`, `addw`) used to prevent uint64 overflow |
