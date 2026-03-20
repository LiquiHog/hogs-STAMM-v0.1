# Edge Case Testing

Property-based fuzz tests exploring boundary conditions, corner cases, and potential overflow scenarios to ensure robust handling across the full parameter space.

---

## Near-Zero Reserve Tests

### Property: Protocol handles minimal liquidity levels

**Scenario**: Operations on tiers with reserves near bootstrap threshold

**Strategy**:
1. Seed tier with minimal reserves (1:1 seed state)
2. Add small liquidity (1-1000 microunits)
3. Execute swaps, mints, burns with tiny amounts
4. Verify no underflow or zero-division errors

**Parameter Ranges**:
- Reserves: 1 to 10,000 microunits
- Swap amounts: 1 to 1,000 microunits
- LP amounts: 1 to 1,000

**Assertions**:
- [OK] No underflow in constant-product formula
- [OK] Minimum 1 microunit fee enforced (prevents zero fees)
- [OK] K-invariant grows even on micro-swaps
- [OK] Bootstrap detection works correctly at threshold
- [OK] Division by zero prevented (reserve guards)

**Results**: [OK] **Passed** - All edge cases handled correctly at minimum liquidity

---

## Large Number Handling

### Property: Wide math prevents overflow in supported ranges

**Scenario**: Operations with reserves and amounts approaching uint64 limits

**Strategy**:
1. Generate reserves up to 10^15 (practical upper bound)
2. Execute swaps with amounts up to 10^12
3. Verify all wide math operations (mulw, divmodw) work correctly
4. Check sqrt128 handles large products without overflow

**Parameter Ranges**:
- Reserves: 10^12 to 10^15 microunits
- Swap amounts: 10^9 to 10^12 microunits
- Products: Up to 2^122 (within 128-bit range)

**Assertions**:
- [OK] No overflow in k-invariant checks (uses full 128-bit comparison)
- [OK] Sqrt128 converges correctly for large inputs
- [OK] Fee calculations accurate for large amounts
- [OK] TWAP accumulator additions handle large increments
- [OK] All quotient high words (_qh) checked for zero (overflow detection)

**Results**: [OK] **Passed** - Wide math handles all supported reserve ranges

---

## Precision Boundary Tests

### Property: Floor division consistently favors pool

**Scenario**: Operations where rounding could favor either side

**Strategy**:
1. Generate amounts that produce fractional results
2. Test floor division in:
   - LP mint calculation (lp = amount x total_lp / reserve)
   - LP burn calculation (out = lp x reserve / total_lp)
   - Fee split (retained = total_fee x 80 / 100)
3. Verify pool always receives rounding benefit

**Parameter Ranges**:
- Amounts producing remainder: e.g., 7 / 3, 100 / 3, 999 / 7
- Large total_lp with small amounts (forces fractional LP)
- Small total_lp with large amounts

**Assertions**:
- [OK] Floor division used for all user-facing outputs
- [OK] Ceiling division used where explicitly required (non-binding mint side)
- [OK] No accumulation of rounding errors over operations
- [OK] Pool holdings always >= sum(user entitlements)

**Results**: [OK] **Passed** - Rounding consistently pool-favored, no exploit vectors

---

## Overflow Prevention Tests

### Property: Contract rejects operations that would overflow

**Scenario**: Attempt operations that exceed mathematical limits

**Strategy**:
1. Attempt swaps where output > reserve (should revert)
2. Attempt mints with product > 2^128 (sqrt128 should handle or revert)
3. Attempt fee calculations on amounts > practical limits
4. Verify all overflow guards trigger correctly

**Test Cases**:

#### K-Product Overflow
1. Create tier with reserves: ra = 2^50, rb = 2^50
2. Product = 2^100 (within 128-bit)
3. Attempt to grow reserves toward 2^60 x 2^60
4. Verify sqrt128 caps or reverts gracefully

**Assertion**:
- [OK] Sqrt128 handles up to ~2^61 per reserve safely

#### Fee Calculation Overflow
1. Swap amount = 2^63 (very large)
2. Fee bps = 300 (3%)
3. Calculate: (2^63 x 300) / 10,000 using mulw/divmodw
4. Verify no overflow

**Assertion**:
- [OK] Wide math prevents overflow in fee splits

#### TWAP Accumulator Overflow
1. Generate years of continuous accumulation
2. Verify 256-bit accumulators never overflow in realistic scenarios
3. Price x time products remain within 128-bit

**Assertion**:
- [OK] TWAP accumulators sufficient for practical timeframes

**Results**: [OK] **Passed** - All overflow guards functioning, supported ranges validated

---

## State Transition Edge Cases

### Property: State transitions handle boundary conditions

**Scenario**: Tiers transitioning between states at exact thresholds

**Strategy**:
1. Test activation exactly at total_lp = 2 (1 seed + 1 minted)
2. Test deactivation exactly at total_lp = 1 (all user LP burned)
3. Test bootstrap threshold exactly at k = BOOTSTRAP_THRESHOLD
4. Verify all boundary checks use correct comparisons (>, >=, <, <=)

**Test Cases**:

