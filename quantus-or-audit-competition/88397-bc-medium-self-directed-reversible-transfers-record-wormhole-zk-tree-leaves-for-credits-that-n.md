# 88397 bc medium self directed reversible transfers record wormhole zk tree leaves for credits that never happen

**Submitted on Aug 13th 2026 at 01:13:30 UTC by @humbleservant for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88397
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

### Brief/Intro

Every balance credit on Quantus is turned into a leaf of the wormhole ZK tree, and that tree is the only structure an exit proof proves membership against. Neither of the two mechanisms that create those leaves checks that value actually moved, and `pallet_reversible_transfers` reaches both of them with a credit where none did. Cancelling a one-time schedule returns the held principal to the sender with `transfer_on_hold(A -> A)`, which emits a `TransferOnHold` event that the event scanner turns into a leaf. Letting a schedule pointed at the sender's own address run to execution takes the `source == dest` early return inside the transfer primitive, moves nothing, and the pallet's explicit recorder mints a leaf anyway. Either way, anyone with a funded account writes arbitrary amounts of credit into the wormhole's deposit record for value that never changed hands, at the price of a transaction fee.

### Vulnerability Details

Leaves are created by two disjoint mechanisms. `WormholeProofRecorderExtension` scans the events a transaction emitted and converts four `pallet_balances` events into leaves, covering everything that happens in transaction context. Pallets whose credits land in hook context, where no transaction extension runs, instead call `TransferProofRecorder::record_transfer_proof` directly. Both mechanisms infer "value moved" from a proxy rather than checking it, and `pallet_reversible_transfers` breaks the proxy on both.

#### Instance one: the event scanner and `cancel`

`post_dispatch_details` matches three of its four arms unconditionally:

```rust
// runtime/src/transaction_extensions.rs:336-381
match event_record.event {
    RuntimeEvent::Balances(pallet_balances::Event::Transfer { from, to, amount }) => {
        let pot = vesting_pot.get_or_insert_with(
            pallet_vesting::Pallet::<Runtime>::pot_account_id,
        );
        (&from != pot && &to != pot).then_some((None, from, to, amount))
    },
    RuntimeEvent::Balances(pallet_balances::Event::Minted { who, amount }) => {
        let minting_account = crate::configs::MintingAccount::get();
        Some((None, minting_account, who, amount))
    },
    RuntimeEvent::Balances(pallet_balances::Event::TransferOnHold {
        source, dest, amount, ..
    }) => Some((None, source, dest, amount)),
    RuntimeEvent::Balances(pallet_balances::Event::ReserveRepatriated {
        from, to, amount, ..
    }) => Some((None, from, to, amount)),
    _ => None,
}
```

The only exclusion anywhere in the scanner is an address comparison against the vesting pot, and it sits inside the `Transfer` arm. Nothing checks the property that actually matters, which is whether the source and the destination are the same account.

`ReversibleTransfers::cancel` reaches the unguarded `TransferOnHold` arm. `cancel_transfer` reads the recipient out of state frozen at schedule time:

```rust
// pallets/reversible-transfers/src/lib.rs:874-885
let apply_fee = pending.guardian != pending.from;
let (recipient, apply_fee) = if apply_fee {
    ensure!(who == &pending.guardian, Error::<T>::InvalidReverser);
    (pending.guardian.clone(), true)
} else {
    ensure!(who == &pending.from, Error::<T>::NotOwner);
    (pending.from.clone(), false)
};
Self::release_held_funds_with_fee(&pending, &recipient, apply_fee)?;
```

For a schedule created through `schedule_transfer_with_delay`, `pending.guardian` is the sender, so `apply_fee` is false and `recipient == pending.from`. `release_held_funds_with_fee` then issues a self-directed hold transfer:

```rust
// pallets/reversible-transfers/src/lib.rs:943-951
pallet_balances::Pallet::<T>::transfer_on_hold(
    &HoldReason::ScheduledTransfer.into(),
    &pending.from,   // source
    recipient,       // == &pending.from
    remaining_amount,
    Precision::Exact,
    Restriction::Free,
    Fortitude::Polite,
)?;
```

