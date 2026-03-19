# TWAP Oracle

STAMM includes an inline time-weighted average price (TWAP) oracle that updates on every reserve-modifying transaction. The oracle provides manipulation-resistant price data for external consumers.

---

## Design

### Aggregate Price

Unlike single-pool AMMs where the oracle reflects one set of reserves, STAMM’s oracle uses **aggregate reserves across all active tiers** (see [Architecture — Pool State](../core/architecture.md#pool-state-global)). This means the reported price is a liquidity-weighted average, making it harder to manipulate by targeting a single low-liquidity tier.

The aggregate reserves (`agg_ra`, `agg_rb`) are maintained incrementally — updated on every tier state change — so the oracle never needs to loop over tiers.

### Inline Updates

The TWAP is updated at the **start** of every swap, mint, and burn operation. There is no separate oracle update call. This guarantees the accumulators advance whenever the pool is active and prevents stale data between external oracle calls.

### Accumulator Mechanics

The oracle maintains two 256-bit accumulators (each split into four uint64 words: `_3` most significant through `_0` least significant):

- **Accumulator A** (`twap_ca_3/2/1/0`): Cumulative price of Asset A in terms of Asset B
- **Accumulator B** (`twap_cb_3/2/1/0`): Cumulative price of Asset B in terms of Asset A

On each update:
1. Compute `elapsed = current_timestamp - last_timestamp`
2. If elapsed > 0 and reserves are non-zero:
   - Calculate current price of A in terms of B: `price_a = agg_rb × 2^32 / agg_ra`
   - Calculate current price of B in terms of A: `price_b = agg_ra × 2^32 / agg_rb`
   - Add `price × elapsed` to respective accumulators (256-bit addition with full carry propagation across all 4 words)
3. Update `last_timestamp`

The scale factor is 2^32 (4,294,967,296). This fixed-point scaling provides sufficient precision for price ratios while keeping arithmetic within AVM capabilities.

---

## Cumulative Volume Tracking

In addition to the TWAP oracle, every pool maintains 128-bit cumulative volume counters for both assets:

- **`vol_a_hi`, `vol_a_lo`**: Cumulative volume of Asset A across all swaps
- **`vol_b_hi`, `vol_b_lo`**: Cumulative volume of Asset B across all swaps

Volume is accumulated on every swap (including both sides of smart-routed swaps). The counters use 128-bit arithmetic (`addw` with carry propagation) to prevent overflow over the pool's lifetime.

These counters enable off-chain analytics and can be used for volume-weighted pricing calculations.

---

## Reading the Oracle

### Off-Chain TWAP Calculation

To compute TWAP over a time window:

1. Read accumulators at time `t1` -> `(ca1_hi, ca1_lo, cb1_hi, cb1_lo, ts1)`
2. Read accumulators at time `t2` -> `(ca2_hi, ca2_lo, cb2_hi, cb2_lo, ts2)`
3. Compute delta: `delta_a = ca2 - ca1` (256-bit subtraction)
4. Divide by time: `twap_a_scaled = delta_a / (ts2 - ts1)`
5. Remove scaling: `twap_a = twap_a_scaled / 2^32` to get human-readable price

**Recommended minimum window: 30 minutes.** Shorter windows are more susceptible to manipulation via flash-loan-style attacks within a single block.

### Extending to Current Time

If the last oracle update was at `ts` and the current time is `now`, calculate the pending accumulation using the current price multiplied by elapsed time since the last update, then add to the stored accumulator value.

This lets consumers compute up-to-the-second TWAP without waiting for another on-chain transaction.

---

## Security Properties

- **Manipulation cost scales with liquidity**: Skewing the aggregate price requires moving reserves across ALL active tiers, not just one
- **Time-weighting**: Temporary price spikes have proportionally less impact over longer windows
- **No external dependencies**: The oracle is self-contained within the pool contract
- **Guaranteed freshness**: Updated on every reserve-modifying operation, not on a schedule
