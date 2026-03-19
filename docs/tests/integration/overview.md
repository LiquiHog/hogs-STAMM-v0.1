# Integration Testing Overview

Integration tests validate end-to-end protocol behavior on a live AlgoKit LocalNet environment, ensuring all components work correctly together in real-world transaction scenarios.

---

## Test Environment

- **Network**: AlgoKit LocalNet (sandboxed Algorand network)
- **Framework**: pytest with custom AMM-specific assertion plugins
- **Execution**: Full contract compilation and deployment for each test run
- **Isolation**: Clean state between test cases for reproducibility

---

## Test Structure

### Pre-Test Setup
1. **LocalNet Initialization**: Spin up fresh Algorand sandbox network
2. **Factory Deployment**: Compile and deploy PoolFactory contract
3. **Pool Creation**: Deploy TieredAMM instance for test asset pair
4. **Bootstrap**: Opt into assets, create all 6 LP tokens with fee rates
5. **LP Box Registration**: Register reverse LP lookup boxes on registry contract
6. **Initial Liquidity**: Seed default tiers and add first real liquidity

### Test Execution
- **Account Management**: Multiple test accounts with funded ALGO and test ASAs
- **Balance Tracking**: Snapshot balances before and after each operation
- **State Verification**: Read contract global state to validate reserve updates
- **Transaction Groups**: Atomic groups with deposits + app calls
- **Error Validation**: Assert expected error messages on invalid operations

### Post-Test Validation
- **Invariant Checks**: K-invariant, LP supply, reserve consistency
- **Balance Reconciliation**: User balances + pool holdings = total supply
- **Logging**: Detailed operation traces for debugging and analysis

---

## Coverage Areas

The integration test suite covers:

- ✅ **Core Infrastructure** — Factory and pool deployment, bootstrap operations
- ✅ **Trading Operations** — Standard swaps, price-limited swaps, multi-tier routing
- ✅ **Liquidity Management** — All mint and burn variants (standard, hybrid, single-sided)
- ✅ **Tier Dynamics** — Auto-activation, auto-deactivation, sweep-on-final-burn
- ✅ **Multi-User Scenarios** — Concurrent operations from different accounts
- ✅ **Error Handling** — Slippage protection, invalid inputs, access control
- ✅ **Fee & Treasury** — Protocol fee extraction, treasury accumulation
- ✅ **Administrative Functions** — Governor controls, admin transfer

---

## Test Results Summary

All integration tests **passed successfully**, confirming:
- All 6 tiers function correctly across all operation types
- K-invariant maintained across swaps
- Slippage protection enforced
- Fee accounting accurate
- Tier activation/deactivation triggers work as designed
- Multi-user operations execute without conflicts
- Treasury accumulation matches expected values
- Contract sizes remain within Algorand limits
- Error conditions properly rejected

---

## Related Documentation

- [Trading Operations](trading-operations.md) — Detailed swap and routing test scenarios
- [Liquidity Management](liquidity-management.md) — Mint and burn operation tests
- [Tier Dynamics](tier-dynamics.md) — Tier lifecycle and state transition tests
