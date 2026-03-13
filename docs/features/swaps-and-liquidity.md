# Swaps & Liquidity

STAMM supports multiple ways to trade and provide liquidity, all operating on a per-tier basis within a single pool contract.

---

## Swaps

### Standard Swap

A standard swap executes entirely on one tier. The trader sends one asset and receives the other, minus the tier's fee.

**Flow:**
1. Trader sends Asset A (or B) to the pool contract. For ALGO pools, sending ALGO uses a Payment transaction; sending ASAs uses an AssetTransfer.
2. Contract applies the two-sided fee: half deducted from input, half from output
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
- The price limit refers to the pool's post-swap **marginal price**, not the user's effective rate. Fees cause the effective rate to be slightly worse than the marginal price, as with any AMM swap
- Uses 128-bit square root (`sqrt128`) and wide math internally; requires an OpUp budget call
- If the current pool price is already at or below the limit, the transaction reverts (`"price at limit"`)

---

## Liquidity Operations

### Standard Mint (Two-Sided)

Deposit both assets proportional to the tier's current reserve ratio. Receive LP tokens representing your share of the tier.

- **Bootstrap mint**: First deposit into a seeded tier. The deposit uses ADD semantics (new reserves = seed reserves + deposit). Any ratio is accepted — the depositor sets the initial price. LP tokens minted using geometric mean calculation to ensure fair initial distribution.
- **Proportional mint**: Subsequent deposits must match the current ratio. The binding side determines LP minted. Excess on the non-binding side is refunded.

### Hybrid Mint

Deposit both assets in any ratio. The contract internally swaps the excess through the tier's reserves to balance the deposit, then mints LP from the result.

- More capital efficient than standard mint (no refund, all deposited assets are used)
- The swap portion incurs the tier's two-sided fee
- Useful when you have an unbalanced position and want to provide all of it

### Single-Sided Mint

Deposit only one asset. The contract computes the optimal swap split, swaps part of the deposit for the other asset, then mints LP from the balanced result.

- Uses 128-bit square root to find the optimal swap amount
- The swap portion incurs the tier's two-sided fee
- Not available on bootstrap tiers (no established ratio to target)
- Tiny residuals from fee rounding are refunded

### Standard Burn (Two-Sided)

Burn LP tokens to withdraw both assets proportionally from the tier.

- Always available, even on inactive tiers (LPs can always exit)
- Floor division favors the pool (prevents rounding exploits)
- Reserves must remain at least 1 microunit after withdrawal
- Final burn (all user LP withdrawn, tier returns to seed state): excess reserves swept to treasury, tier auto-deactivates

### Single-Sided Burn

Burn LP tokens and receive only one asset. The contract burns proportionally (receiving both assets internally), then swaps the unwanted asset for the desired one.

- The swap portion incurs the tier's two-sided fee
- Not available on bootstrap tiers

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
| Mint (standard) | 2 | LP + refund | No | Yes |
| Mint (hybrid) | 2 | LP | On excess swap | Yes (falls back to standard) |
| Mint (single) | 1 | LP + residual | On swap portion | No |
| Burn (standard) | LP | 2 | No | Yes |
| Burn (single) | LP | 1 | On swap portion | No |

---

## Slippage Protection

All user-facing operations include slippage parameters:

- **Swaps**: `min_output` — minimum output tokens
- **Price-limited swaps**: `min_output` — minimum output tokens (in addition to the price limit)
- **Mints**: `min_lp_out` — minimum LP tokens to receive
- **Standard burns**: `min_a_out` / `min_b_out` — minimum per-asset output
- **Single burns**: `min_output` — minimum total output

If the actual result is below the minimum, the transaction fails atomically. No partial execution.
