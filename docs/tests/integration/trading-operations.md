# Trading Operations Testing

Integration tests for swap operations, price-limited swaps, and multi-tier routing scenarios.

---

## Standard Swap Tests

### Basic Swap Validation

**Test**: Execute a simple A→B swap on a single tier

**Scenario**:
1. User deposits 1,000,000 microunits of Asset A
2. Contract applies two-sided fee (half on input, half on output)
3. Constant-product formula computes output amount
4. User receives Asset B (minus output fee)
5. Tier reserves updated, k-invariant verified

**Assertions**:
- Output amount > 0
- Output amount >= `min_output` (slippage protection)
- Post-swap k ≥ pre-swap k (fee capture grows reserves)
- User balance changes match expected deltas
- Tier reserves updated correctly

---

### Reverse Direction Swap

**Test**: Execute B→A swap (opposite direction)

**Scenario**:
1. User deposits Asset B
2. Receives Asset A as output
3. Same fee and k-invariant checks apply

**Assertions**:
- Both directions produce symmetric behavior
- Fee calculations identical for both directions
- K-invariant grows in both cases

---

### Multi-Tier Swaps

**Test**: Execute swaps across different fee tiers

**Scenario**:
1. Seed and activate Tier 0 (3 bps), Tier 2 (30 bps), Tier 4 (300 bps)
2. Execute same-sized swap on each tier
3. Compare output amounts and fee deductions

**Assertions**:
- Lower-fee tiers produce higher output for same input
- Fee deductions scale correctly with tier fee rate
- Each tier's k-invariant grows independently
- No cross-tier interference

---

### Large Trade Handling

**Test**: Execute swap with significant liquidity impact

**Scenario**:
1. Swap amount = 50% of tier reserves
2. Verify slippage is substantial but swap executes
3. K-invariant still grows despite large price impact

**Assertions**:
- Large trades execute successfully
- Output matches constant-product formula for large dx
- No overflow in wide math operations (mulw, divmodw)
- Post-swap reserves remain valid (> 0)

---

## Price-Limited Swap Tests

### Full Input Within Limit

**Test**: Price limit does not bind, full swap executes

**Scenario**:
1. User sends 1,000,000 microunits with price limit 0.95
2. Full amount executes without hitting limit
3. No refund issued

**Assertions**:
- Output matches standard swap output
- Refund amount = 0
- Post-swap marginal price > price limit

---

### Partial Swap with Refund

**Test**: Price limit binds, partial swap executes

**Scenario**:
1. User sends 10,000,000 microunits with tight price limit
2. Contract computes max input from limit: `dx_max`
3. Swaps `dx_max`, refunds remainder

**Assertions**:
- Swap amount < input amount
- Refund amount = input - swap amount
- User receives both output + refund atomically
- Post-swap marginal price ≈ price limit (within fee tolerance)

---

### Price Already at Limit

**Test**: Current pool price violates limit before swap

**Scenario**:
1. User submits swap with price limit already breached
2. Transaction reverts: `"price already at limit"`

**Assertions**:
- No state changes
- User retains all input tokens
- Error message clear and actionable

---

## Asset Pair Type Tests

### ASA/ASA Swaps

**Test**: Swap on pool with two standard ASAs

**Scenario**:
1. Create pool with Asset A = ASA (id 1000), Asset B = ASA (id 2000)
2. Execute swaps in both directions
3. Validate AssetTransfer transactions for both deposits and outputs

**Assertions**:
- Both sides use AssetTransfer (no Payment transactions)
- Asset IDs match expected values
- No special ALGO handling required

---

### ALGO/ASA Swaps

**Test**: Swap on pool with ALGO as Asset A

**Scenario**:
1. Create pool with Asset A = ALGO (id 0), Asset B = ASA
2. Execute A→B swap (ALGO in, ASA out)
3. Execute B→A swap (ASA in, ALGO out)

**Assertions**:
- ALGO deposits use Payment transactions
- ALGO outputs use Payment with MBR guard
- ASA deposits/outputs use AssetTransfer
- Contract balance - min_balance >= ALGO output amount

---

## Slippage Protection Tests

### Min Output Enforcement