`fungible::hold::Mutate::transfer_on_hold` has no `source == dest` short circuit and calls `done_transfer_on_hold` unconditionally (`frame/support/src/traits/tokens/fungible/hold.rs:378-385`), which emits `TransferOnHold { source: A, dest: A, amount }` (`pallets/balances/src/impl_fungible.rs:227-240`). The scanner sees it, calls `record_transfer_proof(None, A, A, amount)`, and a leaf crediting `A` the full amount lands in the ZK tree. `A`'s total balance is unchanged by the release; the funds moved from held to free inside one account.

#### Instance two: the explicit recorder and scheduled execution

The same pallet reaches the other mechanism with the same kind of credit. `do_execute_transfer` releases the hold, dispatches an inner `transfer_keep_alive`, and records a leaf on the strength of the dispatch result alone:

```rust
// pallets/reversible-transfers/src/lib.rs:697-732
let to_lookup = T::Lookup::unlookup(pending.to.clone());
let call: RuntimeCallOf<T> = pallet_balances::Call::<T>::transfer_keep_alive {
    dest: to_lookup,
    value: pending.amount,
}
.into();

pallet_balances::Pallet::<T>::release(
    &HoldReason::ScheduledTransfer.into(),
    &pending.from,
    pending.amount,
    Precision::Exact,
)?;
// ... storage cleanup ...
let post_info =
    call.dispatch(frame_system::RawOrigin::Signed(pending.from.clone()).into());

// Record transfer proof if dispatch was successful
if post_info.is_ok() {
    T::ProofRecorder::record_transfer_proof(
        pending.asset_id.clone(),
        pending.from.clone(),
        pending.to.clone(),
        pending.amount,
    );
}
```

`post_info.is_ok()` is not evidence that value moved. Schedule a transfer to your own address and the inner `transfer_keep_alive` takes the `source == dest` early return, moves nothing, and returns `Ok`. The hold has already been released back to the sender, so the sender ends the block with exactly the balance it started with, and a leaf for the full scheduled amount is inserted. No `Balances::Transfer` event exists for that execution at all, so the scanner never even sees it. My third test asserts both halves of that.

This matters for the fix. A `source != dest` guard on the `TransferOnHold` match arm stops instance one and leaves instance two running, which I confirmed by applying it and re-running.

### Why this is a gap and not intended behaviour

`record_transfer_proof` is the single chokepoint every recorder passes through, and it already drops one class of value-less credit, with a comment that describes this bug's harm precisely:

```rust
// pallets/wormhole/src/lib.rs:1347-1356
// A zero-amount credit moves no value, so a leaf for it is pure state growth:
// it would advance the recipient's transfer count, enlarge the ZK tree, and
// emit a transfer event for nothing. [...]
if amount.is_zero() {
    return false;
}
```

A self-directed credit moves no value for exactly the reason a zero-amount credit does. The chokepoint filters on `amount == 0` and never on `source == dest`.

The transfer primitives point the same way. The unheld path short-circuits self-transfers before `done_transfer`, so a plain self-directed `transfer_keep_alive` emits no event and produces no leaf:

```rust
// frame/support/src/traits/tokens/fungible/regular.rs:326-334
let dust = Self::can_withdraw(source, amount).into_result(preservation != Expendable)?;
let actual = amount.saturating_add(dust);
Self::can_deposit(dest, actual, Extant).into_result()?;
if source == dest {
    // Nothing is debited and no reap occurs on the self path, so the reported amount must
    // not include the hypothetical dust.
    return Ok(amount)
}
```

