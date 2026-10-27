# 89136 bc critical missing deposit release calls in the high security whitelist permanently freeze reserved balances including an uninvolved third party s recovery deposit

**Submitted on Aug 17th 2026 at 12:40:04 UTC by @OxPrince for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89136
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

Quantus offers two account-protection features that the runtime deliberately wires together: **high-security mode**, which restricts an account to a closed whitelist of four calls, and **social recovery**, which requires refundable deposits from both the account owner and the rescuer.

None of the four whitelisted calls can decrease a `reserved` balance, so the two features cannot be combined without permanently freezing funds — and **no enrolment order avoids it**:

* Configure recovery first and its 10–19 UNIT deposit is stranded the moment you enrol.
* Enrol first and `create_recovery` is not whitelisted at all, so the account can never configure recovery.

The damage is not confined to the account owner. A rescuer who completes a fully legitimate M-of-N social recovery of a high-security account cannot reclaim their own 10 UNIT `RecoveryDeposit`: the release call, `close_recovery`, is dispatchable only *as the recovered account*, and the whitelist forbids it there.

The one recovery call the rescuer can dispatch themselves, `cancel_recovered`, refunds nothing and additionally destroys their proxy.

On mainnet parameters this strands roughly **29 UNIT per affected recovery, split across two parties**, with no user-reachable remedy at any time horizon and no on-chain signal warning either party beforehand.

## Vulnerability Details

### 1. The whitelist is a closed set, and nothing in it releases a reserve

`runtime/src/configs/mod.rs:683-693` defines every call a high-security account may dispatch:

```rust
fn is_whitelisted(call: &RuntimeCall) -> bool {
    matches!(
        call,
        RuntimeCall::ReversibleTransfers(pallet_reversible_transfers::Call::schedule_transfer { .. })
            | RuntimeCall::ReversibleTransfers(pallet_reversible_transfers::Call::cancel { .. })
            | RuntimeCall::ReversibleTransfers(pallet_reversible_transfers::Call::recover_funds { .. })
            | RuntimeCall::Vesting(pallet_vesting::Call::claim { .. })
    )
}
```

The policy is `is_call_allowed(who, call) = !is_high_security(who) || is_whitelisted(call)` (`primitives/high-security/src/lib.rs:132-147`).

It is enforced in the transaction extension and re-enforced at every origin-rewriting wrapper:

* `Recovery::as_recovered`
* `Utility::as_derivative`
* `Utility::dispatch_as`
* `Utility::batch*`
* `Utility::if_else`
* `Multisig::execute`

There is no wrapper through which a non-whitelisted call reaches a high-security origin.

Because the whitelist is a four-element `matches!`, exhaustiveness is verifiable by inspection in nine lines: **no call outside this set is reachable from a high-security origin, and none of these four decreases `AccountData::reserved`.**

| Whitelisted call    | What it moves     | Why reserved balance is untouched                                                                                                                                                                                    |
| ------------------- | ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `schedule_transfer` | free → hold       | `pallet_balances::hold` (`pallets/reversible-transfers/src/lib.rs:827-831`), bounded by `reducible_balance(who, Preservation::Preserve, ..)` — free balance only                                                     |
| `recover_funds`     | free → guardian   | The closing sweep (`pallets/reversible-transfers/src/lib.rs:608-619`) dispatches `Balances::transfer_all { keep_alive: false }`, which moves `reducible_balance(.., Expendable, ..)`; reserved is not part of `free` |
| `cancel`            | hold → recipient  | Operates on `HoldReason::ScheduledTransfer`, not on reserves                                                                                                                                                         |
| `Vesting::claim`    | pot → beneficiary | Inbound only                                                                                                                                                                                                         |

Worse, the outstanding reserve pins a **consumer reference**, so the account cannot be reaped and `Expendable` is additionally forced to leave the existential deposit behind.

No third party can help. `pallet-balances` exposes no signed `unreserve`. `remove_recovery` and `unnote_preimage` are depositor-only, and `close_recovery` is narrower still: it is dispatchable only by the **rescued account** — never by the rescuer whose funds it releases.

Every release path therefore terminates at an account the whitelist has gagged.

### 2. No enrolment order avoids the freeze

