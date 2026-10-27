# 89098 bc low multisig execute wraps transfers at zero proof recording weight

## #89098 \[BC-Low] Multisig execute wraps transfers at zero proof-recording weight

**Submitted on Aug 17th 2026 at 06:30:16 UTC by @AlexSR for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89098
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Inaccuracies to the weight (similar to gas) accounting system that under or overcharge by more than 50% of the correct amount (and are not intentional as indicated by the program's terms)

### Description

## Multisig execute wraps transfers at zero proof-recording weight

### Summary

Every recorded wormhole transfer proof costs the runtime a ZK-tree leaf insert, a `TransferCount` read/write and two event deposits. `WormholeProofRecorderExtension` prices that work statically, per transfer, at 4.875 ms of `ref_time` (`runtime/src/transaction_extensions.rs:149-159`); that price is what `CheckWeight` admits against and what `ChargeTransactionPayment` bills. The matcher that produces the transfer count, `count_transfers`, recognises direct `Balances` transfers and the transparent `Utility`/`Recovery` wrappers but has no `RuntimeCall::Multisig` arm, so `Multisig::execute` falls through to `_ => 0` (`:217-293`) and `weight()` returns `Weight::zero()` for it (`:414-421`). The dispatch does the full recorder work anyway: the post-dispatch scan finds every transfer the stored inner call emitted and inserts a leaf for each (`:450-473`, recording at `:383-394`). The shortfall is handed afterwards to `register_extra_weight_unchecked` (`:508-517`) on a path the runtime itself calls "not fee-charged" (`:475-477`).

One piece of work, two prices. A `Utility::batch` of eight transfers is charged eight per-transfer units at admission and pays for them; the same eight transfers, stored in a multisig proposal and released by `Multisig::execute`, are charged nothing while the recorder performs the same eight inserts. This is not only an attacker's path: an ordinary one-transfer multisig payment is charged about 0.46 ms against a correct 5.34 ms, an undercharge of 91%, every time.

The behaviour is deliberate and the source says so (`:224-227`, `:489-507`, test at `:1247-1285`). It is filed anyway because the weight rungs' carve-out is keyed to "the program's terms", and the programme page does not disclose it. That argument is in the Severity section.

### Root cause

`count_transfers` is a static matcher over the submitted call. Its arms cover every wrapper whose inner calls travel inside the extrinsic: the three `Utility` batch variants recurse over `calls`, and `dispatch_as`, `with_weight`, `as_derivative`, `Recovery::as_recovered` and `if_else` recurse into their boxed inner call (`transaction_extensions.rs:259-276`). `Multisig::execute` carries no inner call; it carries `(multisig_address, proposal_id)` (`pallets/multisig/src/lib.rs:1068-1072`) and later reads the payload from `Proposals` storage (`:1090-1091`, decode at `:1125-1126`). A static call walker cannot see it, so it hits the `_ => 0` fallback (`transaction_extensions.rs:293`). The comment above the match says exactly this and names the call:

> Wrappers whose inner call is stored on-chain rather than in the submitted call (`Multisig::execute`, ...) cannot be counted statically. Proof-recording work they trigger is reconciled in `post_dispatch`, which registers any weight shortfall against the block via `register_extra_weight_unchecked`.

Reconciling in `post_dispatch` fixes the block-weight total and nothing else. It cannot fix the fee: FRAME's post-dispatch correction only refunds downward, `actual_weight` capped at the pre-charged weight, which for this component was zero. It cannot fix admission either. `CheckWeight` does read extension weight, because `weight()` feeds `DispatchInfo::total_weight()`; the problem is that the number `weight()` supplies here is zero, so the figure admission is decided on already omits the work. (`CheckWeight` sits at index 6 of `TxExtension`, the recorder at index 8, `runtime/src/lib.rs:154-176`.)

A static arm was not impossible. `count_transfers` already charges `ReversibleTransfers::recover_funds` a flat `MaxPendingPerAccount + 1` (`:240-252`), accepting an overcharge on a call whose real transfer count lives in on-chain state. `Multisig::execute` has the same shape and an available bound, and no arm; the Suggested fix supplies one.

### Why this is a weight inaccuracy, not a design note

`per_transfer_weight` (`:149-159`) works out to 4.875 ms of `ref_time` (55 reads at 25 us, 26 writes at 100 us, 18 Poseidon evaluations at 50 us; arithmetic in the Validation Steps field) against the 156 us the triggering transfer itself declares (`pallets/balances/src/weights.rs:82-90`). The recorder is 31 times the cost of the call it follows, so it dominates the correct charge rather than rounding it. Transparent path: N transfers are admitted and billed `N * 4.875 ms`. Opaque path: the same N transfers are billed zero, and `N * 4.875 ms` lands on `BlockWeight` afterwards through a function documented not to enforce the limit (`pallets/frame-system/src/lib.rs:1952-1953`). That eventual correction never reaches the fee.

### Attack path

Every step is a signed extrinsic from attacker-funded accounts: no privilege, no leaked key, no governance action. Costs and limits are in the Validation Steps field.

{% stepper %}
{% step %}

### Create the multisig

`create_multisig([A, B], threshold = 1, nonce)`, both accounts attacker-controlled (`pallets/multisig/src/lib.rs:426-501`; two distinct signers is the pallet minimum, `:436`).
{% endstep %}

{% step %}

### Propose the stored batch

`propose(multisig, encoded_call, expiry)` with `encoded_call` a `Utility::batch` of transfers filling the 10240-byte bound. At `threshold = 1` the proposal is stored `Approved` immediately (`:651,686-691`).
{% endstep %}

{% step %}

### Execute the proposal

`execute(multisig, proposal_id)`. `count_transfers` returns 0 for `RuntimeCall::Multisig(execute { .. })`, so no proof-recording weight is declared, admitted or billed. The stored batch dispatches, the transfers emit their events, the recorder scans and inserts every leaf, and only then is the work handed to `register_extra_weight_unchecked`.
{% endstep %}

{% step %}

### Repeat

Proposals are capped per multisig (`:565-582`); nothing caps multisig count.
{% endstep %}
{% endstepper %}

### Impact

Per opaque call the recorder performs about 1.131 s of `ref_time` of proof-recording work that carries no declared weight: a 10240-byte stored call holds 232 transfers (measured, see Validation Steps). Two consequences follow, neither a halt.

**An undercharge on every multisig-dispatched transfer.** The recorder's work on this path is never billed, while the identical work submitted transparently is. On an ordinary one-transfer multisig payment the missing amount is 91% of the correct charge; on a full proposal it is about 97%.

**A bounded admission overshoot, smaller than that shortfall.** `Multisig::execute` declares its own worst case as bookkeeping plus `MaxInnerCallWeight`, one second of `ref_time` (`pallets/multisig/src/lib.rs:1059-1066`), so admission does reserve about 1.0 s for the extrinsic. `CheckWeight`'s post-dispatch reclaim hands back the unused part of that reservation, about 0.96 s for a 232-transfer batch whose calls declare about 36 ms, before the recorder later in the same tuple accrues its 1.131 s. The block therefore finishes about 0.17 s above the figure that admitted the extrinsic, against a Normal-class ceiling of 4.5 s (6 s `max_block` at a 75% Normal ratio, `runtime/src/configs/mod.rs:72,86-89`): under a twentieth of the class budget, bounded by one extrinsic.

No funds are lost, frozen or misdirected; the recorded proofs and ZK-tree state are correct; no chain split, no stall.

### Severity self-assessment

Filed at **Medium**, on this rung, quoted from the programme's Blockchain/DLT ladder:

> Inaccuracies to the weight (similar to gas) accounting system that under or overcharge by more than 50% of the correct amount (and are not intentional as indicated by the program's terms)

**Why over 50%.** The correct compute charge for an executed proposal is multisig bookkeeping plus the inner call's actual weight plus the recorder's per-transfer work. The collected charge is the first two only. For one transfer that is 0.46 ms collected against 5.34 ms correct, 91% missing; for a full 10240-byte proposal, about 36 ms against about 1.17 s, 97% missing. Neither figure depends on an attacker.

### Proof of Concept

#### Arithmetic

`per_transfer_weight` (`transaction_extensions.rs:149-159`) resolves, at `CIRCUIT_MAX_TREE_DEPTH = 16` with `INSERT_LEAF_DB_OPS = (3d + 4, d + 5)` and `d + 2` Poseidon evaluations (`pallets/zk-tree/src/lib.rs:68-99`), to 55 reads, 26 writes and 18 evaluations. At RocksDB 25 us/read, 100 us/write and 50 us/evaluation: 1.375 + 2.600 + 0.900 = **4.875 ms**. A `transfer_keep_alive` declares 31 us + 1 read + 1 write = **156 us**. `WeightInfo::execute(10240)` is 29.3 us + 517 ps/byte + 3 reads + 2 writes = **0.31 ms** of bookkeeping. One transfer: correct 0.31 + 0.156 + 4.875 = 5.34 ms against 0.46 ms collected, **91% missing**. A 10240-byte proposal holds 232 transfers (measured): correct 0.31 + 36.2 + 1131 = 1168 ms against 36.5 ms collected, **97% missing**; net of the 1.0 s admission reserved, the block overshoot is **\~0.17 s**.

#### What the PoC proves vs. stubs (executed green)

| Claim                                                                                                                                                                                                                                                                                               | Status                                                                                                                                                                                      |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `count_transfers` and `weight()` return zero for `Multisig::execute`, while the same transfers as `Utility::batch` cost `per_transfer_weight * n`                                                                                                                                                   | Executed green (test 1)                                                                                                                                                                     |
| Proofs are still recorded, declared extension weight is zero, and the recorder's cost reaches the block only post-dispatch                                                                                                                                                                          | Executed green (test 2)                                                                                                                                                                     |
| Transfers per stored call and the unpriced `ref_time` it implies                                                                                                                                                                                                                                    | Executed green (test 3); measured 232 transfers, 1.131 s                                                                                                                                    |
| The fix compiles and the patch applies                                                                                                                                                                                                                                                              | Authored (`count_multisig_execute_worst_case.patch`)                                                                                                                                        |
| `CheckWeight` admits on the understated number; the recorder work is never fee-charged; `register_extra_weight_unchecked` does not enforce the block limit                                                                                                                                          | Argued from source (`runtime/src/lib.rs:154-176`; `transaction_extensions.rs:475-477`; `pallets/frame-system/src/lib.rs:1952-1953`; FRAME's post-dispatch correction refunds only downward) |
| STUB: the harness drives the extension alone, running neither `CheckWeight` nor `ChargeTransactionPayment`, so it does not model `Multisig::execute`'s `MaxInnerCallWeight` reservation or its reclaim. Test 2's `BlockWeight` delta is the recorder's contribution, not the extrinsic's net effect | Stubbed by design                                                                                                                                                                           |
| The percentages, the \~0.17 s overshoot, the `ProposalFee` comparison                                                                                                                                                                                                                               | Arithmetic over cited constants                                                                                                                                                             |

### Suggested fix

Give the opaque wrapper what `recover_funds` already gets: a static worst case, refunded when it overshoots. One arm plus one constant, both in `count_multisig_execute_worst_case.patch` (attached).

```rust
RuntimeCall::Multisig(pallet_multisig::Call::execute { .. }) =>
    u64::from(<Runtime as pallet_multisig::Config>::MaxCallSize::get())
        .saturating_div(Self::MIN_RECORDED_TRANSFER_CALL_BYTES),
```

`MIN_RECORDED_TRANSFER_CALL_BYTES = 36` is the smallest encoded call that makes the scan record one proof: pallet index, call index, `MultiAddress::Id` prefix, 32-byte account, one-byte compact amount. No new machinery is needed, because `post_dispatch_details` already returns `per_transfer_weight * (charged - recorded)` as unspent weight (`transaction_extensions.rs:519-526`), so an honest one-transfer payment reserves the worst case and pays only for what it used. Confirm the constant against test 3 before adopting it.

```rust
// Drop-in unit tests for the runtime's own `runtime/src/transaction_extensions.rs`,
// inside its existing `#[cfg(test)] mod tests` block.
//
// They reuse that module's own helpers and nothing else: `new_test_ext`, `alice`,
// `bob`, `run_lifecycle_with_result`, and the multisig setup pattern already used by
// `event_based_proof_recording_multisig_transfer` (create_multisig -> fund -> propose
// -> execute, with `pallet_multisig::Pallet::<Runtime>::derive_multisig_address`).
//
// INSERT POSITION: immediately before the module's closing brace (the last line of the
// file at the pinned commit, line 2201). See RUN.md for the exact command.
//
// Run:
//   cargo test -p quantus-runtime --lib transaction_extensions::tests::poc_multisig -- --nocapture
//
// EXECUTION STATUS: executed green (3 passed, 2026-08-16). See POC_EXECUTED_GREEN.md.
//
// WHAT THESE TESTS PROVE
//   - `count_transfers` returns 0 and `weight()` returns `Weight::zero()` for
//     `RuntimeCall::Multisig(execute { .. })`, while the byte-identical set of
//     transfers presented as `Utility::batch` is counted and charged
//     `per_transfer_weight * n`. Same work, two prices.
//   - Driven through the extension lifecycle, `Multisig::execute` still records a
//     proof for every inner transfer, and the whole per-transfer cost plus the event
//     scan appears in `BlockWeight` only AFTER dispatch, via
//     `register_extra_weight_unchecked`, with the extension's declared (fee-bearing)
//     weight left at zero.
//
// WHAT THESE TESTS STUB / DO NOT PROVE (argued from source in the report instead)
//   - The `CheckWeight` admission decision itself: these tests call the extension
//     directly, they do not run the full `TxExtension` tuple, so the "admitted on a
//     number that omits this work" step is read off `runtime/src/lib.rs:154-176`
//     (CheckWeight at index 6, the recorder at index 8), not executed here.
//   - `Multisig::execute`'s OWN weight annotation, which declares bookkeeping plus
//     `MaxInnerCallWeight` (1 s of ref_time, `pallets/multisig/src/lib.rs:1059-1066`)
//     and refunds the unused part post-dispatch. The harness builds its own
//     `DispatchInfo`, so test 2's `BlockWeight` delta below is the RECORDER's
//     contribution alone, not the extrinsic's net effect on the block.
//   - The fee actually charged: `ChargeTransactionPayment` is not run. What is shown
//     is the input it would see, `DispatchInfo::extension_weight == Weight::zero()`.
//   - The 10240-byte worst case (232 transfers, measured): these tests use 8 transfers so they
//     stay fast and readable. `poc_multisig_execute_worst_case_transfer_count` measures
//     the encoded size of one transfer so a reviewer can compute the real bound.
//   - Any economic claim (ProposalFee vs unpriced work). That is arithmetic over
//     configured constants, not something these tests measure.

/// Eight distinct fresh destinations, so every transfer records its own proof.
fn poc_dest(i: u8) -> AccountId {
	AccountId32::from([100u8 + i; 32])
}

fn poc_transfers(n: u8) -> alloc::vec::Vec<RuntimeCall> {
	(0..n)
		.map(|i| {
			RuntimeCall::Balances(pallet_balances::Call::transfer_keep_alive {
				dest: MultiAddress::Id(poc_dest(i)),
				value: EXISTENTIAL_DEPOSIT * 2,
			})
		})
		.collect()
}

/// The static half of the defect: identical work, two prices.
///
/// @audit `count_transfers` has no `RuntimeCall::Multisig` arm
/// (`transaction_extensions.rs:217-293`), so the opaque wrapper falls through to
/// `_ => 0` and `weight()` returns zero (`:414-421`).
#[test]
fn poc_multisig_execute_is_zero_weighted_while_the_same_batch_is_charged() {
	new_test_ext().execute_with(|| {
		let ext = WormholeProofRecorderExtension::<Runtime>::new();
		let per_transfer = WormholeProofRecorderExtension::<Runtime>::per_transfer_weight();

		// Presented transparently, the eight transfers are counted and priced.
		let transparent =
			RuntimeCall::Utility(pallet_utility::Call::batch { calls: poc_transfers(8) });
		assert_eq!(
			WormholeProofRecorderExtension::<Runtime>::count_transfers(&transparent),
			8,
			"a transparent batch must be statically counted"
		);
		assert_eq!(
			<WormholeProofRecorderExtension<Runtime> as TransactionExtension<RuntimeCall>>::weight(
				&ext,
				&transparent
			),
			per_transfer.saturating_mul(8),
			"a transparent batch must reserve the full per-transfer price"
		);

		// A Multisig::execute call, by contrast, has zero proof-recorder extension
		// weight: it carries only (multisig_address, proposal_id), so the static
		// matcher sees no inner transfers and returns 0 (the call itself still
		// declares execute + MaxInnerCallWeight; test 2 stores and dispatches the eight).
		let opaque = RuntimeCall::Multisig(pallet_multisig::Call::execute {
			multisig_address: alice(),
			proposal_id: 0,
		});
		assert_eq!(
			WormholeProofRecorderExtension::<Runtime>::count_transfers(&opaque),
			0,
			"@audit THE DEFECT: count_transfers has no RuntimeCall::Multisig arm"
		);
		assert_eq!(
			<WormholeProofRecorderExtension<Runtime> as TransactionExtension<RuntimeCall>>::weight(
				&ext,
				&opaque
			),
			Weight::zero(),
			"@audit THE DEFECT: the wrapper reserves no proof-recording weight at all"
		);
	});
}

/// The dynamic half: the work is done, recorded, and only then charged to the block,
/// while the fee-bearing extension weight stays at zero.
#[test]
fn poc_multisig_execute_recorder_work_is_registered_after_the_fact() {
	use codec::Encode;

	new_test_ext().execute_with(|| {
		System::set_block_number(1);

		// A 1-of-2 multisig. Both signers are attacker-controlled; `threshold = 1` is
		// accepted (`pallets/multisig/src/lib.rs:435,453`) and makes `propose` alone
		// leave the proposal `Approved` (`:651,686-691`), so no second party is needed.
		let signers = vec![alice(), bob()];
		let threshold = 1u32;
		let nonce = 0u64;
		assert_ok!(Multisig::create_multisig(
			RuntimeOrigin::signed(alice()),
			signers.clone(),
			threshold,
			nonce,
		));
		let multisig_address =
			pallet_multisig::Pallet::<Runtime>::derive_multisig_address(&signers, threshold, nonce);
		assert_ok!(Balances::transfer_keep_alive(
			RuntimeOrigin::signed(alice()),
			MultiAddress::Id(multisig_address.clone()),
			EXISTENTIAL_DEPOSIT * 1000,
		));

		// Store the eight transfers on-chain as one proposal. From here the submitted
		// call carries only (multisig_address, proposal_id).
		let inner = RuntimeCall::Utility(pallet_utility::Call::batch { calls: poc_transfers(8) });
		let encoded: pallet_multisig::BoundedCallOf<Runtime> =
			inner.encode().try_into().expect("inner batch is well under MaxCallSize");
		let expiry = System::block_number() + 100;
		assert_ok!(Multisig::propose(
			RuntimeOrigin::signed(alice()),
			multisig_address.clone(),
			encoded,
			expiry,
		));

		let opaque = RuntimeCall::Multisig(pallet_multisig::Call::execute {
			multisig_address: multisig_address.clone(),
			proposal_id: 0,
		});

		let weight_before = frame_system::Pallet::<Runtime>::block_weight().total();
		let scanned = core::cell::Cell::new(0u32);
		let scanned_bytes = core::cell::Cell::new(0u32);

		let (info, _post_info) = run_lifecycle_with_result(&alice(), opaque, Ok(()), || {
			assert_ok!(Multisig::execute(
				RuntimeOrigin::signed(alice()),
				multisig_address.clone(),
				0,
			));
			scanned.set(frame_system::Pallet::<Runtime>::event_count());
			scanned_bytes.set(frame_system::Pallet::<Runtime>::event_bytes());
		});

		// @audit (1) Everything downstream of `weight()` saw zero proof-recording weight
		//     for this call. In production that value is the recorder's contribution to
		//     `DispatchInfo::total_weight()`, the number `CheckWeight` admits against and
		//     the number `ChargeTransactionPayment` bills.
		assert_eq!(
			info.extension_weight,
			Weight::zero(),
			"@audit THE DEFECT: admission and the fee see no proof-recording weight"
		);

		// @audit (2) The work happened anyway: eight proofs recorded, one per destination.
		for i in 0u8..8 {
			assert_eq!(
				Wormhole::transfer_count(&poc_dest(i)),
				1,
				"every inner transfer must still be recorded as a proof"
			);
		}

		// @audit (3) And its full cost only reached the block afterwards, through
		//     `register_extra_weight_unchecked` (`transaction_extensions.rs:508-517`).
		let per_transfer = WormholeProofRecorderExtension::<Runtime>::per_transfer_weight();
		let weight_after = frame_system::Pallet::<Runtime>::block_weight().total();
		assert_eq!(
			weight_after.saturating_sub(weight_before),
			per_transfer.saturating_mul(8).saturating_add(
				WormholeProofRecorderExtension::<Runtime>::event_scan_weight(
					scanned.get(),
					scanned_bytes.get()
				)
			),
			"the entire per-transfer cost was registered post-dispatch, none of it admitted"
		);
	});
}

/// Measurement aid, not an assertion of the defect: prints the encoded size of one
/// transfer call so a reviewer can compute the real worst case for a stored call of
/// `MaxCallSize = 10240` bytes (`runtime/src/configs/mod.rs:648`). This test measures the
/// bound; the report carries the measured 232-transfer value.
#[test]
fn poc_multisig_execute_worst_case_transfer_count() {
	use codec::Encode;

	new_test_ext().execute_with(|| {
		let one = RuntimeCall::Balances(pallet_balances::Call::transfer_keep_alive {
			dest: MultiAddress::Id(poc_dest(0)),
			value: EXISTENTIAL_DEPOSIT,
		});
		let one_len = one.encode().len();

		let batch_of_two =
			RuntimeCall::Utility(pallet_utility::Call::batch { calls: poc_transfers(2) });
		let two_len = batch_of_two.encode().len();

		let max_call_size = <Runtime as pallet_multisig::Config>::MaxCallSize::get() as usize;
		let per_transfer_weight = WormholeProofRecorderExtension::<Runtime>::per_transfer_weight();

		// Marginal bytes per additional transfer, then the bound it implies.
		let marginal = two_len.saturating_sub(one_len);
		let bound = max_call_size / marginal.max(1);

		println!("encoded transfer_keep_alive          = {one_len} bytes");
		println!("marginal bytes per extra transfer    = {marginal}");
		println!("MaxCallSize                          = {max_call_size} bytes");
		println!("=> transfers per stored call (bound) = {bound}");
		println!("per_transfer_weight ref_time (ps)    = {}", per_transfer_weight.ref_time());
		println!(
			"=> unpriced ref_time per opaque call = {} ps",
			per_transfer_weight.ref_time().saturating_mul(bound as u64)
		);

		assert!(bound > 0, "a stored call must fit at least one transfer");
	});
}
```