The `if source == dest` line there is inherited from upstream, but the surrounding reasoning is not: commit `db94e2a` ("Harden FRAME primitives against audit findings") reworked that exact branch and added the comment about nothing being debited on the self path. Commit `96de0cf` ("Frame fixes medium low") separately edited `transfer_on_hold` in `hold.rs`, adding the destination-existence check for `Restriction::OnHold` mode, and left the function with no self-transfer short circuit at all. So both transfer primitives were reviewed and hardened in this repository, the self-directed case was reasoned about explicitly on one of them, and the recorder that consumes their events was written here from scratch (`runtime/src/transaction_extensions.rs` has 37 commits, all of them Quantus's). The gap is between those pieces, not inside any of them.

Commit `3b243b87` shows the classifier being worked on two days before the code froze: "fix: fee-charge proof inserts for cancel, recover\_funds and close\_recovery. The static `count_transfers` classifier let `ReversibleTransfers::cancel` ... fall through to the zero arm, even though they emit `TransferOnHold` / `ReserveRepatriated` events ... that the post-dispatch scanner turns into ZK-tree proof inserts." The weight charged for the insert was corrected. Whether the insert should happen at all on the self-directed branch was not asked.

The reason no existing test catches this is the harness. `runtime/tests/transactions/reversible_integration.rs` drives the pallet with `RuntimeOrigin::signed` directly, so the transaction-extension pipeline never runs and no leaf is ever recorded in those tests. My PoC goes through `Executive::apply_extrinsic` with real ML-DSA-65 signatures and the complete `TxExtension` tuple, which is the only way to observe the scanner.

### What this is not

Both instances credit an account that signed for the schedule, so `leaf.to = hash_bytes(ml_dsa_pubkey)` (`primitives/dilithium-crypto/src/traits.rs:191-201`). Spending a leaf requires proving knowledge of a secret `s` with `H(H("wormhole" || s)) == leaf.to` (`wormhole/circuit/src/circuit.rs:247-255` into `unspendable_account.rs:215-237`). Both sets are Poseidon2 images and landing in both at once is a preimage problem, so these leaves are not exitable and this is not unauthorized minting today.

Instance two is the one worth checking carefully, because `pending.to` is fully attacker-chosen there and could name a wormhole address. It cannot produce a phantom leaf when it does: the no-op only happens when `to == from`, and any `to != from` moves the value for real, which makes the leaf correct. I also checked every origin-rewriting route that could put a non-signing address into `pending.from`, since that is the other way to break the tie: `Utility::as_derivative`, `Utility::dispatch_as` (Root only), `Recovery::as_recovered` (requires the account to have created its own recovery configuration), and `Multisig::as_multi_threshold_1`. Each derived address is a Blake2 image, so aiming one at a chosen wormhole address is a cross-family collision search. `execute_transfer` itself is gated on `who == Self::account_id()` (`pallets/reversible-transfers/src/lib.rs:443`), a keyless pallet account, so no signed origin reaches it. `set_high_security` does accept an arbitrary guardian address, but both `cancel` and `recover_funds` require the guardian to sign, so a keyless wormhole address can never end up as the credited party there either.

## Impact Details

This maps to the in-scope Blockchain/DLT impact "A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk". A defect in the runtime's proof recorder produces a wormhole deposit leaf and a `NativeTransferred` event for a credit that never occurred, and no funds are at direct risk because the leaf cannot be spent.

On the wording: Quantus supports no token standards and ships no contracts pallet, so "smart contract behavior" has no referent on this chain at all. That impact nonetheless appears on this program's own Blockchain/DLT list, so its operative clause here is unintended behaviour in the layer-1 code with no funds at direct risk, which is what this is. If you read that impact strictly by its wording instead, the fallback is the Medium "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)": there is no profit motive, and the damage is a permanently wrong wormhole deposit record plus a stream of transfer events that describe transfers nobody made.

Three concrete consequences on mainnet:

* The ZK tree permanently contains leaves asserting credits that never happened, and nothing on chain reconciles recorded leaf value against real value any more. The pallet dropped its `PotentialWormholeBalance` and `TotalWormholeExits` counters in the v1 to v2 storage migration five days before the code froze (`pallets/wormhole/src/lib.rs:246-258`), so the recorded total is now the only account of what the tree claims, and it is wrong by an attacker-chosen amount.
* A `NativeTransferred { from: A, to: A, amount }` event fires for a transfer that did not occur. The pallet documents these as the events indexers such as Subsquid consume (`pallets/wormhole/src/lib.rs:459-467`), so downstream balance and volume reconstruction sees traffic that never existed, at whatever scale an attacker chooses to pay for.
* `TransferCount[A]` advances for a credit that never happened, which is the second harm the zero-amount guard names in its own comment.

There is also a forward risk I would not file on its own but which changes how I would prioritise the fix. The moment any path lets a keyless or wormhole-derived address occupy `pending.from`, whether through a pallet-scheduled reversible transfer, a hold placed on a vesting or treasury sub-account, or a future recovery flow, these same three lines start minting exit capacity that a secret holder can spend. The guard that currently keeps this inert is an accident of which accounts can reach the path, not a property the recorder checks.

I am deliberately not claiming this as a cheap state-growth attack. An ordinary transfer already inserts one leaf for one fee, which is cheaper per leaf than the two fees a schedule-and-cancel pair costs, and the known issue about zk-tree depth covers volume-driven capacity exhaustion on its own terms. The defect here is that a leaf records value that never moved, not that leaves can be created quickly.

## References

* `runtime/src/transaction_extensions.rs:336-381` and `:423-435, :450-473`. The event scan, the vesting-pot filter that covers only the `Transfer` arm, the `prepare()` snapshot of `event_count()`, and the `post_dispatch_details` that consumes it.
* `pallets/reversible-transfers/src/lib.rs:865-909` (`cancel_transfer`), `:917-954` (`release_held_funds_with_fee`), `:411-416` (the `cancel` dispatchable), `:465-484` (`schedule_transfer_with_delay`), `:693-738` (`do_execute_transfer` and its `post_info.is_ok()` record), `:434-446` (`execute_transfer` and its keyless-origin gate).
* `frame/support/src/traits/tokens/fungible/hold.rs:346-386`. `transfer_on_hold` with no self-transfer short circuit.
* `frame/support/src/traits/tokens/fungible/regular.rs:318-343`. The self-transfer short circuit that the unheld path does have.
* `pallets/balances/src/impl_fungible.rs:227-240`. `done_transfer_on_hold` emits the event unconditionally.
* `pallets/wormhole/src/lib.rs:1341-1392` (`record_transfer_proof` and its zero-amount drop), `:246-258` (the storage migration that removed the wormhole balance counters), `:459-467` (the `NativeTransferred` event contract).
* `wormhole/circuit/src/circuit.rs:247-255` and `wormhole/circuit/src/unspendable_account.rs:215-237` in `qp-zk-circuits` v4.2.0, for the constraint that makes the leaf unspendable.
* Commit `3b243b87` ("fix: fee-charge proof inserts for cancel, recover\_funds and close\_recovery").
* Checked against the program's three listed known issues (referendum vote removal, plaintext miner-node connection, zk-tree depth versus transaction volume) and against the four public audit reports (Equilibrium on `qp-poseidon`, `qp-zk-circuits` and `chain`, Neodyme on `qp-rusty-crystals`). None of them covers a leaf recorded for a credit that did not occur.

## Proof of Concept

Four tests: the scanner instance, the explicit-recorder instance, a control on the sibling branch of the same function where value genuinely moves, and a repetition test. Every extrinsic in all four goes through `Executive::apply_extrinsic` with a real ML-DSA-65 signature and the full `TxExtension` tuple, which is the only way to reach the scanner at all.

One precision note on the third test. Its scheduling half is submitted as a signed extrinsic like everything else, but the execution half is driven by calling `Scheduler::on_finalize` and `Scheduler::on_initialize` around `System::set_block_number`, which does not run `frame_system::initialize` the way a real block does. That is faithful for what it demonstrates, because the recorder call at `pallets/reversible-transfers/src/lib.rs:726` is explicit rather than event-derived, so no scan window is involved and the event indices are irrelevant to it. In a real block the scheduler dispatch runs inside `on_initialize`, whose events sit below every extrinsic's snapshot (`frame/executive/src/lib.rs:641` resets events before the hooks at `:662-665`), which changes nothing about which recorder fires here.

Save as `runtime/tests/transactions/wormhole_phantom_leaf.rs` and add `pub mod wormhole_phantom_leaf;` to `runtime/tests/transactions/mod.rs`.

```rust
//! A self-directed reversible-transfer cancel records a wormhole ZK-tree leaf
//! for a credit that moves no value.

use codec::Encode;
use frame_support::traits::{Currency, OnFinalize, OnInitialize};
use qp_dilithium_crypto::Dilithium65Pair;
use qp_scheduler::BlockNumberOrTimestamp;
use quantus_runtime::{
	transaction_extensions::{ReversibleTransactionExtension, WormholeProofRecorderExtension},
	Balances, Runtime, RuntimeCall, RuntimeEvent, Signature, SignedPayload, System, TxExtension,
	UncheckedExtrinsic, ZkTree, UNIT, VERSION,
};
use sp_core::Pair;
use sp_runtime::{generic::Era, traits::IdentifyAccount, AccountId32, MultiAddress};

type AccountId = AccountId32;
type AssetId = u32;
type Balance = u128;

fn test_ext(funded: &[&AccountId]) -> sp_io::TestExternalities {
	use quantus_runtime::BuildStorage;
	let t = frame_system::GenesisConfig::<Runtime>::default().build_storage().unwrap();
	let mut ext = sp_io::TestExternalities::new(t);
	ext.execute_with(|| {
		for a in funded {
			Balances::make_free_balance_be(a, 100_000 * UNIT);
		}
		System::set_block_number(1);
	});
	ext
}

/// Sign an arbitrary `RuntimeCall` with ML-DSA-65 and wrap it in the runtime's
/// full `TxExtension` tuple.
fn sign_call(
	pair: &Dilithium65Pair,
	sender: AccountId,
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
	let signature = raw_payload.using_encoded(|e| pair.sign(e));
	UncheckedExtrinsic::new_signed(
		call,
		MultiAddress::Id(sender),
		Signature::Dilithium65(signature),
		tx_ext,
	)
}

fn submit(pair: &Dilithium65Pair, sender: &AccountId, call: RuntimeCall, nonce: u32) {
	let xt = sign_call(pair, sender.clone(), call, nonce);
	quantus_runtime::Executive::apply_extrinsic(xt)
		.expect("extrinsic applies")
		.expect("dispatch succeeds");
}

fn newest_leaf() -> pallet_zk_tree::ZkLeaf<AccountId, AssetId, Balance> {
	let n = ZkTree::leaf_count();
	assert!(n > 0, "expected at least one leaf");
	pallet_zk_tree::Leaves::<Runtime>::get(n - 1).expect("newest leaf exists")
}

fn last_scheduled_tx_id() -> <Runtime as frame_system::Config>::Hash {
	System::events()
		.iter()
		.rev()
		.find_map(|r| {
			if let RuntimeEvent::ReversibleTransfers(
				pallet_reversible_transfers::Event::TransactionScheduled { tx_id, .. },
			) = &r.event
			{
				Some(*tx_id)
			} else {
				None
			}
		})
		.expect("TransactionScheduled emitted")
}

/// The bug: `cancel` on a one-time schedule releases the hold to the sender
/// itself, and the recorder turns that self-directed `TransferOnHold` into a
/// ZK-tree leaf crediting the sender the full scheduled amount. No value moved.
#[test]
fn self_cancel_records_a_wormhole_leaf_for_a_zero_value_credit() {
	let pair = Dilithium65Pair::from_seed_slice(&[7u8; 32]).expect("valid seed");
	let a = pair.public().into_account();
	let recipient = AccountId32::new([0xB0; 32]); // schedule destination, never paid
	let control_dest = AccountId32::new([0xC0; 32]);

	let mut ext = test_ext(&[&a]);
	ext.execute_with(|| {
		let scheduled = 100 * UNIT;

		submit(
			&pair,
			&a,
			pallet_reversible_transfers::Call::<Runtime>::schedule_transfer_with_delay {
				dest: MultiAddress::Id(recipient.clone()),
				amount: scheduled,
				delay: BlockNumberOrTimestamp::BlockNumber(5),
			}
			.into(),
			0,
		);
		let tx_id = last_scheduled_tx_id();

		let a_total_before = Balances::total_balance(&a);
		let leaves_before = ZkTree::leaf_count();
		System::reset_events();

		submit(
			&pair,
			&a,
			pallet_reversible_transfers::Call::<Runtime>::cancel { tx_id }.into(),
			1,
		);

		let a_total_after = Balances::total_balance(&a);

		assert!(
			a_total_after < a_total_before,
			"A's total balance falls by the cancel fee, it does not rise"
		);
		let a_lost = a_total_before - a_total_after;
		assert!(
			a_lost < scheduled / 1000,
			"A's real balance change ({a_lost}) is a fee, nowhere near the leaf's {scheduled}"
		);

		assert_eq!(
			Balances::total_balance(&recipient),
			0,
			"the scheduled destination received nothing"
		);

		assert_eq!(
			ZkTree::leaf_count(),
			leaves_before + 1,
			"exactly one leaf recorded by the self-cancel"
		);
		let leaf = newest_leaf();
		assert_eq!(leaf.to, a, "the leaf credits A itself");
		assert_eq!(leaf.amount, scheduled, "the leaf carries the full scheduled amount");

		let phantom_event = System::events().iter().any(|r| {
			matches!(
				&r.event,
				RuntimeEvent::Wormhole(pallet_wormhole::Event::NativeTransferred { from, to, amount, .. })
					if from == &a && to == &a && *amount == scheduled
			)
		});
		assert!(phantom_event, "a self-directed NativeTransferred event was emitted");

		let control_amount = 50 * UNIT;
		let control_dest_before = Balances::total_balance(&control_dest);
		let leaves_pre_control = ZkTree::leaf_count();
		System::reset_events();

		submit(
			&pair,
			&a,
			pallet_balances::Call::<Runtime>::transfer_keep_alive {
				dest: MultiAddress::Id(control_dest.clone()),
				value: control_amount,
			}
			.into(),
			2,
		);

		assert_eq!(
			Balances::total_balance(&control_dest),
			control_dest_before + control_amount,
			"control: the credited account's balance rises by the full amount"
		);
		assert_eq!(
			ZkTree::leaf_count(),
			leaves_pre_control + 1,
			"control: exactly one leaf for the real transfer"
		);
		let control_leaf = newest_leaf();
		assert_eq!(control_leaf.to, control_dest, "control leaf credits the real recipient");
		assert_eq!(
			control_leaf.amount, control_amount,
			"control leaf carries the transferred amount"
		);
	});
}

/// Second control, on the sibling branch of the same function: when a guardian
/// cancels a high-security account's transfer, `recipient != from`, the value
/// genuinely moves, and the recorded leaf matches the guardian's balance delta.
#[test]
fn guardian_cancel_records_a_leaf_that_matches_a_real_credit() {
	let b_pair = Dilithium65Pair::from_seed_slice(&[8u8; 32]).expect("valid seed");
	let g_pair = Dilithium65Pair::from_seed_slice(&[9u8; 32]).expect("valid seed");
	let b = b_pair.public().into_account();
	let g = g_pair.public().into_account();
	let recipient = AccountId32::new([0xB1; 32]);

	let mut ext = test_ext(&[&b, &g]);
	ext.execute_with(|| {
		let scheduled = 100 * UNIT;

		submit(
			&b_pair,
			&b,
			pallet_reversible_transfers::Call::<Runtime>::set_high_security {
				delay: BlockNumberOrTimestamp::BlockNumber(5),
				guardian: g.clone(),
			}
			.into(),
			0,
		);
		submit(
			&b_pair,
			&b,
			pallet_reversible_transfers::Call::<Runtime>::schedule_transfer {
				dest: MultiAddress::Id(recipient.clone()),
				amount: scheduled,
			}
			.into(),
			1,
		);
		let tx_id = last_scheduled_tx_id();

		let g_before = Balances::total_balance(&g);
		let leaves_before = ZkTree::leaf_count();
		System::reset_events();

		submit(
			&g_pair,
			&g,
			pallet_reversible_transfers::Call::<Runtime>::cancel { tx_id }.into(),
			0,
		);

		assert_eq!(
			ZkTree::leaf_count(),
			leaves_before + 1,
			"exactly one leaf for the guardian cancel"
		);
		let leaf = newest_leaf();
		assert_eq!(leaf.to, g, "the leaf credits the guardian");

		let g_after = Balances::total_balance(&g);
		assert!(g_after > g_before, "the guardian actually received funds");
		let credited = g_after - g_before;
		assert!(
			credited <= leaf.amount && leaf.amount - credited < scheduled / 1000,
			"guardian's real credit ({credited}) equals the leaf amount ({}) net of its own tx fee",
			leaf.amount
		);
	});
}

/// Second instance of the same class, on a different recording mechanism.
/// `do_execute_transfer` records a leaf whenever the inner dispatch returns
/// `Ok`, without checking that value moved. A schedule pointed at the sender's
/// own address makes `Mutate::transfer` take its `source == dest` early return,
/// so nothing moves, the dispatch still succeeds, and the explicit recorder
/// mints a leaf for the full amount.
#[test]
fn self_directed_scheduled_transfer_records_a_leaf_on_execution() {
	let pair = Dilithium65Pair::from_seed_slice(&[13u8; 32]).expect("valid seed");
	let a = pair.public().into_account();

	let mut ext = test_ext(&[&a]);
	ext.execute_with(|| {
		let scheduled = 250 * UNIT;

		submit(
			&pair,
			&a,
			pallet_reversible_transfers::Call::<Runtime>::schedule_transfer_with_delay {
				dest: MultiAddress::Id(a.clone()),
				amount: scheduled,
				delay: BlockNumberOrTimestamp::BlockNumber(2),
			}
			.into(),
			0,
		);

		let a_before = Balances::total_balance(&a);
		let leaves_before = ZkTree::leaf_count();

		let start = System::block_number();
		for n in start..start + 4 {
			quantus_runtime::Scheduler::on_finalize(n);
			System::set_block_number(n + 1);
			quantus_runtime::Scheduler::on_initialize(n + 1);
		}

		let a_after = Balances::total_balance(&a);

		assert_eq!(
			a_after, a_before,
			"a self-directed scheduled transfer moves no value at execution"
		);
		assert_eq!(
			ZkTree::leaf_count(),
			leaves_before + 1,
			"the explicit recorder still mints a leaf"
		);
		let leaf = newest_leaf();
		assert_eq!(leaf.to, a, "the leaf credits A");
		assert_eq!(leaf.amount, scheduled, "the leaf carries the full scheduled amount");

		let moved_event = System::events().iter().any(|r| {
			matches!(
				&r.event,
				RuntimeEvent::Balances(pallet_balances::Event::Transfer { from, to, .. })
					if from == &a && to == &a
			)
		});
		assert!(!moved_event, "no Balances::Transfer event exists for the no-op transfer");
	});
}

/// The self-cancel is repeatable at fee cost, and each repetition inserts
/// another permanent leaf carrying the full nominal amount.
#[test]
fn repeated_self_cancels_accumulate_unbacked_leaf_value() {
	let pair = Dilithium65Pair::from_seed_slice(&[11u8; 32]).expect("valid seed");
	let a = pair.public().into_account();
	let recipient = AccountId32::new([0xB2; 32]);

	let mut ext = test_ext(&[&a]);
	ext.execute_with(|| {
		let scheduled = 1_000 * UNIT;
		let rounds = 5u64;

		let a_before = Balances::total_balance(&a);
		let leaves_before = ZkTree::leaf_count();
		let mut nonce = 0u32;
		let mut leaf_value_recorded: Balance = 0;

		for _ in 0..rounds {
			submit(
				&pair,
				&a,
				pallet_reversible_transfers::Call::<Runtime>::schedule_transfer_with_delay {
					dest: MultiAddress::Id(recipient.clone()),
					amount: scheduled,
					delay: BlockNumberOrTimestamp::BlockNumber(5),
				}
				.into(),
				nonce,
			);
			nonce += 1;
			let tx_id = last_scheduled_tx_id();
			submit(
				&pair,
				&a,
				pallet_reversible_transfers::Call::<Runtime>::cancel { tx_id }.into(),
				nonce,
			);
			nonce += 1;
			leaf_value_recorded += newest_leaf().amount;
		}

		let a_after = Balances::total_balance(&a);
		assert_eq!(
			ZkTree::leaf_count(),
			leaves_before + rounds,
			"one phantom leaf per round"
		);
		assert_eq!(
			leaf_value_recorded,
			scheduled * rounds as Balance,
			"aggregate recorded leaf value scales linearly with the number of cancels"
		);
		assert!(a_after < a_before, "A only ever loses fees");
		let spent = a_before - a_after;
		assert!(
			spent < scheduled,
			"total cost to A ({spent}) is a handful of fees against {leaf_value_recorded} of recorded leaf value"
		);
		assert_eq!(
			Balances::total_balance(&recipient),
			0,
			"the scheduled destination was never paid in any round"
		);
	});
}
```

Run command, from the repository root on WSL2 Debian with the repository's pinned 1.93.0 toolchain:

```bash
QP_NUM_LEAF_PROOFS=1 QP_NUM_PRIVATE_BATCH_PROOFS=1 SKIP_WASM_BUILD=1 \
RUSTFLAGS="-A unused_imports" \
cargo test -p quantus-runtime --test mod wormhole_phantom_leaf
```

`RUSTFLAGS="-A unused_imports"` works around an unrelated rustc 1.93.0 internal compiler error in `check_unused_traits`, which the two pre-existing unused trait imports at `runtime/tests/governance/tech_collective.rs:29` and `:45` trigger intermittently. Without it the test crate sometimes fails to build with "error: the compiler unexpectedly panicked".

Expected output:

```
running 4 tests
test transactions::wormhole_phantom_leaf::self_cancel_records_a_wormhole_leaf_for_a_zero_value_credit ... ok
test transactions::wormhole_phantom_leaf::self_directed_scheduled_transfer_records_a_leaf_on_execution ... ok
test transactions::wormhole_phantom_leaf::guardian_cancel_records_a_leaf_that_matches_a_real_credit ... ok
test transactions::wormhole_phantom_leaf::repeated_self_cancels_accumulate_unbacked_leaf_value ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 42 filtered out; finished in 0.09s
```

To show the tests measure this defect and nothing else, and to show where the fix has to go, I ran two candidate patches.

The narrow patch, on the `TransferOnHold` arm at `runtime/src/transaction_extensions.rs:360-365`:

```rust
RuntimeEvent::Balances(pallet_balances::Event::TransferOnHold { source, dest, amount, .. })
    => (source != dest).then_some((None, source, dest, amount)),
```

stops the scanner instance and leaves the explicit-recorder instance running:

```
test transactions::wormhole_phantom_leaf::self_cancel_records_a_wormhole_leaf_for_a_zero_value_credit ... FAILED
test transactions::wormhole_phantom_leaf::repeated_self_cancels_accumulate_unbacked_leaf_value ... FAILED
test transactions::wormhole_phantom_leaf::self_directed_scheduled_transfer_records_a_leaf_on_execution ... ok
test transactions::wormhole_phantom_leaf::guardian_cancel_records_a_leaf_that_matches_a_real_credit ... ok
```

The chokepoint patch, in `record_transfer_proof` at `pallets/wormhole/src/lib.rs:1354-1356`, immediately after the existing zero-amount drop:

```rust
if amount.is_zero() {
    return false;
}
// A credit whose source and destination are the same account moves no
// value either, for exactly the same reason.
if from == to {
    return false;
}
```

stops all three defect tests while the legitimate guardian path still records its leaf:

```
test transactions::wormhole_phantom_leaf::self_cancel_records_a_wormhole_leaf_for_a_zero_value_credit ... FAILED
test transactions::wormhole_phantom_leaf::repeated_self_cancels_accumulate_unbacked_leaf_value ... FAILED
test transactions::wormhole_phantom_leaf::self_directed_scheduled_transfer_records_a_leaf_on_execution ... FAILED
test transactions::wormhole_phantom_leaf::guardian_cancel_records_a_leaf_that_matches_a_real_credit ... ok

assertion `left == right` failed: exactly one leaf recorded by the self-cancel
assertion `left == right` failed: the explicit recorder still mints a leaf
```

## Suggested fix

Drop the credit at the chokepoint, as in the second patch above. `record_transfer_proof` is the single point every recorder passes through, it already establishes the convention of returning `false` for a deliberately dropped credit so weight reconciliation stays honest, and it covers the `ReserveRepatriated` arm and any future recorder at the same time. Patching the `TransferOnHold` match arm alone is not sufficient, as the run above shows. If you would rather also fix the pallet locally, have `cancel_transfer` call `release` instead of `transfer_on_hold` when `recipient == pending.from`, which is what that branch means, and have `do_execute_transfer` record on the inner dispatch's actual moved amount rather than on `post_info.is_ok()`.