#### Activation Threshold
- total_lp = 1 -> Inactive [OK]
- total_lp = 2 -> Active [OK]
- Boundary: Strict greater-than (>)

#### Bootstrap Threshold
- k <= 1,000,000 -> Bootstrap mode [OK]
- k > 1,000,000 -> Normal mode [OK]
- Boundary: Inclusive less-than-or-equal (<=)

#### Reserve Underflow Guard
- new_reserve = 0 -> Reverts [OK]
- new_reserve = 1 -> Accepts [OK]
- Boundary: Reserves must be >= 1

**Assertions**:
- [OK] All boundary checks use correct operators
- [OK] No off-by-one errors in threshold logic
- [OK] State transitions atomic and deterministic

**Results**: [OK] **Passed** - Boundary conditions handled correctly

---

## Tier P Special Cases

### Property: Tier P behaves differently from standard tiers

**Scenario**: Validate Tier P's unique fee and protocol extraction rules

**Strategy**:
1. Execute swaps on Tier P
2. Verify fee calculation uses near-zero rate with minimum floor
3. Verify no protocol fee extracted (100% retention)
4. Verify Tier P receives 10% of inline spill from other tiers

**Test Cases**:

#### Near-Zero Fee Calculation
- Very small amounts -> fee floored at 1 microunit minimum
- Medium amounts -> fee proportional to amount (approaching 1 ppm)
- Large amounts -> fee scales linearly at approximately 1 part per million

**Assertions**:
- [OK] Fee floor at 1 microunit enforced
- [OK] Fee rate approaches 1 ppm for large amounts

#### No Protocol Extraction
- Execute Tier P swap
- Verify: protocol_fee_in = 0, protocol_fee_out = 0
- All fees remain in Tier P reserves

**Assertion**:
- [OK] Tier P split_protocol_fee returns (total_fee, 0)

#### Spill Allocation
- Execute swap on Tier 2 (generates protocol fees)
- Verify Tier P receives exactly 10% of protocol fee

**Assertion**:
- [OK] Tier P receives SPILL_TP_PCT (10%) of inline spill

**Results**: [OK] **Passed** - Tier P special cases implemented correctly

---

## Final Burn Edge Cases

### Property: Final burn handles all excess configurations

**Scenario**: Tiers with varying excess amounts on final burn

**Strategy**:
1. Generate random reserve states
2. Burn all user LP
3. Verify sweep captures all excess beyond seed (ra > 1, rb > 1)
4. Verify sweep handles cases where ra=1 or rb=1 exactly

**Test Cases**:

#### Large Excess Both Sides
- Reserves: 50M A, 25M B
- Final burn sweeps: 49,999,999 A, 24,999,999 B
- Tier resets: (1, 1, 1)

#### Excess One Side Only
- Reserves: 1M A, 1 B (edge case from asymmetric burns)
- Sweep: 999,999 A, 0 B
- Tier resets: (1, 1, 1)

#### No Excess (Exactly Seed)
- Reserves: 1 A, 1 B
- Sweep: 0 A, 0 B
- Tier remains: (1, 1, 1)

**Assertions**:
- [OK] Sweep = max(0, reserve - 1) per side
- [OK] Treasury incremented by sweep amounts
- [OK] Tier always reset to exact (1, 1, 1)
- [OK] No negative sweep amounts

**Results**: [OK] **Passed** - Final burn sweep robust across all reserve states

---

## Inline Spill Edge Cases

### Property: Inline spill handles all tier configurations

**Scenario**: Spill with varying numbers of active tiers

**Strategy**:
1. Test spill when 0 standard tiers active (besides current)
2. Test spill when 1 standard tier active
3. Test spill when 2+ standard tiers active
4. Verify allocation adjusts according to tier availability

**Test Cases**:

#### 0 Standard Tiers (Only Tier P Active)
- Current tier: Tier P
- Active standard tiers: none
- Allocation: 90% -> treasury, 10% -> Tier P (itself)

**Assertion**:
- [OK] 90% becomes leftover -> treasury claims

#### 1 Standard Tier Active
- Current tier: Tier 2
- Active standard: Tier 4 only
- Allocation: 10% Tier P, 90% Tier 4 (gets both 55% + 35%)

**Assertion**:
- [OK] Single tier receives full 90% standard allocation

#### Multiple Standard Tiers
- Active: Tier 0 (weakest), Tier 3 (2nd weakest), Tier 4
- Allocation: 10% Tier P, 55% Tier 0, 35% Tier 3

**Assertion**:
- [OK] Standard 10/55/35 split applies

**Results**: [OK] **Passed** - Inline spill adapts correctly to all tier configurations

---

## Test Results

[OK] **All edge case tests passed**, confirming:
- Near-zero reserves handled without underflow
- Large numbers processed correctly via wide math
- Precision boundaries favor pool (no rounding exploits)
- Overflow guards prevent exceeding mathematical limits
- State transitions occur correctly at exact thresholds
- Tier P special cases implemented as designed
- Final burn sweep robust across all reserve configurations
- Inline spill adapts to varying tier activity states

**Total edge cases explored**: Thousands across randomized parameter combinations, with zero failures.
