# Admin Contract

The AdminContract is the permanent governor of all registered STAMM pools. It provides treasury withdrawal proxy methods, admin transfer, freeze, and governor migration — all protected by 72-hour timelocks.

---

## Role

After pool creation and LP registration, the factory transfers governor authority to the admin contract. From that point forward, the admin contract is the only account that can call governor-protected methods on pools (treasury withdrawals, governor transfer). This separation means the factory handles deployment while the admin contract handles ongoing governance.

## State

| Key | Type | Description |
|---|---|---|
| `admin` | bytes | Protocol admin address |
| `pending_admin` | bytes | Proposed new admin (zero until `propose_admin` is called) |
| `propose_ts` | uint64 | Admin transfer proposal timestamp for 72-hour timelock |
| `frozen` | uint64 | Irreversible freeze flag: blocks update/delete |
| `freeze_ts` | uint64 | Freeze proposal timestamp for 72-hour timelock |
| `pending_governor` | bytes | Proposed new governor address for pool migration |
| `governor_propose_ts` | uint64 | Migration proposal timestamp for 72-hour timelock |

## ABI Methods

### Admin Transfer

| Method | Description |
|---|---|
| `propose_admin(address)void` | Propose a new admin (starts 72-hour timelock) |
| `accept_admin()void` | Accept admin role after timelock (pending admin only) |
| `cancel_admin_proposal()void` | Cancel a pending admin proposal |

### Freeze

| Method | Description |
|---|---|
| `propose_freeze()void` | Propose freezing the admin contract (starts 72-hour timelock) |
| `confirm_freeze()void` | Confirm and execute irreversible freeze after timelock |
| `cancel_freeze()void` | Cancel a pending freeze proposal |

Once frozen, the admin contract permanently blocks update and delete operations. Treasury withdrawals, admin transfer, and governor migration remain functional.

### Governor Migration

| Method | Description |
|---|---|
| `propose_migration(address)void` | Propose a new governor address for pool migration (starts 72-hour timelock) |
| `confirm_pool_migration(application)void` | Transfer a pool's governor to the proposed address (per-pool, after timelock) |
| `cancel_migration()void` | Cancel a pending migration proposal |

Migration works even when the admin contract is frozen, ensuring pools are never permanently locked.

### Treasury Withdrawals

| Method | Description |
|---|---|
| `withdraw_pool_lp(application,uint64,uint64,address,uint64)void` | Withdraw treasury LP tokens from a pool tier |
| `withdraw_pool_assets(application,uint64,uint64,uint64,uint64,address)void` | Withdraw treasury raw token claims from a pool |

These proxy methods call governor-protected `withdraw_lp` and `withdraw_assets` on the target pool via inner transactions.

## Security Properties

- **72-hour timelocks** on all governance actions (admin transfer, freeze, migration)
- **Irreversible freeze** — once confirmed, cannot be undone; blocks code changes but not operations
- **No direct pool keys** — pools don't have individual admin keys; all governance flows through the admin contract
- **Survives factory replacement** — the admin contract retains governor authority independent of the factory
- **Treasury pull model** — claims are stored in pool state and withdrawn on demand; no opt-in coordination required
