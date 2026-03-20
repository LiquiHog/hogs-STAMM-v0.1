# Trading Operations Testing

Integration tests for swap operations, price-limited swaps, and multi-tier routing scenarios.

---

## Standard Swap Tests

### Basic Swap Validation

**Test**: Execute a simple A->B swap on a single tier

**Scenario**:
1. User deposits 1,000,000 microunits of Asset A
2. Contract applies two-sided fee (half on input, half on output)
3. Constant-product formula computes output amount
4. User receives Asset B (minus output fee)
5. Tier reserves updated, k-invariant verified

**Assertions**:
- Output amount > 0
- Output amount >= `min_output` (slippage protection)
- Post-swap k > pre-swap k (fee floors guarantee strict growth)
- User balance changes match expected deltas
- Tier reserves updated correctly

---

### Reverse Direction Swap

**Test**: Execute B->A swap (opposite direction)

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
- Post-swap marginal price ~ price limit (within fee tolerance)

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
2. Execute A->B swap (ALGO in, ASA out)
3. Execute B->A swap (ASA in, ALGO out)

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
- Post-swap reserves must be >= 1 microunit
- Underflow check prevents reserve depletion
- Error message indicates which reserve would underflow

---

## Multi-User Concurrency Tests

### Interleaved Swaps

**Test**: Multiple users swap on same tier simultaneously

**Scenario**:
1. User A swaps A->B
2. User B swaps B->A (same tier)
3. User C swaps A->B again
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

## Smart-Routed Swap Tests

### Single-Tier Fallback

**Test**: `swap_smart` routes entirely to best tier when input fits within capacity

**Scenario**:
1. Two active tiers with different liquidity depths
2. Small swap where best tier can absorb entire input
3. Verify 100% routes to highest-scored tier

**Assertions**:
- Output matches single-tier swap on best tier
- No sub-swaps to secondary tiers
- RT box scores correctly identify best tier

---

### Two-Tier Waterfall

**Test**: `swap_smart` splits across two tiers when input exceeds best tier's capacity

**Scenario**:
1. Three active tiers with varying liquidity
2. Swap large enough to exceed best tier's equalization capacity
3. Capacity routes to best tier, remainder to second-best

**Assertions**:
- First tier receives exactly the waterfall capacity amount
- Remainder routes to second-best tier
- Combined output > what either tier alone would produce
- Both sub-swaps independently enforce k-growth and fees

---

### Three-Tier Split

**Test**: `swap_smart` uses three tiers for very large input

**Scenario**:
1. Multiple active tiers with moderate liquidity
2. Input exceeds two-tier equalization capacity
3. Three-tier proportional split used for the overflow

**Assertions**:
- Three tiers receive sub-swaps
- Fee-adjusted proportional weighting applied
- Total output optimized across all three tiers
- Each sub-swap applies full two-sided fee and inline spill

---

## Caller-Directed Routed Swap Tests

### Single-Leg Routing

**Test**: `swap_routed` with one explicit (tier, amount) leg

**Scenario**:
1. Caller provides 1 leg: (tier=2, amount=1,000,000)
2. Equivalent to standard swap on Tier 2

**Assertions**:
- Output matches standard single-tier swap
- Leg encoding correctly parsed (9 bytes: 1 tier + 8 amount)

---

### Multi-Leg Routing

**Test**: `swap_routed` with 3 explicit legs across different tiers

**Scenario**:
1. Caller provides 3 legs computed off-chain
2. Each leg targets a different tier with a specific amount
3. All legs execute sequentially in a single transaction

**Assertions**:
- Sum of leg amounts equals total input
- Each tier receives exactly its specified amount
- Each sub-swap applies full fees independently
- Combined output reflects all leg executions

---

### Leg Sum Validation

**Test**: `swap_routed` rejects if leg amounts don't match input

**Scenario**:
1. Caller sends 1,000,000 input but legs sum to 900,000
2. Transaction reverts

**Assertions**:
- Input validation enforced
- No partial execution or leftover funds

---

## Test Results

[OK] **All trading operation tests passed**, confirming:
- Standard swaps execute correctly on all tiers
- Price-limited swaps refund unused input atomically
- Smart-routed swaps optimally split across up to 3 tiers
- Caller-directed routed swaps execute explicit multi-leg strategies
- Slippage protection enforced for all swap types
- Multi-tier routing works independently
- ALGO and ASA pools handle transactions correctly
- Protocol fee extraction and redistribution accurate
- Multi-user concurrency safe and correct
- Error conditions properly rejected with clear messages