| Order                                   | Outcome                                                                                                                      |
| --------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `create_recovery` → `set_high_security` | The 10–19 UNIT config deposit is stranded permanently at the moment of enrolment.                                            |
| `set_high_security` → `create_recovery` | **Impossible.** `create_recovery` is not whitelisted, so a high-security account can never configure social recovery at all. |

This is the crux. There is no sequence that works, no parameter to get right, and no warning the user failed to heed.

A user who wants both documented protections must accept permanent loss of access to the deposit. And `set_high_security` is irreversible by design — enrolment can happen only once per account (`ensure!(!HighSecurityAccounts::contains_key(&who))`), so the state cannot be unwound.

### 3. The proximate cause is Quantus-authored, and it strands an uninvolved third party

The check that strands the rescuer is a Quantus addition to `pallet_recovery::as_recovered` (`pallets/recovery/src/lib.rs:452-458`, commit `d47fed0`, *"Fix: origin-rewriting wrappers bypass high-security restrictions"*):

```rust
let target = Self::proxy(&who).ok_or(Error::<T>::NotAllowed)?;
ensure!(target == account, Error::<T>::NotAllowed);
ensure!(
    T::HighSecurity::is_call_allowed(&account, &call),   // <-- Quantus-added
    Error::<T>::CallNotAllowedForHighSecurity
);
```

Upstream `pallet-recovery` contains no such check.

The consequence is that a rescuer who has completed the entire documented flow reaches a state where recovery demonstrably **works** for moving funds but cannot return their own money:

* `as_recovered(lost, ReversibleTransfers::schedule_transfer { .. })` → **succeeds**
* `as_recovered(lost, Recovery::close_recovery { rescuer })` → **fails**, `CallNotAllowedForHighSecurity`
* `as_recovered(lost, Recovery::remove_recovery {})` → **fails**, `CallNotAllowedForHighSecurity`

Nor can the rescuer route around the whitelist by calling the release path directly.

`close_recovery` takes the deposit from `ActiveRecoveries[origin][rescuer]` and repatriates it to `origin` — it is a *slashing* mechanism against malicious recovery attempts, dispatchable only by the rescued account.

The rescuer is therefore not merely blocked by policy; the call is not addressable by them at all, and its effect would be to pay the deposit **to** the frozen account rather than back.

Every alternative is closed too, by execution rather than argument.

The rescuer commits their 10 UNIT at `initiate_recovery`, **before any of this is observable**. No extrinsic, error, or event signals that the target is high-security.

They are a helpful third party who made no choice that caused this and holds no authority to undo it.

### 4. Which deposits are affected

| Source                            |                             Amount | Release call                        | Who may call it         | Stranded? |
| --------------------------------- | ---------------------------------: | ----------------------------------- | ----------------------- | --------- |
| `Recovery::create_recovery`       | 10 UNIT + 1 UNIT × friends (max 9) | `Recovery::remove_recovery`         | The account itself      | **Yes**   |
| `Recovery::initiate_recovery`     |   10 UNIT, held on the **rescuer** | `Recovery::close_recovery`          | The *recovered* account | **Yes**   |
| `Preimage::note_preimage`         |        1 UNIT + 1 MICRO\_UNIT/byte | `Preimage::unnote_preimage`         | The depositor           | **Yes**   |
| `Multisig::propose`               |                  `ProposalDeposit` | `remove_expired` / `claim_deposits` | **Any signer**          | No        |
| `Referenda` submission / decision |                          Per track | `refund_*_deposit`                  | **Permissionless**      | No        |

The bottom two rows are the tell. `pallet-multisig` deliberately made its refund callable by any signer, with the reasoning stated in its own doc comment (`pallets/multisig/src/lib.rs:899-904`):

> Approved+expired proposals can become stuck if proposer is unavailable (e.g. lost keys, compromise). **Allowing any signer to remove them prevents permanent deposit lockup** and enables multisig dissolution.

Referenda refunds are permissionless for the same reason. **The codebase already recognises this exact hazard and defends against it twice.** The high-security whitelist does not carry that defence across.

### 5. Why this is a defect, not an accepted design choice

**The pallet documents an exit path that does not exist.**

The "Warning: Permanent and Irreversible" block on `set_high_security` (`pallets/reversible-transfers/src/lib.rs:345-367`) tells users:

> Users who no longer wish to use high-security features can simply transfer their funds to a different account using [`schedule_transfer`](self::schedule_transfer).

