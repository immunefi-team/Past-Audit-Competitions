# 88215 bc medium any account can permanently and irrevocably consume another address s bounded guardian index with no consent and no removal path

**Submitted on Aug 12th 2026 at 09:30:08 UTC by @Valisthea for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88215
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

`set_high_security` writes the **caller** into the **guardian's** `GuardianIndex`, a `BoundedVec` capped at `MaxGuardianAccounts = 32`, without any consent from, signature by, or even on-chain existence of the account being written into. The pallet contains no path that ever removes an entry, and `set_high_security` is documented as irreversible. Any attacker can therefore spend 32 ordinary transactions to permanently exhaust the guardian index of any chosen address, after which no honest user can ever name that address as their guardian. On mainnet this permanently denies the pallet's core protective feature to any published guardian address (a custody provider, an institutional cold-wallet guardian, a well-known community address), and the address can be pre-squatted before its owner ever transacts. There is no in-protocol remedy: not for the victim, not for the occupiers, not for the guardian itself.

## Vulnerability Details

### The write

`set_high_security` pushes the caller into the guardian's index:

```rust
// pallets/reversible-transfers/src/lib.rs:394-396
GuardianIndex::<T>::try_mutate(guardian.clone(), |accounts| {
    if !accounts.contains(&who) {
        accounts.try_push(who.clone()).map_err(|_| Error::<T>::TooManyGuardianAccounts)
```

into a bounded container:

```rust
// pallets/reversible-transfers/src/lib.rs:248-253
pub type GuardianIndex<T: Config> = StorageMap<
    _, Blake2_128Concat, T::AccountId,
    BoundedVec<T::AccountId, T::MaxGuardianAccounts>, ValueQuery,
>;
```

```rust
// runtime/src/configs/mod.rs:528
pub const MaxGuardianAccounts: u32 = 32;
```

### Why this is a defect and not the documented bound

**The pallet already ships a test asserting the bound holds** (`tests/test_reversible_transfers.rs:1630`, `guardian_index_respects_max_limit`): fill the index, and the next `set_high_security` correctly fails with `TooManyGuardianAccounts`. **That test is correct, and this report does not dispute it.** The bound working as designed is the premise here, not the finding.

That test establishes *that* the bound holds. It never asks the two questions that turn the bound into a griefing primitive:

**Q1. Who is permitted to consume the bound?** Anyone, against any address. The `guardian` argument to `set_high_security` is an arbitrary `T::AccountId`. It never signs, is never consulted, and need not exist on chain. The only guards on the call are `guardian != who`, `!HighSecurityAccounts::contains_key(&who)`, and `validate_delay` (`lib.rs:383-390`). There is **no deposit**, no reserve, and no balance requirement attached to the index entry.

**Q2. Can a consumed slot ever be released?** No. `GuardianIndex::<T>` has exactly **two write sites in the entire pallet**, and both are push-only:

* `lib.rs:394` — the signed `try_push` shown above
* `lib.rs:992` — the genesis builder

No extrinsic, hook, or helper ever removes an entry. This is confirmed dynamically in the PoC by exercising all six dispatchables (call\_index 0 through 5) and observing the index is unchanged.

Permanence is then locked in by design on the other side as well. `lib.rs:347` documents: *"Enabling high security mode is a one-way operation that cannot be undone."* An enrolled account cannot un-enroll, so its index entry can never lapse.

### The composition

Four properties compose into the issue, and no single one of them is a bug on its own:

1. The written-into account gives **no consent** and need not exist.
2. The entry carries **no deposit**, so occupying a slot is free beyond the transaction fee.
3. There is **no removal path** anywhere in the pallet.
4. Enrollment is **irreversible by design**, so entries never expire.

Together these make "accounts protected by guardian G" a **scarce, per-address, globally-consumable resource** that any unprivileged party can permanently exhaust for the cost of 32 transactions, with no reclamation and no consent from G.

### Why the obvious mitigations do not work

