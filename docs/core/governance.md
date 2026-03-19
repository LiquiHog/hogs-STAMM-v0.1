# Governance

STAMM separates deployment from ongoing governance. The factory deploys pools, the admin contract governs them, and the registry provides permanent discovery. Each contract has independent freeze and admin transfer mechanisms with timelocks calibrated to its role.

---

## Governor Model

After pool creation and LP registration, the factory transfers governor authority to the admin contract. The admin contract becomes the permanent governor of all pools:

- Pool governance operations (treasury withdrawals, governor transfer) require going through the admin contract
- Compromising a single account doesn't affect individual pools — pools don't have individual admin keys
- The factory cannot call governor-protected methods after handoff
- Even if the factory is deleted or replaced, the admin contract retains governor authority

## Timelocks

### Factory — 7-Day Timelocks (604,800 seconds)

All factory governance actions require a propose → confirm cycle with a 7-day waiting period:

| Action | Propose | Confirm | Cancel |
|---|---|---|---|
| Admin transfer | `propose_admin` | `accept_admin` | `cancel_admin_proposal` |
| Admin contract change | `propose_admin_contract` | `confirm_admin_contract` | `cancel_admin_contract_proposal` |
| Registry change | `propose_registry` | `confirm_registry` | `cancel_registry_proposal` |
| Unfreeze | `propose_unfreeze` | `confirm_unfreeze` | `cancel_unfreeze` |

### Admin Contract — 72-Hour Timelocks (259,200 seconds)

The shorter timelock reflects the higher trust level of the admin contract (it is already the governor of all pools):

| Action | Propose | Confirm | Cancel |
|---|---|---|---|
| Admin transfer | `propose_admin` | `accept_admin` | `cancel_admin_proposal` |
| Freeze | `propose_freeze` | `confirm_freeze` | `cancel_freeze` |
| Governor migration | `propose_migration` | `confirm_pool_migration` | `cancel_migration` |

## Freeze Systems

Three independent freeze mechanisms protect different layers of the protocol.

### Factory Freeze

- **Freeze**: Instant — `freeze_factory()` blocks template uploads, contract updates, and deletions immediately
- **Unfreeze**: 7-day timelock — `propose_unfreeze` → `confirm_unfreeze`
- **Scope**: Pool creation and LP registration remain functional during a freeze

### Admin Contract Freeze

- **Freeze**: 72-hour timelock — `propose_freeze` → `confirm_freeze`
- **Irreversible**: Once confirmed, the freeze cannot be undone
- **Scope**: Blocks update and delete operations. Treasury withdrawals, admin transfer, and governor migration remain functional
- Governor migration works when frozen, ensuring pools are never permanently locked

### Registry Freeze

- **Freeze**: Instant — `freeze_registry()` permanently blocks all write operations
- **Irreversible**: Cannot be undone
- **Scope**: Read methods (`get_pool`, `get_lp_info`) remain fully functional. Admin transfer still works, preventing the contract from being orphaned

## Admin Transfer

Each contract uses a two-step process for admin transfer:

1. Current admin proposes a new admin address (starts the timelock)
2. New admin accepts after the timelock expires

The current admin can cancel the proposal at any time before confirmation. For the factory, `accept_admin` is called by the pending admin. For the admin contract, the same pattern applies with a shorter timelock.

## Governor Migration

The admin contract can transfer individual pools to a new governor:

1. Admin proposes a new governor address via `propose_migration` (starts 72-hour timelock)
2. After the timelock, admin calls `confirm_pool_migration(pool_app_id)` per pool
3. Each confirmation calls `transfer_governor` on the target pool

Migration is per-pool, allowing selective or batched transfers. It works even when the admin contract is frozen.

## Creation Mode

The factory controls who can create pools via `creation_mode`:

| Mode | Value | Access |
|---|---|---|
| Paused | 0 | No new pools |
| Admin-only | 1 | Factory admin only |
| Permissionless | 2 | Anyone |

The admin can change the mode at any time via `set_creation_mode`.
