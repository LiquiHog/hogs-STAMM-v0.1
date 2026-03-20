# Governance

STAMM separates deployment from ongoing operations. The factory deploys pools, the admin contract governs and withdraws treasury claims, and the registry provides pair/LP discovery.

---

## Governor Model

After `register_pool_lps`, each pool transfers governor authority from the factory to the admin contract.

- Pool governor methods (`withdraw_lp`, `withdraw_assets`, `transfer_governor`) are called through the admin contract.
- The factory cannot call governor-only pool methods after handoff.
- Governor migration is explicit and timelocked in the admin contract.

## Timelocks

### Factory - 7 Days (604,800 seconds)

| Action | Propose | Confirm | Cancel |
|---|---|---|---|
| Admin transfer | `propose_admin` | `accept_admin` | `cancel_admin_proposal` |
| Admin contract change | `propose_admin_contract` | `confirm_admin_contract` | `cancel_admin_contract_proposal` |
| Registry change | `propose_registry` | `confirm_registry` | `cancel_registry_proposal` |
| Unfreeze | `propose_unfreeze` | `confirm_unfreeze` | `cancel_unfreeze` |

### Admin Contract - 7 Days (604,800 seconds)

| Action | Propose | Confirm | Cancel |
|---|---|---|---|
| Admin transfer | `propose_admin` | `accept_admin` | `cancel_admin_proposal` |
| Treasury operator change | `propose_treasury` | `confirm_treasury` | `cancel_treasury` |
| Governor migration | `propose_migration` | `confirm_pool_migration` | `cancel_migration` |

`guardian` can cancel queued admin/treasury/migration proposals. `set_guardian` is admin-only and not timelocked.

### Registry - 72 Hours (259,200 seconds)

| Action | Propose | Confirm | Cancel |
|---|---|---|---|
| Admin transfer | `propose_admin` | `accept_admin` | `cancel_admin_proposal` |

## Freeze and Update Controls

### Factory

- `freeze_factory()` is instant and blocks template management (`init_template`, `upload_template`) until unfreeze.
- `propose_unfreeze -> confirm_unfreeze` is 7-day timelocked.
- Update/delete are permanently blocked by `on_delete_or_update -> op.err()`.

### Admin Contract

- No freeze mechanism.
- Update/delete are permanently blocked by `on_delete_or_update -> op.err()`.

### Registry

- No freeze mechanism.
- Update/delete remain admin-gated via `on_delete_or_update`.

## Governor Migration Flow

1. Admin proposes a new governor with `propose_migration`.
2. After 7 days, admin calls `confirm_pool_migration(pool_app_id)` per pool.
3. Each call forwards `transfer_governor(new_governor)` to that pool.

Migration is per-pool, so governance can migrate pools in batches.

## Creation Mode

Factory controls creation access through `creation_mode`:

| Mode | Value | Access |
|---|---|---|
| Paused | 0 | No new pools |
| Admin-only | 1 | Factory admin only |
| Permissionless | 2 | Anyone |
