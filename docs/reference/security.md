# Security

STAMM uses layered protections across pool math, governance controls, and infrastructure contracts.

---

## Pool Security

### Invariant Enforcement

Swaps enforce strict constant-product growth (`k_post > k_pre`). Two-sided fee floors guarantee positive reserve retention on successful swaps.

### Reserve Minimums and Seed State

- Tiers operate with a protected seed baseline (`ra=1`, `rb=1`, `lp=1`).
- Final-burn flow sweeps excess to treasury claims and returns tier state to seed values.
- Reserve-underflow checks prevent draining to zero.

### Rounding and Arithmetic Safety

- User-facing outputs use pool-favoring floor arithmetic.
- Wide math (`mulw`, `divmodw`, helper `safe_mul_div`) avoids `uint64` overflow.
- `sqrt128` is used for single-sided and bounded-price calculations.
- TWAP accumulators use 256-bit carry-propagated storage.

### Slippage and Input Validation

- Swaps and liquidity methods enforce min-output/min-LP checks.
- Deposit transaction types are validated (including ALGO `Payment` vs ASA `AssetTransfer`).
- Tier-index and LP-asset checks prevent cross-tier misuse.

### ALGO-Specific Protections

- Outgoing ALGO transfers enforce MBR safety checks.
- ALGO deposits disallow close-out behavior.

---

## Governance and Infra Security

### Governor Model

- Pools are governed by the admin contract after registration.
- Factory cannot perform pool governor actions after handoff.
- Governor migration is timelocked and executed per pool.

### Admin Contract Controls

- 7-day timelocks for admin transfer, treasury transfer, and governor migration.
- Role separation: `admin` (governance), `treasury` (withdrawals), `guardian` (cancellation veto).
- Update/delete are permanently blocked by `on_delete_or_update -> op.err()`.

### Factory Controls

- 7-day timelocks for admin transfer, admin-contract changes, registry changes, and unfreeze.
- `freeze_factory` is instant and blocks template-management methods.
- Update/delete are permanently blocked by `on_delete_or_update -> op.err()`.
- Pool setup enforces creator checks and registration ordering.

### Registry Controls

- Single `writer` address authorized for pair/LP writes.
- `register_lps` verifies pool creator equals writer.
- 72-hour admin-transfer timelock.
- No freeze mode; update/delete remain admin-gated.
- Box-MBR payment checks are enforced in write methods (`register_pair`, `register_lps`).

### Registration and Funding Guards

- `seed_and_mint` requires `registered == 1`, ensuring reverse LP boxes exist before liquidity flow.
- Seed-payment sender/receiver and rekey/close fields are validated in setup paths.

---

## Operational Notes

- Protocol relies on account-level operational security for `admin`, `treasury`, and `guardian` keys.
- TWAP remains timestamp-dependent and should be consumed over meaningful windows.
- Router and spill behavior can shift local tier prices within a block; aggregate TWAP mitigates transient manipulation over longer windows.
