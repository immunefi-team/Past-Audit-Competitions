# 89137 bc medium anyone can permanently exhaust any account s guardian slots for 0 032 unit blocking it from ever protecting another account

**Submitted on Aug 17th 2026 at 12:41:17 UTC by @AvrahamAdvisory for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89137
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

`set_high_security` lets any signed origin nominate an arbitrary account as its guardian. The nominee never consents and is never notified, yet each nomination permanently consumes one of that nominee's `MaxGuardianAccounts` slots in `GuardianIndex`. Because `GuardianIndex` has no removal path anywhere in the codebase, and because enrollment into high security is one-way, an attacker can fill any chosen account's 32 slots with throwaway accounts for roughly **0.032 UNIT** and permanently disqualify it from ever serving as a guardian for anyone else. On mainnet the most damaging target is a custody or guardian service intended to protect many users: the same 32-slot bound that makes the attack cheap is what makes the feature fragile at scale, and recovery requires a runtime upgrade with a storage migration.

## Vulnerability Details

### The three facts that make this permanent

`pallets/reversible-transfers/src/lib.rs:376-402`:

```rust
pub fn set_high_security(
    origin: OriginFor<T>,
    delay: BlockNumberOrTimestampOf<T>,
    guardian: T::AccountId,          // <-- arbitrary; no consent required
) -> DispatchResult {
    let who = ensure_signed(origin)?;   // <-- only the CALLER signs

    ensure!(guardian != who.clone(), Error::<T>::GuardianCannotBeSelf);
    ensure!(
        !HighSecurityAccounts::<T>::contains_key(&who),
        Error::<T>::AccountAlreadyHighSecurity     // <-- one-way: cannot undo or re-enroll
    );

    Self::validate_delay(&delay)?;

    let high_security_account_data =
        HighSecurityAccountData { guardian: guardian.clone(), delay };

    GuardianIndex::<T>::try_mutate(guardian.clone(), |accounts| {   // <-- :394
        if !accounts.contains(&who) {
            accounts.try_push(who.clone()).map_err(|_| Error::<T>::TooManyGuardianAccounts)
        } else {
            Ok(())
        }
    })?;                                                            // <-- aborts the whole extrinsic

    HighSecurityAccounts::<T>::insert(who.clone(), &high_security_account_data);
```

1. **No consent.** Only `who` (the caller) is authenticated. The `guardian` parameter is unvalidated beyond `!= who`. Nothing requires the nominee to accept, and no event obliges them to notice.
2. **No removal path.** `GuardianIndex` is written in exactly two places in the entire repository — the `try_mutate` at `:394` and the genesis builder at `:992`. There is no `remove`, no `retain`, no prune, and no extrinsic that shrinks it. I verified this by exhaustive grep across the whole chain repo.
3. **One-way enrollment.** `AccountAlreadyHighSecurity` (`:302`, `:386`) blocks re-enrollment, and there is no `HighSecurityAccounts::remove` call anywhere. So an attacker's throwaway accounts can never be un-enrolled to free their slots — **not even voluntarily**, and not by the victim, and not by any pallet-level governance call.

### Why the failure is total rather than graceful

The bounded push at `:394` runs **before** the `HighSecurityAccounts::insert` at `:403`. When the push fails, `TooManyGuardianAccounts` propagates and the whole extrinsic aborts, so an honest user who tries to nominate a saturated guardian is left with **no high-security record at all** — they are not silently enrolled with a fallback guardian, they are simply blocked from using the feature with that guardian.

### Parameters

`runtime/src/configs/mod.rs:528` — `MaxGuardianAccounts: u32 = 32`\
`runtime/src/lib.rs:102` — `EXISTENTIAL_DEPOSIT = MILLI_UNIT` (0.001 UNIT)

So saturating one victim costs `32 × 0.001 = 0.032 UNIT` plus transaction fees.

### Reproduction

{% stepper %}
{% step %}

## Pick a victim

Mallory picks a victim `V` — most damagingly a custody or guardian service that intends to protect many users.
{% endstep %}

{% step %}

## Create and fund throwaway accounts

Mallory creates 32 fresh accounts, each funded with the existential deposit (**0.032 UNIT total**).
{% endstep %}

{% step %}

## Nominate the victim as guardian

From each account, Mallory calls `set_high_security(delay = <any valid>, guardian = V)`. `V`'s consent is neither requested nor required, and `V` takes no action at any point.
{% endstep %}

