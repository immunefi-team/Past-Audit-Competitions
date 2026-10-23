# 90220 bc medium dilithium signature verification is undercharged by stale base extrinsic weight

**Submitted on Aug 23rd 2026 at 19:48:30 UTC by @abarbatei for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #90220
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Inaccuracies to the weight (similar to gas) accounting system that under or overcharge by more than 50% of the correct amount (and are not intentional as indicated by the program's terms)

## Description

### Brief/Intro

Quantus accepts signed transactions authenticated with ML-DSA Dilithium signatures, but the checked-in base extrinsic weight is still a generic Substrate no-op benchmark value of 108,157 ns. A valid ML-DSA-87 signed transaction performs account derivation and signature verification before dispatch while block-weight and fee accounting reserve only the stale base amount plus normal call and length components. The runnable proof measured the runtime-like ML-DSA-87 authentication cost at 264,126 ns, so the base component undercharges this mandatory work by 59.05% of the correct cost.

### Vulnerability Details

The intended behavior is that each signed extrinsic reserves and pays for the compute needed to authenticate it. Otherwise a block can contain more verification work than the block weight model says it contains, and transaction fees no longer reflect the actual per-transaction cost.

Quantus changes the runtime signature type from the usual Substrate signature model to Dilithium. The runtime aliases at [`lib.rs#L111-L183`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/lib.rs#L111-L183) make `DilithiumSignatureScheme` the `Signature`, derive `AccountId` from that signature scheme, include `CheckWeight` in `TxExtension`, and use those types for `UncheckedExtrinsic` and `SignedPayload`. The accepted signature enum includes both ML-DSA-87 and ML-DSA-65 variants at [`types.rs#L51-L74`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/dilithium-crypto/src/types.rs#L51-L74).

Signature verification is not a cheap constant check. `DilithiumSignatureScheme::verify` derives the expected account from the public key carried in the signature and then calls the selected ML-DSA verifier.

[`DilithiumSignatureScheme::verify` lines 160-183](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/dilithium-crypto/src/traits.rs#L160-L183)

```rust
match self {
	Self::Dilithium87(sig_public) => {
		let account = sig_public.public().clone().into_account();
		if account != *signer {
			return false;
		}
		crate::verify_ml_dsa_87(
			sig_public.public().as_ref(),
			msg.get(),
			sig_public.signature().as_ref(),
		)
	},
	Self::Dilithium65(sig_public) => {
		let account = sig_public.public().clone().into_account();
		if account != *signer {
			return false;
		}
		crate::verify_ml_dsa_65(
			sig_public.public().as_ref(),
			msg.get(),
			sig_public.signature().as_ref(),
		)
	},
}
```

The account derivation used by that check hashes the Dilithium public key at [`DilithiumSigner::into_account` lines 194-199](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/dilithium-crypto/src/traits.rs#L194-L199), so the authentication path includes both account derivation and the selected ML-DSA verification.

The same check is reached before dispatch during block import. `Executive::do_apply_extrinsic` decodes the unchecked extrinsic, verifies the signature through `check`, then obtains dispatch info and applies the already checked extrinsic.

[`Executive::do_apply_extrinsic` lines 883-899](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/frame/executive/src/lib.rs#L883-L899)

```rust
// Verify that the signature is good.
let xt = check(uxt, &Context::default())?;

let dispatch_info = xt.get_dispatch_info();

if !is_inherent && !<frame_system::Pallet<System>>::inherents_applied() {
	Self::inherents_applied();
}

// We don't need to make sure to `note_extrinsic` only after we know it's going to be
// executed to prevent it from leaking in storage since at this point, it will either
// execute or panic (and revert storage changes).
<frame_system::Pallet<System>>::note_extrinsic(encoded);

// AUDIT: Under no circumstances may this function panic from here onwards.

let r = Applyable::apply::<UnsignedValidator>(xt, &dispatch_info, encoded_len)?;
```

Transaction-pool validation also performs the signed check first at [`Executive::validate_transaction` lines 983-1004](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/frame/executive/src/lib.rs#L983-L1004). This makes the work attacker reachable through ordinary valid signed transactions.

The block-weight path does not add a Dilithium-specific verification charge. `CheckWeight` accounts for the dispatch info, the configured `base_extrinsic`, and encoded length as proof size.

[calculate\_consumed\_weight lines 162-166](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/frame-system/src/extensions/check_weight.rs#L162-L166)

```rust
// Also Consider extrinsic length as proof weight.
let extrinsic_weight = info
	.total_weight()
	.saturating_add(maximum_weight.get(info.class).base_extrinsic)
	.saturating_add(Weight::from_parts(0, len as u64));
```

That configured base is the problem. The checked-in value was generated by a 2024 generic Substrate overhead benchmark and sets the no-op extrinsic overhead to 108,157 ns.

[`extrinsic_weights.rs#L18-L56`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/frame/support/src/weights/extrinsic_weights.rs#L18-L56)

```rust
//! THIS FILE WAS AUTO-GENERATED USING THE SUBSTRATE BENCHMARK CLI VERSION 32.0.0
//! DATE: 2024-11-08 (Y/M/D)
//! HOSTNAME: `runner-wiukf8gn-project-674-concurrent-0`, CPU: `Intel(R) Xeon(R) CPU @ 2.60GHz`
//!
//! SHORT-NAME: `extrinsic`, LONG-NAME: `ExtrinsicBase`, RUNTIME: `Development`
//!
/// Weight of executing a NO-OP extrinsic, for example `System::remark`.
/// Calculated by multiplying the *Average* with `1.0` and adding `0`.
///
/// Stats nanoseconds:
///   Min, Max: 107_464, 109_127
///   Average:  108_157
pub const ExtrinsicBaseWeight: Weight =
	Weight::from_parts(WEIGHT_REF_TIME_PER_NANOS.saturating_mul(108_157), 0);
```

`BlockWeights::builder` installs `constants::ExtrinsicBaseWeight::get()` as each non-mandatory class's `base_extrinsic` at [`BlockWeights::builder` lines 343-355](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/frame-system/src/limits.rs#L343-L355), and the Quantus runtime uses `BlockWeights::with_sensible_defaults` at [configs/mod.rs#L80-L95](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L80-L95). Fees use the same base value through `T::BlockWeights::get().get(class).base_extrinsic` at [`Pallet::compute_fee_raw` lines 668-680](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/transaction-payment/src/lib.rs#L668-L680), with a 1:1 compute-weight fee mapping configured at [configs/mod.rs#L477-L486](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L477-L486).

A focused timing proof measured the runtime-like ML-DSA-87 authentication component at 264,126 ns. Compared with the configured 108,157 ns base, the uncharged portion is 155,969 ns. Using the correct component as the denominator, the undercharge is 155,969 / 264,126 = 59.05%. This already exceeds the program's 50% threshold before including any remaining non-signature extrinsic overhead.

There is useful counterevidence: the current benchmark builder creates Dilithium87 signed `system.remark` extrinsics at [`RemarkBuilder::build` lines 42-53](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/node/src/benchmarking.rs#L42-L53) and [create\_benchmark\_extrinsic lines 99-161](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/node/src/benchmarking.rs#L99-L161). Regenerating and checking in the overhead artifact may therefore be enough to fix this. The audited commit still ships the stale 108,157 ns artifact, so the live configured accounting is under the measured ML-DSA-87 authentication cost.

### Impact Details

An untrusted user can create valid ML-DSA-87 signed low-dispatch-cost extrinsics and submit many of them. Each included extrinsic forces importing nodes to perform Dilithium public-key account derivation and ML-DSA-87 signature verification, but the chain only accounts for the stale generic base, normal dispatch weight, and length component. This causes block-weight and transaction-fee under-accounting for each such transaction.

The impact is medium: "Inaccuracies to the weight (similar to gas) accounting system that under or overcharge by more than 50% of the correct amount (and are not intentional as indicated by the program's terms)." The measured ML-DSA-87 runtime-like authentication component is undercharged by 59.05% of its correct cost.

This should not be overstated as proven network shutdown. Block length is capped and the available evidence proves the weight and fee inaccuracy, not a separate total liveness failure.

### Mitigation

Regenerate `ExtrinsicBaseWeight` for the Quantus runtime and checked-in node at the audited configuration, using the Dilithium87 signed no-op extrinsic path. Check in the regenerated `frame/support/src/weights/extrinsic_weights.rs` value and make the overhead benchmark part of release CI whenever the runtime signature type, transaction extension set, or signature verification implementation changes.

If the base extrinsic weight is not the desired accounting location, add an explicit signed-extrinsic authentication weight that covers the worst accepted signature variant, including public-key account derivation and ML-DSA verification, and make sure both block-weight admission and transaction-payment fees use it.

## References

* Audited chain commit [3b243b870a5442a6e5f443056f67bbba87d472f5](https://github.com/immunefi-team/audit-comp-quantus-chain/commit/3b243b870a5442a6e5f443056f67bbba87d472f5).
* Runtime signed extrinsic aliases: [`lib.rs#L111-L183`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/lib.rs#L111-L183).
* Accepted Dilithium signature variants: [`types.rs#L51-L74`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/dilithium-crypto/src/types.rs#L51-L74).
* Dilithium verification path: [`DilithiumSignatureScheme::verify`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/dilithium-crypto/src/traits.rs#L155-L184).
* Block import signed check: [`Executive::do_apply_extrinsic`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/frame/executive/src/lib.rs#L863-L912).
* Transaction-pool signed check: [`Executive::validate_transaction`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/frame/executive/src/lib.rs#L963-L1004).
* Block-weight accounting formula: [calculate\_consumed\_weight](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/frame-system/src/extensions/check_weight.rs#L153-L220).
* Checked-in base extrinsic artifact: [`extrinsic_weights.rs#L18-L56`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/frame/support/src/weights/extrinsic_weights.rs#L18-L56).
* Fee base uses the same `base_extrinsic`: [`Pallet::compute_fee_raw` lines 668-680](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/transaction-payment/src/lib.rs#L668-L680).
* Current benchmark builder uses Dilithium87 signed remarks: [create\_benchmark\_extrinsic](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/node/src/benchmarking.rs#L99-L161).

## Proof of Concept

### Setup Assumptions

* The audited chain repository is checked out at commit [3b243b870a5442a6e5f443056f67bbba87d472f5](https://github.com/immunefi-team/audit-comp-quantus-chain/commit/3b243b870a5442a6e5f443056f67bbba87d472f5).
* The proof files are installed into `runtime/tests/poc_dilithium_base_weight_undercharge/` for the run, with `offline-vendor/` adjacent to `Cargo.toml`.
* The proof uses local path dependencies for the in-scope `audit-comp-quantus-qp-poseidon` and `audit-comp-quantus-qp-rusty-crystals` repositories. The offline run below uses the vendored registry patches in `offline-vendor/` so Cargo does not need network access.

### Execution Command

```bash
env CARGO_NET_OFFLINE=true cargo +stable test --release --manifest-path runtime/tests/poc_dilithium_base_weight_undercharge/Cargo.toml -- --nocapture ml_dsa_87_authentication_base_weight_undercharges_by_more_than_half
```

### Proof Manifest

```toml
[package]
name = "proof-dilithium-base-weight-undercharge"
version = "0.1.0"
edition = "2021"
publish = false

[workspace]

[dependencies]
qp-poseidon-core = { path = "../../../../audit-comp-quantus-qp-poseidon", default-features = false }
qp-rusty-crystals-dilithium = { path = "../../../../audit-comp-quantus-qp-rusty-crystals/dilithium", default-features = false, features = ["ml-dsa-87"] }

[[test]]
name = "proof_dilithium_base_weight_undercharge"
path = "proof_dilithium_base_weight_undercharge.rs"

[patch.crates-io]
proc-macro2 = { path = "offline-vendor/proc-macro2-1.0.107" }
quote = { path = "offline-vendor/quote-1.0.47" }
syn = { path = "offline-vendor/syn-2.0.119" }
unicode-ident = { path = "offline-vendor/unicode-ident-1.0.24" }
zeroize = { path = "offline-vendor/zeroize-1.8.2" }
zeroize_derive = { path = "offline-vendor/zeroize_derive-1.5.0" }
```

### Proof Source

```rust
use std::{hint::black_box, time::Instant};

use qp_rusty_crystals_dilithium::ml_dsa_87;

const WARMUP_ITERATIONS: u32 = 50;
const MEASURED_ITERATIONS: u32 = 1_000;
const EXPECTED_STALE_BASE_NS: u128 = 108_157;
const UNDERCHARGE_THRESHOLD_BPS: u128 = 5_000;

fn audited_base_extrinsic_ns() -> u128 {
	let generated_weight_source =
		include_str!("../../../frame/support/src/weights/extrinsic_weights.rs");

	assert!(
		generated_weight_source.contains("DATE: 2024-11-08"),
		"the proof is expected to run against the audited generated base-weight artifact"
	);
	assert!(
		generated_weight_source.contains("saturating_mul(108_157)"),
		"the proof is expected to run against the stale 108,157 ns base-extrinsic value"
	);

	EXPECTED_STALE_BASE_NS
}

fn verify_average_ns(
	public_key_bytes: &[u8],
	signature_bytes: &[u8],
	message: &[u8],
	expected_account: &[u8; 32],
	iterations: u32,
) -> u128 {
	let mut accepted = 0u32;
	let started = Instant::now();

	for _ in 0..iterations {
		let computed_account = qp_poseidon_core::hash_bytes(black_box(public_key_bytes));
		let public_key = ml_dsa_87::PublicKey::from_bytes(black_box(public_key_bytes))
			.expect("the measured public key is generated by the target ML-DSA-87 implementation");

		if black_box(&computed_account == expected_account)
			&& black_box(public_key.verify(
				black_box(message),
				black_box(signature_bytes),
				None,
			)) {
			accepted += 1;
		}
	}

	assert_eq!(
		accepted, iterations,
		"the measured path must be the valid ML-DSA-87 signed-extrinsic authentication path"
	);
	started.elapsed().as_nanos() / u128::from(iterations)
}

#[test]
fn ml_dsa_87_authentication_base_weight_undercharges_by_more_than_half() {
	let keypair = ml_dsa_87::Keypair::generate(&mut (&mut [7u8; 32]).into());
	let message = b"low-dispatch-cost signed extrinsic payload";
	let signature = keypair
		.sign(message, None, None)
		.expect("target ML-DSA-87 signing should succeed");
	let public_key_bytes = keypair.public().to_bytes();
	let expected_account = qp_poseidon_core::hash_bytes(public_key_bytes.as_ref());

	for _ in 0..WARMUP_ITERATIONS {
		let public_key = ml_dsa_87::PublicKey::from_bytes(public_key_bytes.as_ref())
			.expect("generated public key should parse");
		assert_eq!(
			qp_poseidon_core::hash_bytes(public_key_bytes.as_ref()),
			expected_account
		);
		assert!(public_key.verify(message, signature.as_slice(), None));
	}

	let configured_base_ns = audited_base_extrinsic_ns();
	assert_eq!(
		configured_base_ns, EXPECTED_STALE_BASE_NS,
		"the audited runtime is still using the stale generated base extrinsic weight"
	);

	let runtime_authentication_ns = verify_average_ns(
		public_key_bytes.as_ref(),
		signature.as_slice(),
		message,
		&expected_account,
		MEASURED_ITERATIONS,
	);
	let undercharged_ns = runtime_authentication_ns.saturating_sub(configured_base_ns);
	let undercharge_bps = undercharged_ns * 10_000 / runtime_authentication_ns;

	println!("iterations={MEASURED_ITERATIONS}");
	println!("configured_base_extrinsic_ns={configured_base_ns}");
	println!("runtime_like_ml_dsa_87_authentication_avg_ns={runtime_authentication_ns}");
	println!("undercharged_ns={undercharged_ns}");
	println!("undercharge_percent={:.2}", undercharge_bps as f64 / 100.0);

	assert!(
		undercharge_bps > UNDERCHARGE_THRESHOLD_BPS,
		"ML-DSA-87 authentication is undercharged by only {:.2}%; expected more than 50%",
		undercharge_bps as f64 / 100.0
	);
}
```

### Observed Proof Output

```
running 1 test
iterations=1000
configured_base_extrinsic_ns=108157
runtime_like_ml_dsa_87_authentication_avg_ns=264126
undercharged_ns=155969
undercharge_percent=59.05
test ml_dsa_87_authentication_base_weight_undercharges_by_more_than_half ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.29s
```

The final assertion first confirms that the checked-in base-extrinsic artifact is the stale `108_157 ns` value, then asserts that the valid ML-DSA-87 authentication path is undercharged by more than `50%` of the measured correct cost. In the recorded run, the measured authentication cost was `264_126 ns`; the uncharged portion was `155_969 ns`, which is `59.05%` of the measured cost.
