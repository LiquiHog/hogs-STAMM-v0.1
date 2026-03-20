# Liquidity Management Testing

Integration tests for all mint and burn operations: standard, hybrid, and single-sided variants.

---

## Standard Mint (Two-Sided) Tests

### Bootstrap Mint

**Test**: First real deposit into seeded tier (total_lp = 1)

**Scenario**:
1. Tier has seed state: reserve_a = 1, reserve_b = 1, total_lp = 1
2. User deposits 1,000,000 microunits of A and 2,000,000 microunits of B
3. Contract accepts any ratio (user sets initial price)
4. LP tokens calculated using geometric mean approach to ensure fair initial distribution
5. New reserves = seed + deposit (ADD semantics)

**Assertions**:
- LP tokens calculated via geometric mean of deposits
- New reserves = 1 + deposit_a, 1 + deposit_b (includes seed)
- Total LP = 1 + minted (seed LP remains locked)
- Tier auto-activates (total_lp > 1)
- No refund issued (any ratio accepted)

---

### Proportional Mint

**Test**: Deposit matching tier's current ratio

**Scenario**:
1. Tier has reserves 10M A : 20M B (ratio 1:2)
2. User deposits 1M A + 2M B (matching ratio)
2. Both sides contribute proportionally to LP mint
3. LP minted determined by binding side (whichever contributes less proportionally)

**Assertions**:
- LP minted calculated from binding side
- Used amounts match deposits exactly (no refund)
- New reserves = old reserves + deposits
- LP tokens transferred to user
- Tier remains active

---

### Refund Excess Deposit

**Test**: Deposit with one side exceeding required ratio

**Scenario**:
1. Tier has reserves 10M A : 20M B (ratio 1:2)
2. User deposits 1M A + 3M B (excess B)
3. Binding side: A (1M A matches 2M B at 1:2 ratio)
4. Used: 1M A, 2M B (ceil division to avoid underpayment)
5. Refund: 0 A, 1M B

**Assertions**:
- LP minted from binding side (A in this case)
- Excess B refunded atomically
- Non-binding side uses ceiling division (lp x reserve / total_lp + 1)
- User receives LP + refund in same transaction

---

### Slippage Protection

**Test**: Mint reverts if LP minted < min_lp_out

**Scenario**:
1. User expects 1M LP but only 900K would be minted
2. User sets min_lp_out = 1,000,000
3. Transaction reverts: `"slippage"`

**Assertions**:
- No state changes on revert
- User retains deposited assets
- Slippage parameter protects against price changes between submission and execution

---

## Hybrid Mint Tests

### Excess A Swapped

**Test**: Deposit with excess A, swap before mint

**Scenario**:
1. Tier has reserves 10M A : 20M B (ratio 1:2)
2. User deposits 2M A + 2M B
3. 1M A matches 2M B at ratio -> 0.5M A excess
4. Contract swaps 0.5M A for B (incurs tier fee)
5. Balanced amounts minted as LP

**Assertions**:
- Excess A identified: lp_from_a > lp_from_b
- Swap portion incurs two-sided fee
- Protocol fee redistributed via inline spill
- No refund issued (all assets used)
- LP minted from post-swap balanced amounts

---

### Excess B Swapped

**Test**: Deposit with excess B, swap before mint

**Scenario**:
1. Tier has reserves 10M A : 20M B (ratio 1:2)
2. User deposits 1M A + 4M B
3. 1M A matches 2M B -> 2M B excess
4. Contract swaps 2M B for A
5. Balanced amounts minted as LP

**Assertions**:
- Excess B identified: lp_from_b > lp_from_a
- Swap executes with fee deduction
- All deposited assets used (no refund)
- More capital-efficient than standard mint with refund

---

### Bootstrap Fallback

**Test**: Hybrid mint on bootstrap tier behaves like standard mint

**Scenario**:
1. Tier in bootstrap state (total_lp = 1, reserves 1:1)
2. User calls mint_hybrid with any ratio deposit
3. Contract skips swap logic
4. Executes bootstrap mint: LP = sqrt(deposit_a x deposit_b)

**Assertions**:
- No swap occurs in bootstrap state
- Deposits added directly to seed reserves (ADD semantics)
- Same behavior as standard bootstrap mint

---

## Single-Sided Mint Tests

### Optimal Swap Split Calculation

**Test**: Deposit A only, contract computes optimal A->B swap amount

**Scenario**:
1. Tier has reserves 10M A : 20M B
2. User deposits 1M A only
3. Contract computes optimal swap split using wide arithmetic to maximize balanced LP output
4. Swaps calculated portion of A for B
5. Mints LP from remaining A + received B

**Assertions**:
- Optimal swap amount calculated using wide arithmetic (no overflow)
- Swap incurs tier fee
- Residual amounts (rounding dust) refunded to user
- LP minted from balanced post-swap amounts
- More user-friendly than requiring both assets

---

### Single B Deposit

**Test**: Deposit B only, swap B->A before minting

**Scenario**:
1. User deposits B only
2. Contract computes optimal B->A swap
3. Mints LP from swapped A + remaining B

**Assertions**:
- Symmetric behavior to single A deposit
- Same fee application and residual refund logic

---

### Bootstrap Rejection

**Test**: Single-sided mint reverts on bootstrap tier

