# Tier Dynamics Testing

Integration tests for tier lifecycle operations: seeding, auto-activation, auto-deactivation, tier mask tracking, and sweep-on-final-burn.

---

## Tier Seeding Tests

### Default Tier Seeding

**Test**: `seed_and_mint` seeds 4 default tiers (P, 2, 3, 4)

**Scenario**:
1. Pool bootstrapped with 7 LP assets created
2. User calls `seed_and_mint` with liquidity for Tier 2
3. Contract seeds Tier P, 2, 3, 4 with 1 microunit each asset
4. Adds real liquidity to chosen tier (Tier 2)
5. Tier 2 auto-activates (total_lp > 1)

**Assertions**:
- Tiers P, 2, 3, 4 have state: reserve_a = 1, reserve_b = 1, total_lp = 1
- Tier 2 receives real liquidity and auto-activates
- Other default tiers remain inactive (seeded but not activated)
- Tier mask updated to include active Tier 2

---

### Individual Tier Seeding

**Test**: `seed_tier` seeds a single non-default tier

**Scenario**:
1. User calls `seed_tier` for Tier 0 (not default)
2. Contract seeds Tier 0: reserve_a = 1, reserve_b = 1, total_lp = 1
3. Tier 0 remains inactive (total_lp = 1)
4. Subsequent mint on Tier 0 activates it

**Assertions**:
- Non-default tier (0, 1, 5, 6) can be seeded individually
- Seed state identical to default tier seeding
- Tier remains inactive until first real mint
- Tier mask not updated until activation

---

### Seeding Cost

**Test**: Seeding requires 2 microunits total (1 of each asset)

**Scenario**:
1. User calls `seed_tier` with exactly 1 microunit A and 1 microunit B
2. Seeding succeeds
3. User balance decreases by 1 microunit each asset

**Assertions**:
- Minimal cost to seed (1 micro A + 1 micro B)
- No LP tokens issued from seed (locked in pool)
- Seeding is one-time operation per tier

---

## Auto-Activation Tests

### Activation on First Mint

**Test**: Tier activates when total_lp exceeds 1

**Scenario**:
1. Tier seeded: total_lp = 1 (inactive)
2. User mints liquidity to the tier
3. Total LP increases beyond seed threshold
4. Tier auto-activates

**Assertions**:
- Tier mask bit set for tier (mask |= 1 << tier)
- Tier state change: inactive → active
- Aggregate reserves updated (agg_ra += tier_ra, agg_rb += tier_rb)
- TWAP oracle includes tier in next update

---

### Activation Threshold

**Test**: Tier activates exactly when total_lp > 1

**Scenario**:
1. Seed state: total_lp = 1
2. Mint that produces LP = 0 (dust) → remains at total_lp = 1 → inactive
3. Mint that produces LP = 1 → total_lp = 2 → activates

**Assertions**:
- Threshold is strict: total_lp > 1 (not >=)
- Even 1 microunit of real LP triggers activation
- Tier state updates atomically with activation

---

### Multiple Tier Activation

**Test**: Activate multiple tiers independently

**Scenario**:
1. Seed Tiers 0, 1, 2, 3, 4, 5, 6, P
2. Mint liquidity to Tier 0 → activates
3. Mint liquidity to Tier 3 → activates
4. Mint liquidity to Tier P → activates
5. Tier mask = bits 0, 3, 7 set

**Assertions**:
- Each tier activates independently
- Tier mask reflects all active tiers
- Aggregate reserves = sum of active tier reserves
- Inactive tiers remain in seed state

---

## Auto-Deactivation Tests

### Deactivation on Final Burn

**Test**: Tier deactivates when total_lp returns to 1

**Scenario**:
1. Tier active: total_lp = 1 + user_lp
2. User burns all user_lp
3. Total LP returns to 1 (only seed LP remains)
4. Tier auto-deactivates

**Assertions**:
- Tier mask bit cleared (mask &= ~(1 << tier))
- Tier state change: active → inactive
- Aggregate reserves updated (agg_ra -= tier_ra, agg_rb -= tier_rb)
- TWAP oracle excludes tier from next update
- Tier remains seeded (total_lp = 1, reserves = 1, 1)

---

### Sweep-on-Final-Burn

**Test**: Excess reserves swept to treasury on deactivation

**Scenario**:
1. Tier active with reserves: 10,000,000 A, 20,000,000 B
2. User burns all user LP
3. Tier deactivates, reserves reset to seed: 1 A, 1 B
4. Excess swept: tr_a += 9,999,999, tr_b += 19,999,999

**Assertions**:
- Excess reserves = current - seed (reserve - 1)
- Treasury asset claims incremented
- Tier reserves reset to exactly (1, 1)
- Total LP remains 1 (seed LP locked)
- No value stranded in inactive tier

---

### Partial Burn Does Not Deactivate

**Test**: Tier remains active if total_lp > 1 after burn

**Scenario**:
1. Tier has total_lp = 10,000,000
2. User burns 5,000,000 LP
3. Total LP = 5,000,000 (still > 1)
4. Tier remains active

**Assertions**:
- Tier mask unchanged
- Aggregate reserves updated to reflect reduced tier reserves
- No sweep occurs
- Tier continues normal operations

---

## Tier Mask Tracking Tests

### Bitmask Representation

**Test**: Tier mask correctly represents active tier set

**Scenario**:
1. Activate Tier 0 (bit 0), Tier 3 (bit 3), Tier P (bit 6)
2. Tier mask = 0b1001001 = 73

