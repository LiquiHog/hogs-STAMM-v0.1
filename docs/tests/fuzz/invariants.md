# Invariant Testing

Property-based tests that validate mathematical invariants - properties that must hold under all conditions across randomized operation sequences.

---

## K-Invariant Preservation

### Property

After any swap, the product of reserves must be strictly greater than the pre-swap product:

```
k_post = reserve_a_post x reserve_b_post
k_pre = reserve_a_pre x reserve_b_pre
assert k_post > k_pre
```

This holds because fee floors guarantee at least 1 microunit is retained on both sides, causing k to strictly grow on every successful swap.

---

### Test

**Property**: K-invariant strictly grows on every swap

**Strategy**:
1. Generate random swap sequences (A->B and B->A)
2. Record k before and after each swap
3. Assert k_post > k_pre for every swap

**Parameters**:
- Swap amounts: 1 to 10^9 microunits
- Tiers: All 6 tiers with varying fee rates
- Directions: Random A->B or B->A
- Sequence length: 1 to 100 swaps per test

**Assertions**:
- `k_post > k_pre` for successful swaps (fees always applied)
- No swaps where k decreases (mathematical impossibility)
- Growth proportional to fee rate (higher fees -> larger k growth)

**Results**: [OK] **Passed** - K-invariant maintained across all generated swap sequences

---

## Fee Accounting Balance

### Property

Total fees collected must equal the sum of tier-retained fees and protocol fees. No tokens should be created or destroyed during fee processing:

```
total_fee = tier_retained + protocol_fee_in + protocol_fee_out
protocol_fee_in + protocol_fee_out = spill_to_tier_p + spill_to_weakest + spill_to_2nd_weakest + dust_to_treasury
```

---

### Test

**Property**: Fee accounting balances to zero error

**Strategy**:
1. Generate random swap with known input amount and tier
2. Calculate expected fees: `total_fee = calculate_tier_fee(amount, tier)`
3. Track fee splits:
   - Input side: `in_tier_retained + in_protocol = input_fee`
   - Output side: `out_tier_retained + out_protocol = output_fee`
   - Protocol redistribution: `in_protocol + out_protocol = tp_spill + t1_spill + t2_spill + dust`
4. Assert all fees accounted for (no creation or loss)

**Parameters**:
- Input amounts: 1 to 10^12 microunits
- Tiers: All 6 (including Tier P with 100% retention)
- Spill targets: Random weak tier configurations

**Assertions**:
- `input_fee + output_fee = total_fee` (no rounding loss)
- `tier_retained_in + tier_retained_out + protocol_total = total_fee`
- `spill_sum + dust = protocol_total` (all protocol fees distributed)
- For Tier P: `protocol_total = 0` (100% retention)

**Results**: [OK] **Passed** - Fee accounting balances perfectly across all test cases

---

## Reserve Consistency

### Property

Contract reserve state must match actual token balances minus minimum balance requirements (for ALGO) and treasury claims:

```
contract_balance_a - treasury_claim_a = sum(tier_reserves_a) for all active tiers
contract_balance_b - treasury_claim_b = sum(tier_reserves_b) for all active tiers
```

---

### Test

**Property**: Reserve state consistent with contract balances

**Strategy**:
1. Generate random operation sequence (swaps, mints, burns)
2. After each operation:
   a. Read contract global state for all tier reserves
   b. Query contract token balances
   c. Calculate expected balances from state
3. Assert reserves match balances (accounting for treasury)

**Parameters**:
- Operations: All 7 types (swap, swap_limit, mint, burn, mint_single, burn_single, mint_hybrid)
- Sequence length: 10 to 500 operations
- Users: 1 to 5 concurrent users
- Tiers: Random tier activation states

**Assertions**:
- Sum of tier reserves <= contract balance (some may be in transit or treasury)
- After accounting for treasury claims: `sum(tier_reserves) + treasury = contract_balance`
- No "phantom" tokens appearing or disappearing
- State updates atomic (no partial updates observable)

**Results**: [OK] **Passed** - Reserve consistency maintained across all operation sequences

---

## LP Supply Conservation

### Property

Changes in LP token supply must precisely match mint and burn amounts:

```
Delta total_lp = lp_minted - lp_burned
```

For any operation sequence, the sum of all mints minus the sum of all burns must equal the change in total LP supply.

---

### Test

**Property**: LP supply changes match mint/burn operations

