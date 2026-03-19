# Swaps & Liquidity

STAMM supports multiple ways to trade and provide liquidity, all operating on a per-tier basis within a single pool contract.

---

## Swaps

### Standard Swap

A standard swap executes entirely on one tier. The trader sends one asset and receives the other, minus the tier's fee.

**Flow:**
1. Trader sends Asset A (or B) to the pool contract. For ALGO pools, sending ALGO uses a Payment transaction; sending ASAs uses an AssetTransfer.
2. Contract applies the [two-sided fee](fee-engine.md#1-two-sided-fee): half deducted from input, half from output
3. Constant-product formula determines the raw output on the chosen tier
4. Protocol fees from both sides are redistributed inline via spill
5. Output asset (minus output fee) is sent to the trader. ALGO outputs use Payment; ASA outputs use AssetTransfer.
6. Tier reserves and the k-invariant are updated

**Protections:**
- `min_output` parameter for slippage protection
- k-invariant must grow after every swap (minimum 1 microunit fee on each side guarantees this)
- Unseeded tiers cannot be swapped through

### Price-Limited Swap

A price-limited swap (`swap_limit`) executes as much of the input as possible without pushing the pool's marginal price below a user-specified threshold. Any unused input is refunded atomically.

**Flow:**
1. Trader sends input tokens and specifies a price limit (`price_num / price_den`, output per input)
2. Contract computes the maximum effective input that maintains the price limit using constant-product mechanics and wide arithmetic
3. The effective maximum is converted to a pre-fee amount (conservative floor division)
4. If the full input fits within the limit, a normal swap executes with no refund
5. Otherwise, a partial swap executes and the remainder is refunded to the sender
6. The same two-sided fee, inline spill, k-growth assertion, and `min_output` slippage protection apply

**Use cases:**
- Large traders deploying size at a maximum price without worrying about slippage
- Bots and arbitrageurs targeting precise rate thresholds
- DeFi composability — other contracts can call `swap_limit` for rate-bounded execution without off-chain precomputation

**Key properties:**
- The contract adjusts the stated price limit for fee impact, so the result approximates an effective rate guarantee. The post-swap marginal price will be at or slightly above the adjusted limit
- Uses 128-bit square root (`sqrt128`) and wide math internally; requires an OpUp budget call
- If the current pool price is already at or below the limit, the transaction reverts (`"price already at limit"`)

### Smart-Routed Swap

A smart-routed swap (`swap_smart`) automatically routes a trade across up to 3 tiers using waterfall routing. The caller does not need to choose a tier — the contract selects the optimal path using a pre-computed routing table (RT) box.

**Flow:**
1. Trader sends input tokens. No tier index is specified.
2. Contract reads the routing table (RT) box, which stores fee-adjusted scores for each tier
3. The best, second-best, and third-best tiers are identified by score
4. Contract computes the waterfall capacity — the amount that equalizes the best tier's post-swap rate with the second-best's current rate
5. If the input fits within capacity (or only one tier qualifies), 100% routes to the best tier
6. Otherwise, capacity routes to the best tier and the remainder is split between the best and second-best tiers using fee-adjusted proportional weighting
7. If the remainder exceeds the second-best’s equalization capacity with the third-best, a 3-tier proportional split is used
8. All sub-swaps apply the full two-sided fee, inline spill, and k-growth assertion independently
9. Total output from all tiers is sent to the trader

**Key properties:**
- Automatically finds the best execution across the pool's liquidity
- Supports up to 3-tier waterfall split for large trades
- `min_output` slippage protection applies to the combined output
- Routing table is updated after every state-changing operation (swaps, mints, burns)
- Uses wide math for capacity calculation (`sqrt128`, `safe_mul_div`)

### Caller-Directed Routed Swap

A caller-directed routed swap (`swap_routed`) lets the caller specify explicit (tier, amount) legs computed off-chain. This bypasses the on-chain routing table and enables custom routing strategies beyond the automatic 3-tier waterfall.

**Flow:**
1. Trader sends input tokens and provides packed routing legs: N × 9 bytes (1 byte tier index + 8 bytes amount per leg)
2. Supports 1 to 6 legs. Each leg specifies a tier and the amount to route through it
3. The sum of all leg amounts must exactly equal the deposited input
4. Each leg executes independently: constant-product swap with two-sided fee, k-growth assertion, and reserve update
5. After all legs execute, aggregate slippage is checked once against `min_output`
6. A single inline spill redistributes combined protocol fees from all legs
7. Routing table entries are updated once for all affected tiers
8. Total output from all legs is sent to the trader

**Key properties:**
- Requires an OpUp budget call (budget 5600 for 6-leg worst case)
- Gives callers full control over routing — useful for MEV-aware strategies, custom aggregators, and off-chain optimizers
- Same fee, spill, and k-growth guarantees as standard swaps
- `min_output` slippage protection applies to the combined output

---

## Liquidity Operations

All mint and burn operations use a single unified entry point each. The contract determines the operation mode from the provided parameters.

### Mint

A single `mint` method handles all deposit modes:

- **Bootstrap mint**: First deposit into a seeded tier (k ≤ bootstrap threshold). Both assets required, any ratio accepted. The depositor sets the initial price. LP tokens minted using geometric mean (`sqrt(deposit_a × deposit_b)`).
- **Balanced mint**: Both deposits match the tier's current ratio. No swap fee. The binding side determines LP minted; excess on the non-binding side is refunded.
- **Hybrid mint**: Both deposits provided but off-ratio. The contract computes the optimal swap amount, internally swaps the excess through the tier's reserves to balance the deposit, then mints LP from the result. The swap portion incurs the tier's two-sided fee. More capital efficient than balanced mint (all deposited assets are used).
- **Single-sided mint**: One deposit is zero. The contract computes the optimal swap split using 128-bit square root, swaps part of the deposit for the other asset, then mints LP from the balanced result. The swap portion incurs the tier's two-sided fee. Not available on bootstrap tiers. Tiny residuals from rounding are refunded.

The caller sends two deposits (A and B) in the same group. Either deposit may be zero for single-sided minting. For ALGO pools: Payment for A-side, AssetTransfer for B-side. Tier P is not open to public mints.

### Burn

A single `burn` method handles all withdrawal modes, selected by the `output_asset` parameter:

- **Proportional burn** (`output_asset` = pool LP asset): Returns both Asset A and Asset B proportional to the LP share. Always available, even on inactive tiers. Floor division favors the pool.
- **Single-sided burn to A** (`output_asset` = asset A): Burns proportionally, then internally swaps the B portion for A. Incurs the tier's two-sided fee on the swap portion.
- **Single-sided burn to B** (`output_asset` = asset B): Burns proportionally, then internally swaps the A portion for B. Incurs the tier's two-sided fee on the swap portion.

Final burn (all user LP withdrawn, tier returns to seed state): excess reserves beyond (1,1) are swept to treasury claims, and the tier auto-deactivates.

---

## ALGO Pool Deposits

For ALGO/ASA pools where asset A is ALGO (id 0):

- **A-side deposits** (ALGO) use Payment transactions
- **B-side deposits** (ASA) use AssetTransfer transactions as normal
- **Swap deposits** use Payment for ALGO input, AssetTransfer for ASA input
- **Single-sided deposits** use whichever transaction type matches the deposited asset
- **LP burns** always use AssetTransfer (LP tokens are always ASAs)
- **Outgoing ALGO** uses Payment with an MBR guard (`balance - min_balance >= amount`)

The SDK handles this automatically — callers specify asset IDs and amounts, and the correct transaction type is constructed.

---

## Operation Summary

| Operation | Assets In | Assets Out | Fee Charged | Bootstrap OK |
|---|---|---|---|---|
| Swap | 1 | 1 | Yes (two-sided) | No |
| Swap (limit) | 1 | 1 + refund | Yes (two-sided) | No |
| Swap (smart) | 1 | 1 | Yes (two-sided, per tier) | No |
| Swap (routed) | 1 | 1 | Yes (two-sided, per leg) | No |
| Mint (balanced) | 2 | LP + refund | No | Yes |
| Mint (hybrid) | 2 | LP + residual | On excess swap | No |
| Mint (single-sided) | 1 (+ 0) | LP + residual | On swap portion | No |
| Burn (proportional) | LP | 2 | No | Yes |
| Burn (single-sided) | LP | 1 | On swap portion | No |

---

## Slippage Protection

All user-facing operations include slippage parameters:

- **Swaps**: `min_output` — minimum output tokens
- **Price-limited swaps**: `min_output` — minimum output tokens (in addition to the price limit)
- **Smart-routed swaps**: `min_output` — minimum combined output from all tiers
- **Caller-directed routed swaps**: `min_output` — minimum combined output from all legs
- **Mints**: `min_lp_out` — minimum LP tokens to receive
- **Burns**: `min_a_out` / `min_b_out` — minimum per-asset output (used for both proportional and single-sided modes)

If the actual result is below the minimum, the transaction fails atomically. No partial execution.
