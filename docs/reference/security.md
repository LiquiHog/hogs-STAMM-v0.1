# Security

STAMM implements multiple layers of protection against common AMM attack vectors and operational risks.

---

## Invariant Enforcement

Every swap enforces the constant-product invariant: `k_new > k_old`, where `k = reserve_a * reserve_b`. The strict inequality (not just `>=`) is guaranteed by minimum fee floors — both the input fee and output fee are floored to at least 1 microunit, ensuring k always grows regardless of trade size or fee rate. Swaps too small to produce any output after the fee (typically under 3-4 microunits) fail with a clear error rather than a silent invariant violation.

Wide math (128-bit multiplication via `mulw`) is used to compute k without overflow. All wide-math divisions include overflow assertions on the quotient high word, ensuring results always fit in uint64. If the k product exceeds uint64 max, it is capped at max — this is safe because the comparison only needs to determine relative ordering, not exact magnitude.

## Reserve Minimums

All operations enforce that reserves remain at least 1 microunit after execution. This prevents:

- **Zero-reserve attacks**: Draining a tier to zero would break the constant-product formula
- **Division by zero**: Many calculations divide by reserves
- **Dust extraction**: Repeatedly withdrawing to manipulate rounding

## Seed State Protection

Every tier starts with a locked seed state: reserve_a = 1, reserve_b = 1, total_lp = 1. The 1 LP token is permanently locked in the pool. This means:

- Tiers can never be fully drained — the seed LP cannot be burned
- Auto-deactivation resets the tier to seed state and sweeps excess reserves to treasury
- The seed state establishes a minimum baseline that prevents division-by-zero edge cases

## Rounding Direction

All rounding consistently favors the pool over the user:

- **Swap output**: Floor division (user gets less)
- **Burn withdrawal**: Floor division (user gets less)
- **Mint LP calculation**: Floor division (user gets fewer LP tokens)
- **Proportional mint ceiling**: Non-binding side uses ceiling division (user deposits slightly more)
- **Fee floors**: Both input and output fees floored to 1 microunit, guaranteeing k growth
- **Protocol fee split**: Tier-retained portion floored to 1, protocol fee computed via underflow-safe subtraction
- **Price-limited swap**: Floor division on the pre-fee amount conversion, ensuring the actual swap stays at or above the user's price limit

This ensures rounding cannot be exploited to extract value from the pool.

## Overflow Protection

The AVM operates on uint64 arithmetic (0 to 2^64 - 1). STAMM uses several strategies to handle large values safely:

- **`mulw` / `divmodw`**: 128-bit wide math for products that may exceed uint64
- **`safe_mul_div`**: Computes `a * b / c` using wide intermediates
- **`sqrt128`**: 128-bit square root via Newton's method for single-sided mint and price-limited swap calculations
- **TWAP accumulators**: 128-bit (split into hi/lo words) to prevent wraparound over long periods

## Slippage Protection

Every user-facing operation includes minimum output parameters. If market conditions change between transaction submission and execution, the transaction fails rather than executing at a worse price. Price-limited swaps provide an additional layer: the user specifies a marginal price threshold and any input that would push the price beyond that limit is refunded rather than executed.

## ALGO Pool Safety

ALGO pools (asset A = ALGO, id 0) include additional protections:

- **MBR guard**: Before sending ALGO, the contract asserts `balance - min_balance >= amount` to ensure the pool never goes below its minimum balance requirement
- **No ALGO opt-in**: Bootstrap skips the opt-in step for ALGO (ALGO doesn't require opt-in)
- **Transaction type validation**: A-side deposits are validated as Payment transactions for ALGO pools, AssetTransfer for ASA pools. Mixed types are rejected.
- **Close-remainder protection**: Payment deposits assert `close_remainder_to == zero_address` to prevent draining

## Governor Model

Pools are governed by the factory contract, not by individual accounts. This means:

- Pool configuration changes require going through the factory
- The factory's admin controls all pools uniformly
- Compromising a single pool's admin key doesn't happen (pools don't have individual admin keys)

## Pool Ownership Verification

All factory proxy methods verify that the target pool was created by the factory (via `app_creator` check). This prevents interaction with rogue contracts that impersonate STAMM pools.

## Seed Payment Validation

Pool creation, LP registration, and tier seeding methods validate that the seed payment sender matches the transaction sender. This prevents third parties from hijacking creation flows by front-running seed payments.

## Registration Gate

The `seed_and_mint` method requires `registered == 1` before proceeding. This ensures the factory's reverse LP lookup boxes are written before any liquidity enters the pool, maintaining registry completeness.

## Auto Tier Management

Tiers are managed automatically rather than by governor action:

- **Auto-activate**: A tier becomes active when its total LP exceeds 1 (first real deposit)
- **Auto-deactivate**: A tier becomes inactive when total LP returns to 1 (all user LP burned)
- **Sweep-on-final-burn**: Excess reserves are swept to treasury and the tier resets to seed state

This eliminates governance attack vectors around manual tier activation/deactivation and ensures no value is stranded in inactive tiers.

## Tier Index Validation

All operations that accept a tier index validate that it falls within the valid range (0-7). This prevents out-of-bounds access to state keys.

## Permissionless Operations

Several operations are intentionally permissionless to prevent the protocol from holding user funds hostage:

- Pool creation — anyone can create a pool for a new asset pair
- LP box registration — anyone can register a pool's LP boxes after creation

## Treasury Pull Model

Treasury claims (LP tokens, dust assets) are stored in pool state rather than transferred during swaps. The admin withdraws on demand via factory proxy methods. This eliminates opt-in coordination, reduces inner transactions during trading, and ensures withdrawals never block normal pool operations.

## Known Limitations

- **No reentrancy guard**: Algorand's execution model (no recursive app calls within a group) provides inherent reentrancy protection
- **Timestamp dependence**: The TWAP oracle relies on block timestamps, which have limited granularity on Algorand (~3.3 second blocks)
- **Single-block manipulation**: Within a single block, a large trade can temporarily skew the price. The TWAP's time-weighting mitigates this over longer windows
- **Price limit approximation**: The `swap_limit` method uses a fee-free approximation for the maximum input calculation. Tier-retained fees cause the actual post-swap price to differ from the exact limit by a negligible amount (1-2 microunits), always in the conservative direction (price stays at or above the limit)
