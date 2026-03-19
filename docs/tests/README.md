# Testing & Validation

STAMM has been extensively tested through comprehensive integration tests and property-based fuzz testing to ensure correctness, safety, and robustness across all protocol features.

---

## Test Coverage

### Integration Tests

The integration test suite validates end-to-end protocol behavior on a live AlgoKit LocalNet environment, covering:

#### Core Infrastructure
- **Factory Deployment**: Factory contract compilation, deployment, and initialization
- **Pool Creation**: Multi-tier pool instantiation with automatic LP token creation for all 6 tiers
- **Bootstrap Operations**: Initial liquidity seeding and tier setup

#### Trading Operations
- **Standard Swaps**: Constant-product trades with fee application and slippage protection
- **Price-Limited Swaps**: Partial execution swaps with atomic refunds when price limits are hit
- **Multi-Tier Routing**: Trades across different fee tiers (3 bps to 300 bps)
- **Caller-Directed Routing**: Explicit (tier, amount) legs via `swap_routed` with 1-6 legs
- **Asset Pair Support**: Both ASA/ASA and ALGO/ASA pool configurations

#### Liquidity Management
- **Two-Sided Mint**: Proportional deposits of both assets for LP tokens
- **Two-Sided Burn**: Proportional LP token redemption for both underlying assets
- **Single-Sided Mint**: Deposit one asset with optimal internal swap split (using 128-bit sqrt math)
- **Single-Sided Burn**: Redeem LP tokens for a single asset via burn + internal swap
- **Hybrid Mint**: Deposit with automatic excess swapping to maintain pool ratios

#### Tier Dynamics
- **Auto-Activation**: Tiers automatically activate when receiving first real liquidity (total LP > 1)
- **Auto-Deactivation**: Tiers automatically deactivate when all user LP is burned (total LP == 1)
- **Sweep-on-Final-Burn**: Excess reserves transferred to treasury when tier resets to seed state
- **Tier Mask Tracking**: Bitmask-based active tier monitoring verified across operations

#### Multi-User Scenarios
- **Concurrent Operations**: Multiple users performing swaps, mints, and burns simultaneously
- **Interleaved Transactions**: Mixed operation sequences from different accounts
- **Balance Tracking**: User balance checkpoints validated throughout test lifecycle

#### Error Handling & Edge Cases
- **Slippage Protection**: Min output / max input validation for all operations
- **Invalid LP Token**: Rejection of wrong-tier LP tokens in tier-specific operations
- **Insufficient Balance**: Proper error handling for underfunded operations
- **Zero-Amount Guards**: Prevention of zero-value transactions
- **Large Trade Handling**: Swap execution with significant liquidity amounts

#### Fee & Treasury Operations
- **Protocol Fee Extraction**: Validation of fee splits between tier-retained and protocol treasury
- **Treasury Accumulation**: Tracking of collected fees across multiple operations
- **Sweep Operations**: Correct reserve cleanup on tier deactivation

#### Administrative Functions
- **Governor Controls**: Admin operations restricted to admin contract (as governor)
- **Governor Transfer**: Factory transfers governor to admin contract during registration
- **Admin Transfer**: 7-day timelock on factory admin transfer; 72-hour timelock on admin contract admin transfer
- **Factory Freeze/Unfreeze**: Instant freeze, 7-day timelock unfreeze
- **Admin Contract Freeze**: 72-hour timelock propose/confirm, irreversible
- **Governor Migration**: 72-hour timelock propose/confirm per pool via admin contract; functional even when admin contract is frozen
- **Registry Management**: Factory registry change with 7-day timelock (propose/confirm/cancel)
- **Registry Freeze**: Permanent/irreversible freeze blocking all writes; reads remain functional
- **Registry Writer Authorization**: Only the factory (writer) can register pair/LP data
- **Treasury Proxy**: Admin contract withdraws treasury claims from pools (`withdraw_pool_lp`, `withdraw_pool_assets`)
- **Access Control**: Rejection of unauthorized admin calls

#### Contract Validation
- **Size Constraints**: Factory and AMM contracts verified to fit within Algorand's program size limits
- **ABI Compatibility**: Verifying method signatures remain stable across updates
- **State Management**: Global and local state updates validated across operations