1. **Raising `MaxGuardianAccounts` does not repair it.** It is a compile-time `pub const` inside `parameter_types!` (`runtime/src/configs/mod.rs:528`), not a storage-backed parameter, so changing it requires a full runtime upgrade. More importantly, squatting simply refills any newly granted capacity at the same trivial cost. The bound is not a defence; it is the resource being consumed.
2. **Purging existing squatters requires a storage migration**, not a parameter change, because the pallet has no eviction path at all.
3. **Governance is narrower than it appears.** `runtime/src/lib.rs:253-259` leaves pallet index 10 (`Referenda`) and index 12 (`ConvictionVoting`) vacant, commented *"removed with the public/token-weighted governance lane"*. Only `TechCollective` and `TechReferenda` remain, so remediation is a permissioned technical runtime upgrade with a migration, available to neither the victim nor ordinary token holders.

## Impact Details

**Selected impact: Griefing (no profit motive for an attacker, but damage to the users or the protocol).** Every element of that definition is satisfied literally:

* **No profit motive.** The attacker gains nothing. The squatting accounts hold only their own balances, and those are in fact *recoverable by the victim*, since the victim is their guardian and `recover_funds` succeeds against every one of them (asserted in the PoC). The attack partially self-funds its own target.
* **Damage to the users.** A chosen address is permanently denied the ability to serve as a guardian. Every honest user who tries to name it receives `TooManyGuardianAccounts` and is left with `is_high_security() == None`, that is, no protection at all. The victim has no way to clear the index and no way to refuse the entries in the first place.
* **Damage to the protocol.** The pallet's core protective feature is permanently degraded for the targeted address, and the degradation is unbounded in time and irreparable without a runtime upgrade plus a storage migration.

The addresses this targets are precisely the ones that would be published and reused: a custody or guardianship-as-a-service provider, an institutional cold-wallet guardian, a well-known community address. Because the guardian need not exist on chain, **an address can be pre-squatted before its owner ever transacts**, so a service can be denied its own published identity from day zero.

Scale: one target address costs the attacker 32 transactions. Denying N published guardian addresses costs 32N transactions and is trivially parallelizable.

Honest bounding, stated rather than omitted: no funds are lost or frozen by this attack, and a user can choose a different guardian address, so the denial is per-address rather than global. Rotating a published service address is operationally costly rather than impossible. What resists downgrade to Insight is that the victim has **no in-protocol remedy whatsoever** and the natural mitigation is defeated at negligible cost.

## References

* Write site: `pallets/reversible-transfers/src/lib.rs:394-396`
* Storage definition: `pallets/reversible-transfers/src/lib.rs:248-253`
* Production bound: `runtime/src/configs/mod.rs:528`
* Irreversibility documented: `pallets/reversible-transfers/src/lib.rs:347`
* Genesis write site (the only other one): `pallets/reversible-transfers/src/lib.rs:992`
* Governance lanes removed: `runtime/src/lib.rs:253-259`
* The project's own bound test, which this report does not dispute: `pallets/reversible-transfers/src/tests/test_reversible_transfers.rs:1630`

## Link to Proof of Concept

<https://gist.github.com/Valisthea/b2fb4324db71630e1a3d8476ada64704>

## Proof of Concept

Two runnable tests, both passing against the in-scope asset at commit `3b243b87`. The full existing suite still passes alongside them (53 passed, 0 failed), so nothing here is achieved by breaking or altering the project's own tests. The mock's `MaxGuardianAccounts` is deliberately left at its existing value of 10 rather than raised to the production 32, because raising it would break the project's own `guardian_index_respects_max_limit`, and because the bound's *value* is not the defect: it changes only how many transactions the attacker sends.

## Setup

{% stepper %}
{% step %}

