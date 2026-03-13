# TWAP Oracle

STAMM includes an inline time-weighted average price (TWAP) oracle that updates on every reserve-modifying transaction. The oracle provides manipulation-resistant price data for external consumers.

---

## Design

### Aggregate Price

Unlike single-pool AMMs where the oracle reflects one set of reserves, STAMM's oracle uses **aggregate reserves across all active tiers**. This means the reported price is a liquidity-weighted average, making it harder to manipulate by targeting a single low-liquidity tier.

The aggregate reserves (`agg_ra`, `agg_rb`) are maintained incrementally — updated on every tier state change — so the oracle never needs to loop over tiers.

### Inline Updates

The TWAP is updated at the **start** of every swap, mint, and burn operation. There is no separate oracle update call. This guarantees the accumulators advance whenever the pool is active and prevents stale data between external oracle calls.

### Accumulator Mechanics

The oracle maintains two 128-bit accumulators (each split into high/low uint64 words):

- **Accumulator A**: Cumulative price of Asset A in terms of Asset B
- **Accumulator B**: Cumulative price of Asset B in terms of Asset A

On each update:
1. Compute `elapsed = current_timestamp - last_timestamp`
2. If elapsed > 0 and reserves are non-zero:
   - Calculate current price of A in terms of B using aggregate reserves with fixed-point scaling
   - Calculate current price of B in terms of A using aggregate reserves with fixed-point scaling
   - Add `price × elapsed` to respective accumulators (128-bit addition with carry handling)
3. Update `last_timestamp`

The fixed-point scaling provides sufficient precision for price ratios while keeping arithmetic within AVM capabilities.

---

## Reading the Oracle

### Off-Chain TWAP Calculation

To compute TWAP over a time window:

1. Read accumulators at time `t1` -> `(ca1_hi, ca1_lo, cb1_hi, cb1_lo, ts1)`
2. Read accumulators at time `t2` -> `(ca2_hi, ca2_lo, cb2_hi, cb2_lo, ts2)`
3. Compute time-weighted average by dividing the accumulator delta by the time delta
4. Apply inverse fixed-point scaling to get human-readable prices

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
