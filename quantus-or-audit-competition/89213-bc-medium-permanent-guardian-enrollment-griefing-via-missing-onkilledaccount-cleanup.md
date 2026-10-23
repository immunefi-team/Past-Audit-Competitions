# 89213 bc medium permanent guardian enrollment griefing via missing onkilledaccount cleanup

**Submitted on Aug 17th 2026 at 21:11:28 UTC by @craftsoldier for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89213
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary

The guardian system `pallet_reversible_transfers` implements a high-security (HS) mode where users can enroll a trusted account a.k.a. "guardian" that can cancel their fraudulent transfers and recover funds on their behalf. A user calls `set_high_security(delay, guardian)` to enroll, and this is a one-way operation where there is no `unset_high_security` extrinsic, by design (line 310 of `pallets/reversible-transfers/src/lib.rs`: "Enabling high security mode is a one-way operation that cannot be undone.").

Each guardian's wards are tracked in `GuardianIndex`:

```rust
// pallets/reversible-transfers/src/lib.rs
#[pallet::storage]
pub type GuardianIndex<T: Config> = StorageMap<
    _,
    Blake2_128Concat,
    T::AccountId,
    BoundedVec<T::AccountId, T::MaxGuardianAccounts>,  // capped at 32
    ValueQuery,
>;
```

`MaxGuardianAccounts = 32` is set in `runtime/src/configs/mod.rs`.

### The bug: GuardianIndex is append-only with no eviction path

`set_high_security` is the only extrinsic that writes to `GuardianIndex`. It pushes the caller into the guardian's BoundedVec:

```rust
// pallets/reversible-transfers/src/lib.rs — set_high_security
GuardianIndex::<T>::try_mutate(guardian.clone(), |accounts| {
    if !accounts.contains(&who) {
        accounts.try_push(who.clone()).map_err(|_| Error::<T>::TooManyGuardianAccounts)
    } else {
        Ok(())
    }
})?;
```

When the BoundedVec is full (32/32), `try_push` fails → `Error::TooManyGuardianAccounts`. There is no code path in the entire pallet that removes an entry from `GuardianIndex`. No `retain`, no `remove`, no `swap_remove`, no `clear`. The BoundedVec is purely append-only.

### No OnKilledAccount hook

When `frame_system` reaps an account (balance → 0, providers → 0), it calls `T::OnKilledAccount::on_killed_account(who)`:

```rust
// pallets/frame-system/src/lib.rs — dec_providers
(1, 0, 0) => {
    Pallet::<T>::on_killed_account(who.clone());  // fires the hook
    Ok(DecRefStatus::Reaped)
}
```

The runtime config wires only `Pubkey` (the Dilithium pubkey cache pallet) for this callback:

```rust
// runtime/src/configs/mod.rs:172
type OnKilledAccount = Pubkey;
```

`pallet_reversible_transfers` does not implement `OnKilledAccount` and is not in the handler chain. Grepping for `OnKilledAccount` in `pallets/reversible-transfers/src/lib.rs` returns nothing. So when an HS account is reaped:

* Its `HighSecurityAccounts` entry persists (orphaned — points to a dead account)
* Its entry in the guardian's `GuardianIndex` BoundedVec persists (dead account permanently occupying a slot)

### recover\_funds does not clean up either

The guardian's only defense is `recover_funds`, which drains the HS account's balance to the guardian:

```rust
// pallets/reversible-transfers/src/lib.rs — recover_funds
let call: RuntimeCallOf<T> = pallet_balances::Call::<T>::transfer_all {
    dest: T::Lookup::unlookup(who.clone()),
    keep_alive: false,  // drains to 0 → triggers reap
}
.into();
call.dispatch(frame_system::RawOrigin::Signed(account.clone()).into());
```

This reaps the throwaway, but `recover_funds` does not remove the account from `GuardianIndex` or `HighSecurityAccounts`. The docs explicitly state (line 503): "The high-security status and guardian relationship are intentionally preserved after recovery." So after reaping, the dead entry persists in `GuardianIndex`.

### set\_high\_security is permissionless

`set_high_security` is a normal signed extrinsic. Any account can call it, naming any other account as their guardian. No consent, approval, or minimum balance is required from the guardian. The guardian does not need to be online, funded, or aware. An attacker only needs to know the guardian's public address.

### The attack

{% stepper %}
{% step %}

## A legitimate ward enrolls

A legitimate ward enrolls with a victim guardian via `set_high_security`. `GuardianIndex = 1/32`.
{% endstep %}

{% step %}

## Fill the guardian capacity