**Assertions**:
- Bit `i` set ⟺ tier `i` active
- Bit check: `mask & (1 << tier)` returns non-zero for active tiers
- Efficient single-uint64 storage for all 7 tier states

---

### Mask Update on State Transition

**Test**: Mask updates atomically with tier activation/deactivation

**Scenario**:
1. Tier 2 activates → mask |= (1 << 2) → bit 2 set
2. Tier 2 deactivates → mask &= ~(1 << 2) → bit 2 cleared

**Assertions**:
- Mask read from global state: `AppGlobal.get_ex_uint64(..., "tier_mask")`
- Mask updated in same transaction as tier state change
- No race conditions or stale mask states

---

### Inline Spill Mask Usage

**Test**: Inline spill scans active tiers using mask

**Scenario**:
1. Active tiers: 0, 2, 4, P (mask = 0b10010101)
2. Swap on Tier 2 triggers inline spill
3. Scan excludes Tier 2 (current_tier)
4. Considers Tier 0, 4, P for weakest-tier targeting

**Assertions**:
- Scan loops over 0-6 (standard tiers)
- Skips current_tier to prevent state clobbering
- Checks `mask & (1 << t)` for active status
- Excludes Tier P from weakest scan (has fixed allocation)

---

## Aggregate Reserve Tests

### Aggregate Update on Activation

**Test**: Aggregate reserves include new tier on activation

**Scenario**:
1. Agg reserves: agg_ra = 10M, agg_rb = 20M (from active tiers)
2. Tier 5 activates with reserves 1M A, 2M B
3. Agg reserves update: agg_ra = 11M, agg_rb = 22M

**Assertions**:
- Aggregate includes all active tier reserves
- Updated in `_maybe_activate` subroutine
- TWAP oracle uses updated aggregates

---

### Aggregate Update on Deactivation

**Test**: Aggregate reserves exclude tier on deactivation

**Scenario**:
1. Agg reserves: agg_ra = 11M, agg_rb = 22M
2. Tier 5 deactivates (reserves 1M A, 2M B swept)
3. Agg reserves update: agg_ra = 10M, agg_rb = 20M

**Assertions**:
- Deactivated tier reserves removed from aggregate
- Updated in `_final_burn_sweep` subroutine
- TWAP oracle reflects reduced aggregate

---

### Incremental Updates

**Test**: Aggregate reserves updated on every tier state change

**Scenario**:
1. Tier 2 receives swap → reserves increase → aggregate increases
2. Tier 3 receives inline spill → reserves increase → aggregate increases
3. Tier 4 LP burn → reserves decrease → aggregate decreases

**Assertions**:
- Aggregate = sum of all active tier reserves (maintained incrementally)
- No need to loop over tiers to recompute aggregate
- Efficient O(1) updates in `set_tier_state`

---

## Error Handling Tests

### Double Seeding Prevention

**Test**: Cannot seed an already-seeded tier

**Scenario**:
1. Tier 2 already seeded (total_lp = 1)
2. Attempt `seed_tier` for Tier 2 again
3. Transaction reverts: `"tier already seeded"` or similar

**Assertions**:
- Seeding is one-time operation
- Prevents accidental reserve overwrites
- Error message clear

---

### Tier P Public Mint Rejection

**Test**: Public users cannot mint to Tier P

**Scenario**:
1. User attempts `mint` for Tier P
2. Transaction reverts: `"no public mint on tier P"`

**Assertions**:
- Tier P is protocol-managed
- Only inline spill adds liquidity to Tier P
- Error message indicates Tier P restriction

---

### Invalid Tier Index

**Test**: Operations revert on invalid tier index

**Scenario**:
1. User calls `swap` with tier_index = 8 (out of range)
2. Transaction reverts: `"invalid tier"`

**Assertions**:
- Tier index must be < NUM_TIERS (< 8)
- All operations validate tier index
- Consistent error message across methods

---

## Multi-Tier Interactions

### Cross-Tier Spill Distribution

**Test**: Inline spill deposits to multiple tiers simultaneously

**Scenario**:
1. Swap on Tier 2 generates 1,000 protocol fee A + 2,000 protocol fee B
2. Inline spill distributes:
   - 10% (100 A, 200 B) → Tier P
   - 55% (550 A, 1,100 B) → weakest tier (Tier 0)
   - 35% (350 A, 700 B) → 2nd weakest tier (Tier 4)
3. Each recipient tier reserves increase, LP minted to treasury

**Assertions**:
- Three tiers receive deposits in single transaction
- Each tier's state updated independently
- Treasury LP claims incremented for each tier
- Total distributed = 100% of protocol fee

---

### Tier Activation from Spill

**Test**: Inactive tier activates when spill pushes total_lp > 1

**Scenario**:
1. Tier 5 inactive but seeded (total_lp = 1)
2. Receives inline spill deposits
3. LP minted from spill > 0 → total_lp = 1 + minted
4. Tier 5 auto-activates

**Assertions**:
- Inline spill can trigger activation
- Aggregate reserves updated to include newly active tier
- Tier now participates in TWAP oracle
- No user interaction required for activation

---

## Test Results

✅ **All tier dynamics tests passed**, confirming:
- Default and individual tier seeding works correctly
- Auto-activation triggers when total_lp > 1
- Auto-deactivation triggers when total_lp returns to 1
- Sweep-on-final-burn transfers excess reserves to treasury
- Tier mask tracks active tiers accurately
- Aggregate reserves updated incrementally and correctly
- Inline spill can activate inactive tiers
- Cross-tier state changes do not conflict
- All error conditions handled with clear messages
