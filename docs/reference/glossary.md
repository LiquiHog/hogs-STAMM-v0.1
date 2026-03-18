# Glossary

| Term | Definition |
|---|---|
| **STAMM** | Stratified Automated Market Maker — STAMM's multi-tier AMM architecture |
| **Tier** | An independent constant-product market within a pool, with its own reserves, fee rate, and LP token |
| **Tier P** | The passive/backstop tier (index 5) with near-zero fees and no protocol fee extraction |
| **Constant Product** | The invariant `k = reserve_a * reserve_b` that governs pricing. Output is calculated so k never decreases |
| **k-value** | The product of a tier's reserves (`reserve_a * reserve_b`). Higher k = deeper liquidity = less slippage |
| **LP Token** | An ASA issued per tier representing proportional ownership of that tier's reserves |
| **Seed State** | A tier's initial state: reserve_a = 1, reserve_b = 1, total_lp = 1. The 1 LP is locked permanently in the pool |
| **Bootstrap** | A tier's state after seeding but before its first real deposit. Auto-activates on first mint |
| **Auto-Activate** | A tier automatically becomes active when total LP > 1 (first real deposit after seeding) |
| **Auto-Deactivate** | A tier automatically becomes inactive when total LP returns to 1 (all user LP burned). Excess reserves swept to treasury |
| **Sweep-on-Final-Burn** | When a tier auto-deactivates, excess reserves beyond seed state (1,1) are sent to treasury claims |
| **BPS (Basis Points)** | Fee unit. 1 bps = 0.01%. A 30 bps fee = 0.3% |
| **PPM (Parts Per Million)** | Fee unit for Tier P. 1 ppm = 0.0001% |
| **Two-Sided Fee** | Fee model where each swap is charged on both input and output sides (half the fee rate on each, each side floored to 1 microunit) |
| **Tier-Retained Fee** | 80% of the tier fee that stays in the tier's reserves, directly benefiting LPs. Floored to 1 on every swap |
| **Protocol Fee** | 20% of the tier fee, extracted for inline spill redistribution |
| **Inline Spill** | Per-swap redistribution of protocol fees to Tier P (10%), the weakest tier (55%), and the 2nd weakest tier (35%). Targets determined by inline O(5) scan of standard tiers (0-4), excluding the current tier |
| **Pull Model Treasury** | Treasury claims stored in pool state (`tr_a`, `tr_b`, `t{c}_tl`), withdrawn on demand by the admin via factory proxy methods |
| **Swap Limit** | Price-limited partial swap that executes up to a user-specified marginal price threshold, refunding unused input. Uses `sqrt128` to compute the maximum input from the constant-product formula |
| **TWAP** | Time-Weighted Average Price — a manipulation-resistant price computed from cumulative accumulators |
| **Aggregate Reserves** | The sum of reserves across all active tiers, used for the TWAP oracle's weighted price |
| **Tier Mask** | A uint64 bitmask tracking which tiers are active (bit i = tier i). 6 bits used (0-5) |
| **Registered** | A pool-level flag (0 or 1) indicating whether the factory has written the reverse LP lookup boxes. `seed_and_mint` requires `registered == 1` |
| **Reverse LP Registry** | Factory box storage mapping each LP asset ID to its pool, pair, and tier. 6 entries per pool, written during `register_pool_lps` |
| **Value Encoding** | ABI encoding where asset params are passed as raw uint64 IDs and account params as address bytes, rather than as indices into foreign arrays |
| **ALGO Pool** | A pool where asset A is ALGO (id 0). A-side deposits use Payment transactions; outgoing ALGO uses Payment with MBR guard |
| **OpUp** | An inner app call to a helper contract that increases the available opcode budget |
| **MBR** | Minimum Balance Requirement — ALGO that must be held by an account to maintain its on-chain state |
| **Governor** | The address authorized to perform admin operations on a pool (always the factory contract) |
| **Factory** | The contract that deploys and manages all STAMM pools |
| **Wide Math** | 128-bit arithmetic operations (`mulw`, `divmodw`, `addw`) used to prevent uint64 overflow |
| **ARC-4** | Algorand standard for ABI-compatible smart contract method calls |
| **ASA** | Algorand Standard Asset — the token standard used for LP tokens and pool assets |
| **Smart-Routed Swap** | Auto-routed swap (`swap_smart`) that routes across up to 3 tiers using waterfall routing for optimal execution. Best tier fills first; overflow cascades to the second-best and third-best tiers via fee-adjusted proportional splitting |
| **Creation Mode** | Factory setting controlling pool creation access: 0 = paused, 1 = admin-only, 2 = permissionless. Configurable by factory admin |
| **Volume Counters** | 128-bit cumulative volume counters (`vol_a_hi/lo`, `vol_b_hi/lo`) tracking total swap volume for each asset across a pool's lifetime |
| **256-bit TWAP** | STAMM's TWAP accumulators use 256-bit precision (4 × uint64 words each) with full carry propagation, preventing overflow over long pool lifetimes |