That is false for reserved balance, and `schedule_transfer` is the very call the sentence names.

The request in this report is that behaviour match the documentation — not that new capability be added.

**The correct rule is already applied elsewhere in the same whitelist.**

`Vesting::claim` is whitelisted precisely because it is inbound-only: it cannot weaken the account or confer authority over it.

`remove_recovery`, `close_recovery` and `unnote_preimage` are inbound-only by the identical argument — each strictly *increases* the account's spendable balance.

The rule was recognised and applied once, then missed in three sibling cases.

**The integration was deliberate.**

`pallet_recovery::Config` for this runtime sets `type HighSecurity = HighSecurityConfig` (`runtime/src/configs/mod.rs:521`).

A composition built on purpose that cannot be exited is a defect in that composition, not a user error.

## Impact Details

### Funds frozen per occurrence

|                          Amount | What                                                                                                                              |
| ------------------------------: | --------------------------------------------------------------------------------------------------------------------------------- |
|                     **10 UNIT** | The rescuer's `RecoveryDeposit` — on an account that never enrolled in high security, whose owner made no choice that caused this |
|                  **10–19 UNIT** | The high-security account's `create_recovery` deposit (10 UNIT base + 1 UNIT × up to 9 friends)                                   |
| **1 UNIT + 1 MICRO\_UNIT/byte** | Any `note_preimage` deposit outstanding at enrolment                                                                              |
|               **1 MILLI\_UNIT** | The existential deposit, which the reserve's consumer reference forces `transfer_all` to leave behind                             |

Realistic worst case for one user who used both protections and then actually needed recovery: **\~29 UNIT plus ED, split across two parties.**

Constants at `runtime/src/configs/mod.rs:500-509`.

### Who is affected

Exactly the target population.

`set_high_security` is documented for *"institutional users and high-value accounts"* (`primitives/high-security/src/lib.rs:4-5`), and social recovery is its natural companion control — the runtime wires the two together on purpose.

The users most likely to adopt both are the ones this hits.

The rescuer variant is the more serious half: the loss falls on a **third party with no relationship to the enrolment decision**, who cannot inspect the hazard before committing funds and cannot release them afterwards.

## References

**Root cause**

* `runtime/src/configs/mod.rs:683-693` — the closed four-call whitelist
* `primitives/high-security/src/lib.rs:132-147` — `is_call_allowed` / `is_call_allowed_given`
* `pallets/recovery/src/lib.rs:452-458` — the Quantus-added `as_recovered` check (commit `d47fed0`)

**Blocked release paths**

* `pallets/recovery/src/lib.rs:712-748` — `close_recovery`, dispatchable only as the recovered account
* `pallets/reversible-transfers/src/lib.rs:608-619` — the `transfer_all` sweep in `recover_funds`
* `pallets/reversible-transfers/src/lib.rs:827-831` — `hold` in `schedule_transfer`

## Proof of Concept