### Test Results

All integration tests **passed successfully**, validating:
- ✅ All 6 tiers function correctly across all operation types
- ✅ K-invariant maintained across swaps (reserves × fees preserved)
- ✅ Slippage protection enforced for all operations
- ✅ Fee accounting accurate across single and multi-tier operations
- ✅ Tier activation/deactivation triggers work as designed
- ✅ Multi-user operations execute correctly without conflicts
- ✅ Treasury accumulation matches expected protocol fee extraction
- ✅ Contract sizes remain within Algorand limits with comfortable margins
- ✅ Error conditions properly rejected with appropriate error messages

---

## Fuzz Testing

### Property-Based Validation

The fuzz test suite uses Hypothesis for property-based testing, generating randomized operation sequences to verify mathematical invariants and protocol safety properties:

#### Core Invariants
- **K-Invariant Preservation**: After any swap, `reserves_in × reserves_out` must be greater than or equal to the pre-swap product (accounting for fees)
- **Fee Accounting**: Total fees collected must equal sum of tier-retained and protocol treasury fees
- **Reserve Consistency**: Reserves must always remain positive and match contract state after operations
- **LP Conservation**: Total LP supply changes must precisely match mint/burn amounts

#### Mathematical Operations
- **Wide Arithmetic**: Geometric mean and other wide arithmetic functions validated against reference implementations across full input ranges
- **Multiply-Divide**: `mul_div` floor division tested for correctness without overflow
- **Constant Product Formula**: Output calculations verified against constant-product mechanics
- **Fee Calculation**: Multi-tier fee splits validated for all fee rate combinations

#### State Transitions
- **Tier Activation Logic**: Auto-activation triggers properly when total LP crosses threshold
- **Tier Deactivation Logic**: Auto-deactivation and sweep occur correctly on final burn
- **Tier Mask Updates**: Bitmask state changes validated across thousands of operation sequences
- **Reserve Updates**: Balance changes match expected deltas for all operation types

#### Edge Case Exploration
- **Near-Zero Reserves**: Protocol behavior with minimal liquidity levels
- **Large Numbers**: Operations with reserves approaching 64-bit limits
- **Precision Boundaries**: Rounding behavior at precision limits for floor division
- **Sweep Conditions**: Correct handling when burning all LP in a tier

### Fuzz Test Results

Property-based tests **passed across hundreds of thousands of randomized examples**, confirming:
- ✅ K-invariant holds for all generated swap sequences
- ✅ Fee accounting balances to zero error across complex operation chains
- ✅ Wide arithmetic functions match reference implementations
- ✅ No overflow conditions in supported reserve ranges (up to ~2^61 for sqrt operations)
- ✅ Tier state transitions occur correctly under all tested scenarios
- ✅ Reserve balances remain consistent through randomized operation sequences

---

## Testing Infrastructure

### Test Execution
- **Environment**: AlgoKit LocalNet (sandboxed Algorand network for testing)
- **Framework**: pytest with custom plugins for AMM-specific assertions
- **Logging**: Detailed operation traces, balance checkpoints, and invariant tracking
- **Reproducibility**: Seeded random operations for deterministic test replay

### Coverage
The test suite exercises:
- All 6 fee tiers (0.03% to 3%, plus ~0.0001% protocol tier)
- All operation types (swap, swap_limit, mint, burn, mint_single, burn_single, mint_hybrid)
- Both asset pair types (ASA/ASA and ALGO/ASA)
- Multi-user concurrent scenarios
- Error conditions and input validation
- Admin and governor operations
- Full lifecycle: deploy → mint → swap → burn → sweep

### Validation Methodology
- **Unit-level**: Individual math functions tested against reference implementations
- **Integration-level**: Full transaction flows on live test network
- **Property-based**: Randomized sequences validated against invariants
- **Regression**: ABI compatibility checks ensure no breaking changes

---

## Conclusion

STAMM's comprehensive test suite provides strong confidence in protocol correctness, safety, and reliability. The combination of integration testing (real-world scenarios) and fuzz testing (mathematical properties) ensures the protocol behaves correctly under both typical usage patterns and edge-case conditions.

All tests pass successfully, validating that STAMM operates as designed across its full feature set.
