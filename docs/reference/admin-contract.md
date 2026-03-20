# Admin Contract

The AdminContract is the permanent governor of registered STAMM pools. It separates governance from treasury execution by introducing dedicated `admin`, `treasury`, and `guardian` roles.

---

## Role

After `register_pool_lps`, the factory transfers each pool's governor to the admin contract. From that point onward:

- Pool governor methods are invoked through the admin contract.
- The factory no longer controls pool-governor operations.
- Treasury withdrawals are performed by `treasury` (not `admin`).

## Roles

| Role | Permissions |
|---|---|
| `admin` | Propose/confirm admin, treasury, and governor migration changes; set guardian |
| `treasury` | Call `withdraw_pool_lp` and `withdraw_pool_assets` |
| `guardian` | Cancel queued admin, treasury, and migration proposals |

## Timelocks

All admin-contract governance proposals use a 7-day timelock (604,800 seconds):

- `propose_admin` -> `accept_admin`
- `propose_treasury` -> `confirm_treasury`
- `propose_migration` -> `confirm_pool_migration`

## State

| Key | Type | Description |
|---|---|---|
| `admin` | bytes | Governance admin address |
| `treasury` | bytes | Treasury operator address |
| `guardian` | bytes | Guardian address (veto/cancel role) |
| `pending_treasury` | bytes | Proposed treasury address |
| `treasury_propose_ts` | uint64 | Treasury proposal timestamp |
| `pending_admin` | bytes | Proposed admin address |
| `propose_ts` | uint64 | Admin proposal timestamp |
| `pending_governor` | bytes | Proposed new governor address |
| `governor_propose_ts` | uint64 | Governor-migration proposal timestamp |

## ABI Methods

### Admin Transfer

| Method | Description |
|---|---|
| `propose_admin(address)void` | Propose a new admin (7-day timelock) |
| `accept_admin()void` | Pending admin accepts after timelock |
| `cancel_admin_proposal()void` | Cancel pending admin proposal (admin or guardian) |

### Treasury Management

| Method | Description |
|---|---|
| `propose_treasury(address)void` | Propose a new treasury operator (7-day timelock) |
| `confirm_treasury()void` | Confirm treasury change after timelock |
| `cancel_treasury()void` | Cancel pending treasury proposal (admin or guardian) |
| `set_guardian(address)void` | Set or clear guardian (admin only, immediate) |

### Governor Migration

| Method | Description |
|---|---|
| `propose_migration(address)void` | Propose new governor address (7-day timelock) |
| `confirm_pool_migration(application)void` | Migrate one pool to pending governor |
| `cancel_migration()void` | Cancel pending migration proposal (admin or guardian) |

### Treasury Withdrawals

| Method | Description |
|---|---|
| `withdraw_pool_lp(application,uint64,uint64,address,uint64)void` | Withdraw treasury LP claims from a pool tier (treasury only) |
| `withdraw_pool_assets(application,uint64,uint64,uint64,uint64,address)void` | Withdraw treasury asset claims from a pool (treasury only) |

## Security Properties

- Governance and treasury execution are separated (`admin` vs `treasury`).
- Guardian can veto queued governance transitions by cancellation.
- Delete/update are permanently blocked by `on_delete_or_update -> op.err()`.
- Pool control remains centralized at the contract address, not individual pool keys.
