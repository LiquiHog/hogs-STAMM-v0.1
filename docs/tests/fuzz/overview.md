# Fuzz Testing Overview

Property-based fuzz testing uses randomized operation sequences to validate mathematical invariants and safety properties that must hold under all conditions.

---

## Framework

- **Tool**: Hypothesis (Python property-based testing library)
- **Strategy**: Generate random sequences of operations with varying parameters
- **Validation**: Assert mathematical properties after each operation
- **Coverage**: Hundreds of thousands of examples per test function
- **Reproducibility**: Seed-based random generation for deterministic replay of failures

---

## Test Methodology

### Example Generation

Hypothesis generates random test cases including:

- **Operation types**: swap, mint, burn, mint_single, mint_hybrid, burn_single
- **Amounts**: Random values from small (1 microunit) to large (millions)
- **Tiers**: Random tier selection (0-7)
- **Asset directions**: A→B or B→A for swaps
- **User roles**: Multiple simulated users with independent balances

### Property Validation

After each operation, validators check:

1. **Mathematical invariants** — Properties that must always hold (e.g., k-invariant preservation)
2. **State consistency** — Contract state matches expected values from simulation
3. **Balance conservation** — Total supply = user balances + pool holdings
4. **Monotonic properties** — Values that must only increase (e.g., cumulative fees)

### Failure Handling

When Hypothesis finds a failing case:
1. **Shrinking**: Automatically reduces the input to a minimal failing example
2. **Logging**: Records full operation sequence leading to failure
3. **Replay**: Failed case can be replayed with same seed for debugging
4. **Fix & Retest**: After fix, Hypothesis generates new examples to find other issues

---

## Coverage Targets

### Operation Sequences

- **Length**: 1 to 1000 operations per sequence
- **Variety**: All 7 core operation types interleaved randomly
- **Users**: 1 to 10 concurrent users
- **Tiers**: Random tier selection with varying activation states

### Parameter Ranges

- **Amounts**: 1 microunit to 10^12 (covers dust to large trades)
- **Reserves**: Bootstrap state (1:1) to deep liquidity (millions)
- **LP Supply**: Seed state (1 LP) to large pools (billions of LP)
- **Fee Tiers**: All 7 tiers with fee rates from 3 bps to 500 bps

### Edge Cases Explored

- **Boundary values**: 0, 1, max uint64, max-1
- **Precision limits**: Operations that test rounding and floor division
- **State transitions**: Bootstrap → active → inactive → bootstrap again
- **Overflow risks**: Large multiplications that require wide math

---

## Test Categories

Fuzz tests are organized by property type:

1. **[Invariants](invariants.md)** — Mathematical properties that must always hold
   - K-invariant preservation
   - Fee accounting balance
   - Reserve consistency
   - LP supply conservation

2. **[Edge Cases](edge-cases.md)** — Boundary conditions and corner cases
   - Near-zero reserves
   - Large number handling
   - Precision boundaries
   - Overflow prevention

---

## Test Execution

### Running Fuzz Tests

```bash
pytest tests/test_fuzz.py -v --hypothesis-show-statistics
```

### Statistics Reported

- **Examples tried**: Number of random cases generated
- **Shrink passes**: Number of shrinking attempts on failures
- **Runtimes**: Time spent generating and validating examples
- **Coverage**: Distribution of operation types and parameter ranges tested

### Continuous Fuzzing

Fuzz tests run on every commit in CI/CD:
- **Quick run**: 100 examples per test (fast feedback)
- **Extended run**: 10,000+ examples (nightly builds)
- **Regression suite**: Previously-failed cases replayed to prevent regressions

---

## Benefits of Fuzz Testing

- **Comprehensive coverage**: Tests scenarios humans wouldn't manually construct
- **Mathematical rigor**: Validates invariants across entire input space
- **Regression prevention**: Automatically catches bugs introduced by changes
- **Edge case discovery**: Finds boundary conditions and corner cases
- **Confidence building**: Hundreds of thousands of passing examples provide strong assurance

---

## Test Results Summary

All fuzz tests **passed across hundreds of thousands of randomized examples**, confirming:
- ✅ All mathematical invariants hold for generated operation sequences
- ✅ No overflow conditions in supported reserve ranges
- ✅ State transitions occur correctly under random operations
- ✅ Reserve balances remain consistent through complex operation chains
- ✅ Fee accounting balances to zero error
- ✅ Math functions match reference implementations exactly

---

## Related Documentation

- [Invariants](invariants.md) — Detailed property validation tests
- [Edge Cases](edge-cases.md) — Boundary condition and corner case tests
- [Integration Tests](../integration/overview.md) — End-to-end scenario validation