{% step %}

## Fill the guardian index

`GuardianIndex[V]` now holds 32 entries and is full.
{% endstep %}

{% step %}

## Block legitimate users

Any honest user who subsequently calls `set_high_security(..., guardian = V)` fails with `TooManyGuardianAccounts`.
{% endstep %}

{% step %}

## Permanent denial

There is no recovery. No extrinsic removes an entry from `GuardianIndex`, and the squatting accounts cannot un-enroll. Only a runtime upgrade with a storage migration restores `V`.
{% endstep %}
{% endstepper %}

## Impact Details

**Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)** — Medium.

This is denial, not theft. No funds move, nothing already held is frozen, and the attacker gains nothing except the denial itself. Existing high-security relationships keep working; what is destroyed is the victim's **future** ability to act as a guardian, for everyone, permanently.

The severity comes from three properties rather than from magnitude:

* **Permanent.** Irreversible without a storage migration.
* **Cheap and permissionless.** \~0.032 UNIT, no privileged role, no victim interaction, no timing requirement.
* **Structurally amplified.** A guardian can protect at most 32 accounts chain-wide, so the service intended to guard the most users is simultaneously the most valuable target and the easiest to saturate.

I am explicitly **not** claiming any of the program's Critical impacts — no total network shutdown, no chain split, no direct loss of funds, no permanent freezing of funds, no unauthorized minting, no arbitrary command execution. None of them apply, and a report claiming otherwise should be downgraded.

### Why the Sybil exclusion does not apply

The program excludes "Impacts from Sybil attacks". Addressing that directly rather than leaving it for triage:

A Sybil attack is the creation of many identities to gain **disproportionate influence** in a system that weights decisions by identity count — voting power, reputation, consensus weight. That is not what happens here. Each squatting account performs exactly the action the pallet intends for it (enrolling in high security) and gains influence over nothing. The harm comes from a **missing removal path on a bounded list**: the same denial would be permanent if the cap were 3 and the attacker used 3 accounts. The defect is that `GuardianIndex` grows monotonically with no way to shrink, and that entries can be added to a third party's list without consent — both true regardless of how many identities exist.

The decisive test: raising `MaxGuardianAccounts`, which is the standard mitigation for a genuine Sybil concern, does **not** fix this. It only makes the attack linearly more expensive while leaving the permanence entirely intact.

### Relationship to declared known issues

None of the three declared known issues touch this area:

* referendum member votes not being removed on member removal
* the miner-to-node connection being plaintext on a trusted network
* zk-tree depth eventually exceeding its maximum, to be handled by a runtime upgrade

No privileged role is required at any step: `set_high_security` is callable by any signed origin (`ensure_signed`, `lib.rs:381`), so the "requires access to privileged addresses" exclusion does not apply either.

## Recommended fix

Any one of the first two closes it; the third is the structural option.

1. **Require guardian consent.** Make the nominee accept the role — a two-step `nominate_guardian` / `accept_guardianship` flow, mirroring the `set_recovered`-style patterns already present elsewhere in the runtime. This removes the ability to write into a stranger's index entirely.
2. **Provide a removal path.** Add an un-enroll extrinsic (subject to the reversibility delay so it cannot be used to escape a pending reversal) that removes the caller from both `HighSecurityAccounts` and `GuardianIndex[guardian]`. This also fixes the more general problem that high security is currently a one-way door for honest users.
3. Alternatively, drop `GuardianIndex` as consensus state altogether and let guardians discover their protected accounts by indexing `HighSecurityAccounts` off-chain, removing the bounded resource entirely.

{% hint style="warning" %}
Raising `MaxGuardianAccounts` is not a fix — it raises the attack's cost linearly while making the storage bound worse.
{% endhint %}

## References

In-scope code (`immunefi-team/audit-comp-quantus-chain` @ `audit-comp-ready`):

* `pallets/reversible-transfers/src/lib.rs:376-403` — `set_high_security`: no consent, bounded push before insert
* `pallets/reversible-transfers/src/lib.rs:394` — the only non-genesis writer of `GuardianIndex`
* `pallets/reversible-transfers/src/lib.rs:992` — the genesis writer (the only other one)
* `pallets/reversible-transfers/src/lib.rs:302`, `:386` — `AccountAlreadyHighSecurity`, making enrollment one-way
* `runtime/src/configs/mod.rs:528` — `MaxGuardianAccounts = 32`
* `runtime/src/lib.rs:102` — `EXISTENTIAL_DEPOSIT = MILLI_UNIT`