Attacker creates 31 throwaway accounts and funds each with \~1 UNIT (1 UNIT = 10¹² plancks; existential deposit = 0.001 UNIT).

Each throwaway calls `set_high_security(delay=2, guardian=victim)`. `GuardianIndex` fills to 32/32.
{% endstep %}

{% step %}

## Block innocent enrollment

An innocent user tries to enroll → `TooManyGuardianAccounts`. Blocked.
{% endstep %}

{% step %}

## Reap the throwaways

Guardian calls `recover_funds` on all 31 throwaways → all drained to 0 and reaped.
{% endstep %}

{% step %}

## Dead entries persist

All 31 throwaways are dead, but `GuardianIndex` is still 32/32 (dead entries persist - no `OnKilledAccount` cleanup). `HighSecurityAccounts` entries also persist (orphaned).
{% endstep %}

{% step %}

## Enrollment remains blocked

Innocent user tries again → still `TooManyGuardianAccounts`. Permanently blocked.
{% endstep %}

{% step %}

## Existing ward remains protected

The legitimate ward from step 1 is still HS and still protected - only new enrollment is blocked.
{% endstep %}
{% endstepper %}

### Impact: Griefing

* **Attacker cost**: In the PoC as written, Alice spends \~35 UNIT (31 throwaways × 1 UNIT + ward + innocent + guardian funding). The attacker can reduce this to \~4 UNIT + fees by having each throwaway drain its balance back via `schedule_transfer` (whitelisted for HS accounts, uses `transfer_keep_alive` so the throwaway keeps \~ED) before the guardian reaps them. The guardian then only recovers \~31 × ED ≈ 0.031 UNIT via `recover_funds`.
* **Attacker gain**: nothing.
* **Guardian loss**: permanently loses the ability to accept new wards. Their 32-slot capacity is filled with dead accounts that can never be removed.
* **User loss**: any user who wants to enroll with this guardian is permanently denied HS protection.
* **Permanence**: no extrinsic, hook, or mechanism can remove dead entries from `GuardianIndex`. The only remediation is a runtime upgrade that (a) adds `OnKilledAccount` to `pallet_reversible_transfers`, (b) wires it into `type OnKilledAccount = (Pubkey, ReversibleTransfers);`, and (c) runs a storage migration to purge existing dead entries.

## Link to Proof of Concept

<https://gist.github.com/craftsoldier/3a1770297febb4911791fe3d1bc8f19e>

## Proof of Concept

All testing was performed on a local `--dev` node. No mainnet or public testnet was involved.

### Setup