**Strategy**:
1. Generate random mint and burn sequences across multiple tiers
2. Track cumulative:
   - `total_minted = sum(all mint operations)`
   - `total_burned = sum(all burn operations)`
   - `lp_supply_change = final_lp - initial_lp` (per tier)
3. Assert `lp_supply_change = total_minted - total_burned` per tier

**Parameters**:
- Operations: mint, burn, mint_single, burn_single, mint_hybrid
- Amounts: Random LP quantities from 1 to 10^9
- Tiers: All 6 tiers
- Sequence length: 10 to 100 operations

**Assertions**:
- `total_lp_per_tier = seed_lp + minted - burned` (exact equality)
- Seed LP (1 per tier) never burned
- No fractional LP (all amounts uint64)
- LP supply never negative

**Results**: [OK] **Passed** - LP supply conservation holds exactly across all test sequences

---

## Constant Product Formula

### Property

Swap output must match the constant-product formula:

Swap output is calculated using constant-product mechanics, where the effective input (after fees) is used to maintain the reserve product invariant.

---

### Test

**Property**: Constant-product formula implemented correctly

**Strategy**:
1. Generate random swap with known reserves and input
2. Calculate expected output using constant-product reference implementation
3. Execute swap via contract
4. Assert contract output matches expected (accounting for output fee)

**Parameters**:
- Input amounts: 1 to 10^9
- Reserves: 1,000 to 10^12 (bootstrap to deep liquidity)
- Tiers: All 6 fee rates

**Assertions**:
- Contract raw output (before output fee) = `dy_expected`
- User receives `dy_expected - output_fee`
- Formula holds for both A->B and B->A swaps
- Wide math (mulw/divmodw) produces exact results

**Results**: [OK] **Passed** - Constant-product formula implemented exactly

---

## TWAP Accumulator Monotonicity

### Property

TWAP accumulators must never decrease (they are cumulative):

```
accumulator_a_after >= accumulator_a_before
accumulator_b_after >= accumulator_b_before
```

Equality holds only if elapsed time = 0 or reserves = 0.

---

### Test

**Property**: TWAP accumulators monotonically increase over time

**Strategy**:
1. Generate random operation sequences with time progression
2. Record TWAP accumulator values before and after each operation
3. Calculate expected increment: `price x elapsed_time`
4. Assert accumulators increase by expected amount

**Parameters**:
- Time steps: 0 to 3600 seconds between operations
- Reserves: Varying liquidity states
- Operations: All types (all trigger `_update_twap`)

**Assertions**:
- If `elapsed > 0` and `reserves > 0`: accumulator increases
- If `elapsed = 0` or `reserves = 0`: accumulator unchanged
- Increment = `(aggregate_price_scaled * elapsed)` (128-bit addition)
- No overflow (128-bit storage sufficient for years of accumulation)

**Results**: [OK] **Passed** - TWAP accumulators increase correctly over time

---

## Aggregate Reserve Invariant

### Property

Aggregate reserves must equal the sum of all active tier reserves:

```
agg_ra = sum(tier_ra for all active tiers)
agg_rb = sum(tier_rb for all active tiers)
```

---

### Test

**Property**: Aggregate reserves match sum of active tier reserves

**Strategy**:
1. Generate random operation sequences that activate/deactivate tiers
2. After each operation:
   a. Read `agg_ra`, `agg_rb` from global state
   b. Sum all active tier reserves (check tier_mask for active status)
   c. Assert `agg_ra = sum(active_tier_ra)`, `agg_rb = sum(active_tier_rb)`

**Parameters**:
- Operations: All types (especially mints/burns that change tier states)
- Tiers: Random activation patterns (0 to 8 active tiers)
- Sequence length: 50 to 200 operations

**Assertions**:
- Aggregate equals sum of active tiers (exact equality)
- Inactive tier reserves not included in aggregate
- Updates atomic with tier state changes
- No drift or accumulation errors

**Results**: [OK] **Passed** - Aggregate reserves accurate across all tier states

---

## Test Results

[OK] **All invariant tests passed**, confirming:
- K-invariant preserved across all swap sequences (monotonically increasing)
- Fee accounting balances perfectly (no token creation/destruction)
- Reserve state consistent with contract balances (no phantom tokens)
- LP supply conserved exactly (mints - burns = Delta supply)
- Constant-product formula implemented correctly (matches reference)
- TWAP accumulators increase monotonically over time (no decreases)
- Aggregate reserves match sum of active tiers (no drift)

**Total examples validated**: Hundreds of thousands across all property tests, with zero failures.
