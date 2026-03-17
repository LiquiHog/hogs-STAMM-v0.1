# Fee Engine

STAMM has a multi-layered fee system that extracts protocol revenue from swaps and redistributes it inline to strengthen the pool. No fees are charged on standard mints or burns — only the internal swap portions of hybrid and single-sided operations incur fees.

---

## Fee Flow Overview

```
Swap Amount
    |
    v
[Two-Sided Fee] --> Half fee on input, half on output
    |
    v
Each half split: 80% stays in tier reserves (tier-retained)
                 20% extracted as protocol fee
    |
    v
[Inline Spill] --> Tier P: 10%
                    Weakest tier: 55%
                    2nd weakest tier: 35%
    |
    v
[Dust] --> Treasury claims (tr_a/tr_b)
```

## Step by Step

### 1. Two-Sided Fee

Each swap charges fees on both the input and output sides. For a tier with fee rate `f`:

- **Input fee** = `floor(f / 2)` of the input amount, floored to 1 microunit
- **Output fee** = remainder applied to the raw output, floored to 1 microunit

Both sides are floored to a minimum of 1 microunit. This guarantees that every successful swap grows the k-invariant, even for very small trades where the calculated fee would otherwise round to zero.

This produces protocol fees in both assets on every swap, enabling balanced inline redistribution without a separate balancing mechanism.

| Tier Type | Fee Calculation |
|---|---|
| Standard (0-5) | Proportional to tier's fee rate (in basis points), split across input and output (each side min 1 micro) |
| Tier P | Approximately 1 part per million, with minimum 1 microunit floor (each side min 1 micro) |

### 2. Tier Retained vs. Protocol

For standard tiers, each half-fee is split:
- **80%** stays in the tier's reserves (benefits LPs directly)
- **20%** is extracted as the protocol fee

Tier P keeps **100%** of its fee — no protocol extraction. This protects Tier P and keeps the near-zero fee tier attractive.

Combined with the per-side fee floor of 1 microunit, the tier-retained portion guarantees the k-invariant always grows on every successful swap.

### 3. Inline Spill

Protocol fees from both sides are redistributed immediately during the same swap transaction. No accumulation, no keeper, no separate call.

The redistribution targets three tiers:

| Recipient | Share | Selection |
|---|---|---|
| Tier P | 10% | Always (if active and seeded) |
| Weakest tier | 55% | Lowest k-value among active standard tiers |
| 2nd weakest tier | 35% | Second-lowest k-value |

The weakest tiers are determined by an inline O(6) scan during each swap. The scan excludes the tier being operated on to prevent state conflicts (the caller has a pending state update that would overwrite the spill's changes). If only one standard tier is active (besides the current), it receives the full 90% standard allocation. If no standard tiers qualify, the 90% goes to treasury claims.

### 4. LP Minting and Treasury Claims

For each recipient tier, the spill amounts are added to the tier's reserves and proportional LP tokens are minted. These LP tokens stay in the pool — the treasury's share is tracked in `t{c}_tl` state for later withdrawal by the admin.

If the spill amount is too small to mint any LP (dust), the tokens go to treasury asset claims (`tr_a`/`tr_b`) instead.

Inactive but seeded tiers can also receive spill — amounts are added at the aggregate reserve ratio and LP is minted proportionally.

---

## Treasury Claims (Pull Model)

All protocol revenue is stored in pool state rather than transferred to a separate contract:

| Claim Type | State Key | Source |
|---|---|---|
| LP tokens | `t{c}_tl` per tier | Minted during inline spill |
| Asset A | `tr_a` | Dust from spill |
| Asset B | `tr_b` | Dust from spill |

The admin withdraws claims via factory proxy methods:
- `withdraw_pool_lp(pool, tier, amount, receiver, lp_asset)`
- `withdraw_pool_assets(pool, a_asset, b_asset, a_amount, b_amount, receiver)`

This pull model eliminates the need for treasury opt-ins, reduces inner transactions during trading, and ensures withdrawals never block normal pool operations.

---

## Fee Parameters

| Parameter | Value | Description |
|---|---|---|
| Protocol share | 20% | Portion of tier fee extracted as protocol fee |
| Tier retained | 80% | Portion of tier fee that stays in reserves |
| Tier P spill share | 10% | Portion of protocol fee to Tier P |
| Weakest tier share | 55% | Portion of protocol fee to weakest tier |
| 2nd weakest share | 35% | Portion of protocol fee to 2nd weakest tier |

These values are currently hardcoded in the contract. They may be made configurable in future updates.