```rust
// ============================================================================
//  PROOF OF CONCEPT — "Missing deposit-release calls in the high-security
//  whitelist permanently freeze reserved balances, including an uninvolved
//  third party's recovery deposit"
//
//  These two modules are reproduced verbatim from the audited tree at
//      runtime/src/transaction_extensions.rs
//  and run against the REAL runtime (`tests::new_test_ext`) — production
//  `HighSecurityConfig`, `pallet_recovery`, `pallet_reversible_transfers`
//  and `pallet_balances`. No mocks and no stubbed policy.
//
//  TO REPRODUCE
//  ------------
//  The modules are already present in the submitted tree. Simply run:
//
//      cargo test -p quantus-runtime --lib poc_high_security -- --nocapture
//
//  Expected output:
//
//      running 2 tests
//      test ...poc_high_security_reserved_freeze::audit_reserved_deposit_frozen_by_high_security_enrollment ... ok
//      test ...poc_high_security_strands_rescuer_deposit::audit_rescuer_deposit_stranded_by_high_security_target ... ok
//      test result: ok. 2 passed; 0 failed
//
//  If applying to a clean checkout instead, append both modules to the end of
//  runtime/src/transaction_extensions.rs (they depend only on that file's
//  existing `mod tests::new_test_ext` helper) and run the same command.
//
//  WHAT EACH TEST ESTABLISHES
//  --------------------------
//  audit_rescuer_deposit_stranded_by_high_security_target  (PRIMARY)
//    Runs the complete documented flow — create_recovery -> set_high_security
//    -> initiate_recovery -> 2x vouch_recovery -> claim_recovery — then shows
//    by real dispatch that recovery WORKS for moving funds
//    (as_recovered(schedule_transfer) succeeds) but the rescuer can never
//    reclaim their own 10 UNIT RecoveryDeposit. Every rescuer-side escape path
//    is then closed by execution, not by argument:
//      * close_recovery called directly  -> NotStarted (not addressable by them)
//      * poke_deposit                    -> refunds nothing
//      * cancel_recovered                -> refunds nothing AND destroys the proxy
//
//  audit_reserved_deposit_frozen_by_high_security_enrollment  (SUPPORTING)
//    Shows the owner-side variant: a 12 UNIT recovery-config deposit stranded
//    at enrolment, surviving even the guardian's recover_funds sweep, which
//    leaves free_balance == ED while reserved_balance == 12 UNIT.
// ============================================================================

/// PoC: funds RESERVED on an account before it enrolls in high security become
/// permanently unreachable — the high-security whitelist contains no call that can
/// release a reserve, and neither `schedule_transfer` nor the guardian's
/// `recover_funds` sweep can move reserved balance.
#[cfg(test)]
mod poc_high_security_reserved_freeze {
	use super::tests::new_test_ext;
	use crate::*;
	use frame_support::{
		assert_noop, assert_ok,
		traits::{Currency, ReservableCurrency},
	};
	use qp_high_security::HighSecurityInspector;
	use qp_scheduler::BlockNumberOrTimestamp;
	use sp_runtime::AccountId32;

	fn acc(b: u8) -> AccountId {
		AccountId32::from([b; 32])
	}

	#[test]
	fn audit_reserved_deposit_frozen_by_high_security_enrollment() {
		new_test_ext().execute_with(|| {
			System::set_block_number(1);

			let victim = acc(20);
			let guardian = acc(21);
			let f1 = acc(22);
			let f2 = acc(23);

			let _ = Balances::deposit_creating(&victim, 1_000 * UNIT);
			let _ = Balances::deposit_creating(&guardian, 10 * UNIT);

			// --- Step 1: the victim sets up social recovery (a documented, intended
			// security feature). This RESERVES ConfigDepositBase + 2 * FriendDepositFactor.
			let mut friends = vec![f1.clone(), f2.clone()];
			friends.sort();
			assert_ok!(Recovery::create_recovery(
				RuntimeOrigin::signed(victim.clone()),
				friends,
				2,
				10
			));
			let locked = Balances::reserved_balance(&victim);
			assert_eq!(locked, 12 * UNIT, "10 UNIT base + 2 x 1 UNIT per friend");

			// --- Step 2: the victim enrols in high security (the other flagship feature).
			// Nothing checks for, or warns about, the outstanding reserve.
			assert_ok!(ReversibleTransfers::set_high_security(
				RuntimeOrigin::signed(victim.clone()),
				BlockNumberOrTimestamp::BlockNumber(10),
				guardian.clone(),
			));

			// --- Step 3: every call that could release the reserve is now unreachable.
			let remove_recovery = RuntimeCall::Recovery(pallet_recovery::Call::remove_recovery {});
			let close_recovery = RuntimeCall::Recovery(pallet_recovery::Call::close_recovery {
				rescuer: sp_runtime::MultiAddress::Id(acc(24)),
			});
			for call in [remove_recovery, close_recovery] {
				assert!(
					!<crate::configs::HighSecurityConfig as HighSecurityInspector<
						AccountId,
						RuntimeCall,
					>>::is_call_allowed(&victim, &call),
					"whitelist unexpectedly permits {call:?}"
				);
			}

			// --- Step 4: the only fund-moving call the victim has left cannot touch it.
			// `schedule_transfer` holds from FREE balance only.
			let free = Balances::free_balance(&victim);
			assert_noop!(
				ReversibleTransfers::schedule_transfer(
					RuntimeOrigin::signed(victim.clone()),
					sp_runtime::MultiAddress::Id(guardian.clone()),
					free + 1,
				),
				sp_runtime::TokenError::FundsUnavailable
			);

			// --- Step 5: the guardian's emergency sweep also leaves it behind.
			assert_ok!(ReversibleTransfers::recover_funds(
				RuntimeOrigin::signed(guardian.clone()),
				victim.clone()
			));
			// The reserve keeps a consumer reference, so the account cannot be reaped and
			// `transfer_all(keep_alive: false)` is forced to leave the existential deposit
			// behind as well — that ED is stranded with the reserve.
			assert_eq!(
				Balances::free_balance(&victim),
				EXISTENTIAL_DEPOSIT,
				"sweep leaves the ED behind because the reserve pins a consumer ref"
			);
			assert_eq!(
				Balances::reserved_balance(&victim),
				locked,
				"12 UNIT is permanently stranded: no signed origin can ever release it"
			);

			// A second sweep changes nothing: the reserve is unreachable forever.
			assert_ok!(ReversibleTransfers::recover_funds(
				RuntimeOrigin::signed(guardian.clone()),
				victim.clone()
			));
			assert_eq!(Balances::reserved_balance(&victim), locked);
			assert_eq!(
				Balances::total_balance(&victim),
				locked + EXISTENTIAL_DEPOSIT,
				"12.001 UNIT permanently frozen"
			);
		});
	}
}

/// PoC (stronger variant): completing the protocol's own social-recovery remedy on a
/// high-security account permanently strands the RESCUER's `RecoveryDeposit`. The
/// rescuer never enrolled in high security and has no way to release their own funds:
/// `close_recovery` must be dispatched as the recovered (high-security) account, and
/// the whitelist forbids it. The one recovery call the rescuer CAN dispatch themselves,
/// `cancel_recovered`, refunds nothing and additionally destroys their proxy (step 6c).
#[cfg(test)]
mod poc_high_security_strands_rescuer_deposit {
	use super::tests::new_test_ext;
	use crate::*;
	use frame_support::{
		assert_noop, assert_ok,
		traits::{Currency, ReservableCurrency},
	};
	use sp_runtime::{AccountId32, MultiAddress};

	fn acc(b: u8) -> AccountId {
		AccountId32::from([b; 32])
	}

	#[test]
	fn audit_rescuer_deposit_stranded_by_high_security_target() {
		new_test_ext().execute_with(|| {
			System::set_block_number(1);

			let lost = acc(30); // victim, will become high-security
			let guardian = acc(31);
			let rescuer = acc(32); // the victim's NEW account after losing the key
			let f1 = acc(33);
			let f2 = acc(34);

			for a in [&lost, &guardian, &rescuer, &f1, &f2] {
				let _ = Balances::deposit_creating(a, 1_000 * UNIT);
			}

			// 1. Social recovery configured, then high security enabled.
			let mut friends = vec![f1.clone(), f2.clone()];
			friends.sort();
			assert_ok!(Recovery::create_recovery(
				RuntimeOrigin::signed(lost.clone()),
				friends,
				2,
				2, // delay_period in blocks
			));
			assert_ok!(ReversibleTransfers::set_high_security(
				RuntimeOrigin::signed(lost.clone()),
				qp_scheduler::BlockNumberOrTimestamp::BlockNumber(10),
				guardian.clone(),
			));

			// 2. The key is lost. The rescuer runs the documented recovery flow.
			let rescuer_before = Balances::free_balance(&rescuer);
			assert_ok!(Recovery::initiate_recovery(
				RuntimeOrigin::signed(rescuer.clone()),
				MultiAddress::Id(lost.clone())
			));
			let rescuer_deposit = Balances::reserved_balance(&rescuer);
			assert_eq!(rescuer_deposit, 10 * UNIT, "RecoveryDeposit");
			assert_eq!(Balances::free_balance(&rescuer), rescuer_before - 10 * UNIT);

			for f in [&f1, &f2] {
				assert_ok!(Recovery::vouch_recovery(
					RuntimeOrigin::signed(f.clone()),
					MultiAddress::Id(lost.clone()),
					MultiAddress::Id(rescuer.clone()),
				));
			}
			System::set_block_number(10);
			assert_ok!(Recovery::claim_recovery(
				RuntimeOrigin::signed(rescuer.clone()),
				MultiAddress::Id(lost.clone())
			));

			// 3. Recovery works for moving funds — `schedule_transfer` is whitelisted.
			assert_ok!(Recovery::as_recovered(
				RuntimeOrigin::signed(rescuer.clone()),
				MultiAddress::Id(lost.clone()),
				Box::new(RuntimeCall::ReversibleTransfers(
					pallet_reversible_transfers::Call::schedule_transfer {
						dest: MultiAddress::Id(rescuer.clone()),
						amount: 100 * UNIT,
					}
				)),
			));

			// 4. ...but reclaiming the rescuer's OWN deposit is blocked. `close_recovery`
			// can only be dispatched as the recovered account, and that account is
			// high-security.
			assert_noop!(
				Recovery::as_recovered(
					RuntimeOrigin::signed(rescuer.clone()),
					MultiAddress::Id(lost.clone()),
					Box::new(RuntimeCall::Recovery(pallet_recovery::Call::close_recovery {
						rescuer: MultiAddress::Id(rescuer.clone()),
					})),
				),
				pallet_recovery::Error::<Runtime>::CallNotAllowedForHighSecurity
			);
			// Same for the config deposit the lost account itself put up.
			assert_noop!(
				Recovery::as_recovered(
					RuntimeOrigin::signed(rescuer.clone()),
					MultiAddress::Id(lost.clone()),
					Box::new(RuntimeCall::Recovery(pallet_recovery::Call::remove_recovery {})),
				),
				pallet_recovery::Error::<Runtime>::CallNotAllowedForHighSecurity
			);

			// 5. The rescuer's 10 UNIT is still reserved, with no call left that can free it.
			assert_eq!(
				Balances::reserved_balance(&rescuer),
				10 * UNIT,
				"rescuer's own deposit is permanently frozen by the target's HS status"
			);
			// `reserved_balance` also counts the 100 UNIT `ScheduledTransfer` hold placed in
			// step 3, which IS recoverable (the guardian can cancel it). The stranded part
			// is the 12 UNIT recovery-config deposit underneath it.
			assert_eq!(
				Balances::reserved_balance(&lost),
				12 * UNIT + 100 * UNIT,
				"12 UNIT config deposit (stranded) + 100 UNIT scheduled-transfer hold"
			);

			// 6. Close every remaining path a reviewer would try, by execution rather than
			//    by appeal to the whitelist being a closed set.

			// 6a. The rescuer cannot invoke the release call directly. `close_recovery`
			//     reads `ActiveRecoveries[origin][rescuer]`, so it is dispatchable ONLY as
			//     the *rescued* account — which is exactly the account the whitelist gags.
			//     The rescuer is not merely blocked by policy; the call is not addressable
			//     by them at all.
			assert_noop!(
				Recovery::close_recovery(
					RuntimeOrigin::signed(rescuer.clone()),
					MultiAddress::Id(rescuer.clone())
				),
				pallet_recovery::Error::<Runtime>::NotStarted
			);
			assert_noop!(
				Recovery::close_recovery(
					RuntimeOrigin::signed(rescuer.clone()),
					MultiAddress::Id(lost.clone())
				),
				pallet_recovery::Error::<Runtime>::NotStarted
			);

			// 6b. `poke_deposit` is rescuer-callable and untouched by the whitelist, but it
			//     only ever refunds a DIFFERENCE against current parameters. With parameters
			//     unchanged it releases nothing.
			let before_poke = Balances::reserved_balance(&rescuer);
			assert_ok!(Recovery::poke_deposit(
				RuntimeOrigin::signed(rescuer.clone()),
				Some(MultiAddress::Id(lost.clone()))
			));
			assert_eq!(
				Balances::reserved_balance(&rescuer),
				before_poke,
				"poke_deposit refunds nothing while RecoveryDeposit is unchanged"
			);

			// 6c. `cancel_recovered` is the ONE recovery call the rescuer can successfully
			//     dispatch on their own. It succeeds — and makes the position strictly
			//     worse: it removes the proxy (so `as_recovered` is gone too) while leaving
			//     the deposit reserved. There is no rescuer-side cancel that refunds.
			assert_ok!(Recovery::cancel_recovered(
				RuntimeOrigin::signed(rescuer.clone()),
				MultiAddress::Id(lost.clone())
			));
			assert_eq!(
				Balances::reserved_balance(&rescuer),
				10 * UNIT,
				"cancel_recovered drops the proxy but refunds nothing"
			);
			assert!(
				pallet_recovery::Pallet::<Runtime>::proxy(&rescuer).is_none(),
				"the rescuer has now also lost the ability to act as the account"
			);
		});
	}
}
```