The pallet's own `guardian_index_respects_max_limit` (`src/tests/test_reversible_transfers.rs:1630`) already exercises the bounded-push failure, but frames it as a benign limit reached by legitimate users. The difference demonstrated here is that the filler is hostile, unrelated to the victim, and needs no consent from it.

## Link to Proof of Concept

<https://gist.github.com/shahar20201993-spec/eb55367262b32daae6b4741953fd4ad8>

## Proof of Concept

## Step-by-step reproduction

**Setup.** `MaxGuardianAccounts = 32` in the production runtime (`runtime/src/configs/mod.rs:528`) and `EXISTENTIAL_DEPOSIT = MILLI_UNIT` (`runtime/src/lib.rs:102`), so the whole attack costs `32 × 0.001 = 0.032 UNIT` plus fees.

{% stepper %}
{% step %}

## Pick a victim

Mallory chooses any account `V` — most damagingly a custody or guardian service that intends to protect many users. `V` is never contacted and takes no action at any point in this attack.
{% endstep %}

{% step %}

## Fund throwaway accounts

Fund 32 throwaway accounts, each with the existential deposit (0.032 UNIT total).
{% endstep %}

{% step %}

## Call `set_high_security`

From each throwaway account, call:

```
set_high_security(delay = <any value passing validate_delay>, guardian = V)
```

Each call succeeds. `V`'s consent is neither requested nor required — `guardian` is a plain parameter, and only the caller signs (`ensure_signed`, `lib.rs:381`).
{% endstep %}

{% step %}

## Fill `GuardianIndex[V]`

`GuardianIndex[V]` is now full — 32 entries, all placed by a hostile third party.
{% endstep %}

{% step %}

## Verify honest users are rejected

Any honest user who now calls `set_high_security(..., guardian = V)` **fails** with `TooManyGuardianAccounts`. Because the bounded push at `:394` runs *before* the `HighSecurityAccounts::insert` at `:403`, the whole extrinsic aborts: the honest user is left with no high-security record at all, not enrolled with some fallback guardian.
{% endstep %}

{% step %}

## Verify there is no recovery

No extrinsic in the pallet removes an entry from `GuardianIndex` (its only two writers are the `try_mutate` at `:394` and the genesis builder at `:992`), and the squatters cannot un-enroll because `AccountAlreadyHighSecurity` makes enrollment one-way. Only a runtime upgrade with a storage migration restores `V`.
{% endstep %}
{% endstepper %}

## Automated proof — 3 passing tests

Three tests against the **unmodified** pallet, using the project's own mock runtime (`MaxGuardianAccounts` is 10 there; 32 in production — the attack scales linearly and stays cheap either way). Verified with Rust 1.93.0, the version `rust-toolchain.toml` pins.

```
cargo test -p pallet-reversible-transfers poc_guardian

running 3 tests
test tests::poc_guardian_squat::attack_does_not_disturb_existing_relationships ... ok
test tests::poc_guardian_squat::attacker_permanently_blocks_a_guardian_service_without_its_consent ... ok
test tests::poc_guardian_squat::the_denial_is_permanent_no_recovery_path_exists ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 51 filtered out
```

**What each test establishes:**

* `attacker_permanently_blocks_a_guardian_service_without_its_consent` — the victim takes no action; unrelated throwaway accounts fill its slots; an honest user is then rejected with `TooManyGuardianAccounts`, and is asserted to hold **no** `HighSecurityAccounts` record.
* `the_denial_is_permanent_no_recovery_path_exists` — a squatter calling `set_high_security` again, even naming a *different* guardian, is rejected with `AccountAlreadyHighSecurity`. The index stays full and the honest user stays locked out.
* `attack_does_not_disturb_existing_relationships` — scoping the claim honestly: a user already protected by the victim keeps working. Only *future* guardianship is destroyed.

The pallet's own `guardian_index_respects_max_limit` (`src/tests/test_reversible_transfers.rs:1630`) already exercises the bounded-push failure, but frames it as a benign limit reached by legitimate users. The difference here is that the filler is hostile, unrelated to the victim, needs no consent, and the denial is permanent.

Full setup and a per-test breakdown are in the attached secret Gist (`SETUP.md`).