**Test**: Swap reverts if output < min_output

**Scenario**:
1. User submits swap with `min_output` set too high
2. Expected output = 950,000, `min_output` = 1,000,000
3. Transaction reverts: `"slippage"`

**Assertions**:
- No state changes
- User retains input tokens
- Error message indicates slippage protection triggered

---

### Zero Output Prevention

**Test**: Swap reverts if output rounds to zero

**Scenario**:
1. User submits extremely small swap (1 microunit)
2. Output calculation rounds to zero
3. Transaction reverts: `"output is zero"`

**Assertions**:
- Prevents zero-value swaps
- Protects against rounding exploits
- User receives clear error message

---

## Edge Case Tests

### Bootstrap Tier Rejection

**Test**: Swap on unseeded or bootstrap tier reverts

**Scenario**:
1. Attempt swap on tier with total_lp = 1 (seed state)
2. Transaction reverts: `"tier in bootstrap"`

**Assertions**:
- Bootstrap tiers cannot be swapped through
- Only active tiers (total_lp > 1) accept swaps
- Clear error message

---

### Inactive Tier Rejection

**Test**: Swap on deactivated tier reverts

**Scenario**:
1. Tier was active, all user LP burned, tier deactivated
2. Attempt swap on now-inactive tier
3. Transaction reverts: `"tier inactive"`

**Assertions**:
- Tier mask correctly updated on deactivation
- Inactive tiers reject swaps
- Error message distinguishes from bootstrap state

---

### Reserve Underflow Prevention

**Test**: Output cannot exceed available reserves

**Scenario**:
1. User attempts swap where calculated output > available reserves
2. Transaction reverts: `"reserve_b underflow"` or `"reserve_a underflow"`

**Assertions**:
- Post-swap reserves must be ≥ 1 microunit
- Underflow check prevents reserve depletion
- Error message indicates which reserve would underflow

---

## Multi-User Concurrency Tests

### Interleaved Swaps

**Test**: Multiple users swap on same tier simultaneously

**Scenario**:
1. User A swaps A→B
2. User B swaps B→A (same tier)
3. User C swaps A→B again
4. Verify all swaps execute correctly in sequence

**Assertions**:
- Reserves update correctly after each swap
- K-invariant grows monotonically
- No race conditions or state conflicts
- Each user receives expected output

---

### Cross-Tier Concurrency

**Test**: Users swap on different tiers concurrently

**Scenario**:
1. User A swaps on Tier 2
2. User B swaps on Tier 4
3. User C swaps on Tier 0

**Assertions**:
- Each tier's state updates independently
- No cross-tier state corruption
- Aggregate reserves update correctly
- TWAP oracle reflects all active tiers

---

## Fee Extraction Tests

### Protocol Fee Capture

**Test**: Protocol fees extracted and redistributed via inline spill

**Scenario**:
1. Execute swap with known input amount and fee rate
2. Calculate expected tier-retained vs protocol fee split
3. Verify protocol fee redistributed to Tier P + weak tiers
4. Check treasury LP claims minted

**Assertions**:
- 80% of fees stay in tier reserves (tier-retained)
- 20% extracted as protocol fee
- Protocol fee split: 10% Tier P, 55% weakest, 35% 2nd weakest
- Treasury LP claims incremented correctly

---

### Tier P Fee Retention

**Test**: Tier P keeps 100% of fees, no protocol extraction

**Scenario**:
1. Execute swap on Tier P
2. Verify all fees stay in Tier P reserves
3. No protocol fee extracted
4. No inline spill from Tier P swaps

**Assertions**:
- Tier P split_protocol_fee returns (total_fee, 0)
- No treasury LP minted from Tier P's own fees
- Tier P k-invariant grows by full fee amount

---

## Test Results

✅ **All trading operation tests passed**, confirming:
- Standard swaps execute correctly on all tiers
- Price-limited swaps refund unused input atomically
- Slippage protection enforced for all swap types
- Multi-tier routing works independently
- ALGO and ASA pools handle transactions correctly
- Protocol fee extraction and redistribution accurate
- Multi-user concurrency safe and correct
- Error conditions properly rejected with clear messages