### Clone the in-scope asset and check out the frozen branch

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-chain
cd audit-comp-quantus-chain
git checkout audit-comp-ready   # commit 3b243b870a5442a6e5f443056f67bbba87d472f5
```

{% endstep %}

{% step %}

### Save the attached file

Save the attached file as `pallets/reversible-transfers/src/tests/poc_guardian_index_squat.rs`.
{% endstep %}

{% step %}

### Register the test module

Append the following to `pallets/reversible-transfers/src/tests/mod.rs`:

```rust
#[cfg(test)]
mod poc_guardian_index_squat;
```

{% endstep %}

{% step %}

### Run the tests

```bash
cargo test -p pallet-reversible-transfers poc_
```

{% endstep %}
{% endstepper %}

Expected output:

```
running 2 tests
test tests::poc_guardian_index_squat::poc_guardian_slot_is_consumed_without_consent_and_target_need_not_exist ... ok
test tests::poc_guardian_index_squat::poc_no_dispatchable_can_evict_a_squatted_entry ... ok
test result: ok. 2 passed; 0 failed
```

## Test 1 — a slot is consumed without consent, against an account that does not exist

{% stepper %}
{% step %}

### Choose a target address

Choose a target address `victim_guardian`. Assert it does **not** exist on chain: `frame_system::Account::<Test>::contains_key(&victim_guardian)` is `false`. It has no genesis balance and no account entry. It is merely an address someone intends to publish.
{% endstep %}

{% step %}

### Assert the index is empty

Assert its guardian index starts empty.
{% endstep %}

{% step %}

### Consume a slot

A single unrelated account `attacker` calls `set_high_security(delay, victim_guardian)`. The target signs nothing and is never consulted.
{% endstep %}

{% step %}

### Observe the successful write

Observe the call succeeds and `GuardianIndex[victim_guardian]` now contains `attacker`.
{% endstep %}

{% step %}

### Confirm the target still does not exist

Assert the target **still does not exist on chain**. It never consented, and could not have consented, because there is nothing there to consent.
{% endstep %}
{% endstepper %}

This is the step the pallet's own `guardian_index_respects_max_limit` never performs: it establishes that the party whose bounded resource is consumed is not the party consuming it, and has no say in the matter.

## Test 2 — no dispatchable in the pallet can release an occupied slot

{% stepper %}
{% step %}

### Saturate the guardian index

Saturate `GuardianIndex[victim_guardian]` to `MaxGuardianAccounts` using throwaway accounts, each calling `set_high_security(delay, victim_guardian)`. Assert the index is full.
{% endstep %}

{% step %}

### Exercise every dispatchable

Exercise **every one of the pallet's six dispatchables**, from every origin with plausible standing, and observe that none removes an entry:

* call\_index 0, `set_high_security`, as the guardian itself
* call\_index 1, `cancel`, as the guardian
* call\_index 2, `execute_transfer`, as the guardian
* call\_index 3, `schedule_transfer`, as the guardian
* call\_index 4, `schedule_transfer_with_delay`, as the guardian
* call\_index 5, `recover_funds`, as the guardian, against **every** squatter. This is the strongest authority the pallet grants a guardian and it is asserted with `assert_ok!`, so the remediation genuinely executes rather than silently failing.
* `set_high_security` again as an occupier, showing the occupier has no exit either.

Several of these return `Err` (a dummy `tx_id` does not exist, for instance). That is not the point and is stated openly: the point is that no code path, successful or failed, ever removes an entry. The static counterpart is that `GuardianIndex::<T>` has exactly two write sites in the pallet, `lib.rs:394` (`try_push`) and `lib.rs:992` (genesis), and both are push-only.
{% endstep %}

{% step %}

### Confirm the index is unchanged

After the entire dispatchable surface has been exercised, assert the index length is **unchanged**. The occupation survives full guardian remediation.
{% endstep %}

{% step %}

### Attempt enrollment as an honest user

An honest user then calls `set_high_security(delay, victim_guardian)` and receives `Error::TooManyGuardianAccounts`.
{% endstep %}

{% step %}

### Confirm the honest user has no protection

Assert the honest user ends with `is_high_security() == None`: no protection at all, permanently.
{% endstep %}
{% endstepper %}

## Recommended fix

In descending order of preference:

1. **Drop `GuardianIndex` from consensus state.** It exists only for UI lookup (`lib.rs:243-245`: *"This allows the UI to efficiently query all accounts for which a given account is a guardian"*). Critically, `recover_funds` authorizes against `HighSecurityAccounts`, **not** against the index, so the index is load-bearing for no authorization decision. An off-chain indexer reconstructs it from the `HighSecuritySet` event. The bounded on-chain index buys nothing and costs a permanent griefing surface.
2. **Require guardian consent**, via a two-step propose/accept flow.
3. **Charge a refundable deposit** per index entry, making saturation costly and reclaimable.
4. **Add an eviction path** (`renounce_guardianship`, guardian-initiated), which the pallet lacks entirely.

Any fix also needs a storage migration to purge entries created before it lands.