Build the node from stock `audit-comp-ready with no modifications:` cargo build --release -p quantus-node`. Start it in dev mode: eg:` ./target/release/quantus-node --dev --base-path /tmp/poc --rpc-port 9944\`.

The dev genesis pre-funds Alice with enough balance to bankroll the entire attack. The protocol caps each guardian at 32 wards (`MaxGuardianAccounts = 32`), and the existential deposit is 0.001 UNIT.

The PoC creates five roles: Alice (the attacker, genesis-funded), a victim guardian (a regular account that offers guardian protection), a legitimate ward (a real user who enrolls with the guardian before the attack), 31 throwaway accounts (attacker-controlled, used to fill the guardian's slots), and an innocent user (a real user who tries to enroll after the attack). The PoC connects to the node over RPC, submits real signed transactions on behalf of each account, and queries chain storage to verify the result at each step.

{% stepper %} {% step %}

## Guardian is working

Alice funds the legitimate ward with 1 UNIT via `Balances::transfer_keep_alive`. The ward calls `set_high_security(delay=2, guardian=victim)`. The code path: `ensure_signed` → `ensure!(guardian != who)` → `ensure!(!HighSecurityAccounts::contains_key(&who))` → `validate_delay` → `GuardianIndex::try_mutate(victim, |accounts| accounts.try_push(ward))` → `HighSecurityAccounts::insert(ward, { guardian: victim, delay: 2 })`. `GuardianIndex[victim] = [ward]`, length 1. The guardian is open for business. {% endstep %}

{% step %}

## Fund throwaways

Alice sends 1 UNIT to each of 31 throwaway accounts and 1 UNIT to the innocent user, all via `Balances::transfer_keep_alive`. Alice tracks nonces locally and submits all 32 funding txs in rapid succession. The PoC polls the block number and waits for 3 blocks to include them all. {% endstep %}

{% step %}

## Fill the GuardianIndex

Each throwaway calls `set_high_security(delay=2, guardian=victim)`. Same code path as step 0 `GuardianIndex::try_mutate(victim, |accounts| accounts.try_push(throwaway[i]))`. The `try_push` succeeds 31 times. After the last call, `GuardianIndex[victim]` has 32 entries (1 legitimate ward + 31 throwaways), which is the `MaxGuardianAccounts` cap. The BoundedVec is full. {% endstep %}

{% step %}

## Innocent user blocked

The innocent user calls `set_high_security(delay=2, guardian=victim)`. The tx is a normal signed extrinsic, so it validates and enters the pool. At dispatch, `GuardianIndex::try_mutate(victim, |accounts| accounts.try_push(innocent))` calls `try_push` on the full BoundedVec, which fails → `Error::TooManyGuardianAccounts`. The dispatch fails, the nonce is consumed, and `HighSecurityAccounts[innocent]` does not exist. The user is blocked. {% endstep %}

{% step %}

## Guardian attempts defense

Alice funds the guardian with 2 UNIT for `recover_funds` fees. The guardian calls `recover_funds(account=throwaway[i])` for each of the 31 throwaways. The code path: `ensure_signed(guardian)` → `HighSecurityAccounts::get(throwaway[i])` → `ensure!(guardian == hs_data.guardian)` → pending\_tx\_ids is empty (no scheduled transfers) → `pallet_balances::Call::transfer_all { dest: guardian, keep_alive: false }` dispatched from `RawOrigin::Signed(throwaway[i])`.

This drains the throwaway's entire free balance to the guardian. The throwaway's balance drops to 0, below ED (0.001 UNIT). `pallet_balances` calls `frame_system::dec_providers(throwaway[i])`. The account has providers=1, consumers=0, sufficients=0 → `(1, 0, 0)` → `Pallet::<T>::on_killed_account(throwaway[i])` fires → `T::OnKilledAccount::on_killed_account(throwaway[i])` → only `Pubkey::on_killed_account` runs (removes cached Dilithium pubkey if present). `pallet_reversible_transfers` is not in the handler chain `type OnKilledAccount = Pubkey;` so `GuardianIndex` and `HighSecurityAccounts` are untouched. The guardian tracks nonces locally and submits all 31 `recover_funds` calls in rapid succession. {% endstep %}

{% step %}

## Dead entries persist

The PoC queries `frame_system::Account` storage for each throwaway — all 31 return null. The accounts are dead. It queries `GuardianIndex` for the guardian — length is still 32. The 31 dead throwaways are still in the BoundedVec. It queries `HighSecurityAccounts` for throwaway 0 — the entry still exists, pointing to a dead account. Orphaned state that no extrinsic or hook can clean up. {% endstep %}

{% step %}

## Permanence confirmed

The innocent user calls `set_high_security` again — same code path, same `try_push` failure, same `TooManyGuardianAccounts`. Still blocked. The PoC queries `HighSecurityAccounts` for the legitimate ward — still exists, still HS. The legitimate ward from step 0 is unaffected. The guardian is permanently at 32/32: 31 dead accounts that can never be removed, plus 1 live legitimate ward. No `unset_high_security` exists. No extrinsic removes entries from `GuardianIndex`. No `OnKilledAccount` hook fires for `pallet_reversible_transfers`. The only remediation is a runtime upgrade. {% endstep %} {% endstepper %}

### Observed result

```
GuardianIndex: 32/32 (31 dead + 1 alive legitimate ward)
Dead throwaways: 31/31 (frame_system::Account = null for all)
Innocent user: blocked (HighSecurityAccounts does not exist)
Legitimate ward: still HS (HighSecurityAccounts exists)
Guardian recourse: none
Runtime: ~2 minutes with block polling
```

### Recommended fix

A proper fix requires three changes:

{% stepper %}
{% step %}

## Implement `OnKilledAccount` in `pallet_reversible_transfers`

When an account is reaped, look up `HighSecurityAccounts::get(who)`. If it exists, read the guardian and remove `who` from `GuardianIndex[guardian]` (e.g. `accounts.retain(|a| a != who)`), then delete `HighSecurityAccounts::remove(who)`.
{% endstep %}

{% step %}

## Wire the hook into the runtime

Change `type OnKilledAccount = Pubkey;` to `type OnKilledAccount = (Pubkey, ReversibleTransfers);`.
{% endstep %}

{% step %}

## Run a storage migration

Purge existing dead entries from both `GuardianIndex` and `HighSecurityAccounts`.
{% endstep %}
{% endstepper %}

With this fix, Step 5 would instead show `GuardianIndex[victim]` dropping back to 1 (only the legitimate ward), and Step 6 would show the innocent user successfully enrolling.