**Scenario**:
1. Tier in bootstrap state (total_lp = 1)
2. User attempts single-sided mint
3. Transaction reverts: `"bootstrap needs both"`

**Assertions**:
- No established ratio to target in bootstrap state
- Single-sided mint requires active tier
- Error message clear

---

## Standard Burn (Two-Sided) Tests

### Proportional Withdrawal

**Test**: Burn LP for proportional A + B withdrawal

**Scenario**:
1. Tier has reserves 10M A : 20M B, total_lp = 5M
2. User burns 1M LP
3. Receives: (1M / 5M) x 10M A = 2M A, (1M / 5M) x 20M B = 4M B
4. Floor division favors pool (prevents rounding exploits)

**Assertions**:
- Output amounts = (lp_burned x reserve) / total_lp (floor division)
- Total LP reduced by burned amount
- Reserves reduced by output amounts
- User receives both assets
- Slippage protection: output >= min_a_out, output >= min_b_out

---

### Final Burn and Sweep

**Test**: Burn all user LP, tier resets to seed state

**Scenario**:
1. Tier has reserves 10.000001 A : 20.000001 B, total_lp = 1 + user_lp
2. User burns all user LP
3. Total LP returns to 1 (seed LP)
4. Reserves reset to 1:1 (seed state)
5. Excess reserves (10M A, 20M B) swept to treasury claims

**Assertions**:
- Final burn allowed even when tier deactivates
- Tier mask updated (tier marked inactive)
- Excess reserves swept: tr_a += excess_a, tr_b += excess_b
- Tier returns to seed state: (1, 1, 1)
- Slippage protection still enforced on final burn

---

### Slippage Protection

**Test**: Burn reverts if output < min_a_out or min_b_out

**Scenario**:
1. User expects 2M A and 4M B
2. Sets min_a_out = 2,000,000, min_b_out = 4,000,000
3. If actual output = 1.999999M A, transaction reverts: `"slippage a"`

**Assertions**:
- Both sides protected independently
- Error message indicates which asset failed slippage check
- No partial execution (atomic revert)

---

## Single-Sided Burn Tests

### Burn for A Only

**Test**: Burn LP, receive A only (contract swaps B internally)

**Scenario**:
1. User burns 1M LP
2. Contract calculates proportional withdrawal: 2M A, 4M B
3. Contract internally swaps 4M B -> A
4. User receives total A (2M + swap output)

**Assertions**:
- Internal swap incurs tier fee
- User receives single asset
- Slippage protection on final A output
- More convenient than receiving both assets

---

### Burn for B Only

**Test**: Burn LP, receive B only (contract swaps A internally)

**Scenario**:
1. User burns LP
2. Proportional withdrawal calculated
3. Internal A -> B swap
4. User receives total B

**Assertions**:
- Symmetric to burn-for-A behavior
- Fee applies to swap portion
- Single slippage parameter for B output

---

### Bootstrap Rejection

**Test**: Single-sided burn reverts on bootstrap tier

**Scenario**:
1. Tier in bootstrap state
2. User attempts single-sided burn
3. Transaction reverts: `"tier in bootstrap"`

**Assertions**:
- No established ratio for internal swap
- Standard burn still allowed

---

## Multi-User Liquidity Tests

### Concurrent Mints

**Test**: Multiple users mint to same tier

**Scenario**:
1. User A mints 1M A + 2M B -> receives LP_A
2. User B mints 500K A + 1M B -> receives LP_B
3. Reserves update correctly after each mint
4. Total LP = initial + LP_A + LP_B

**Assertions**:
- Each mint calculates LP independently
- Reserves accumulate correctly
- No state conflicts between users
- Each user's LP share accurate

---

### Mint After Burn

**Test**: User A burns, User B mints (same tier)

**Scenario**:
1. User A burns LP, withdraws assets
2. Reserves decrease
3. User B mints, adds liquidity
4. Reserves increase

**Assertions**:
- Operations execute correctly in sequence
- Tier remains active if total_lp > 1
- Reserve ratio may shift slightly due to floor division
- K-invariant maintained

---

## Edge Case Tests

### Zero LP Mint Prevention

**Test**: Mint reverts if calculated LP = 0

**Scenario**:
1. User deposits very small amounts
2. LP calculation rounds to zero
3. Transaction reverts: `"LP minted is zero"`

**Assertions**:
- Prevents zero-value LP issuance
- Protects against dust exploitation
- User receives clear error message

---

### Wrong LP Asset Rejection

**Test**: Burn reverts if wrong tier's LP token sent

**Scenario**:
1. User sends Tier 2 LP token to Tier 4 burn call
2. Transaction reverts: `"wrong LP asset"`

**Assertions**:
- LP asset ID validated against tier state
- Prevents cross-tier LP token confusion
- Error message clear

---

## Test Results

[OK] **All liquidity management tests passed**, confirming:
- Standard mints work correctly in bootstrap and proportional modes
- Hybrid mints swap excess and use all deposited assets
- Single-sided mints calculate optimal swap splits without overflow
- Standard burns withdraw proportionally with slippage protection
- Single-sided burns execute internal swaps correctly
- Final burns sweep excess and reset tiers to seed state
- Multi-user operations execute safely and correctly
- All edge cases and error conditions handled properly
