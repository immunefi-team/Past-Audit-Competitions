# 88855 bc medium base extrinsic is substrate s stock sr25519 constant signed extrinsic is undercharged

**Submitted on Aug 15th 2026 at 08:26:46 UTC by @roulette6041 for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88855
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Inaccuracies to the weight (similar to gas) accounting system that under or overcharge by more than 50% of the correct amount (and are not intentional as indicated by the program's terms)

## Description

## Bug Description

The runtime builds its block weights with `BlockWeights::with_sensible_defaults`, which adopts `frame_support`'s stock `ExtrinsicBaseWeight` of 108,157 ns. Parity generated that constant by benchmarking a no-op `System::remark` on an Intel Xeon at 2.60GHz, where the signature is sr25519. Quantus replaced sr25519 with ML-DSA-87 and added a Poseidon hash of the 2592-byte public key for account derivation. The constant was never re-derived.

That work runs inside `UncheckedExtrinsic::check`, which happens before any transaction extension, so `base_extrinsic` is the only weight budget that covers it. Measured on one machine, the signature work is 6.5 times the sr25519 work the adopted constant assumes. Every signed transaction is charged a weight, and a fee derived from that weight, that understates its real cost by more than half.

## Vulnerability Details

{% stepper %}
{% step %}

## The constant is adopted unmodified

[`runtime/src/configs/mod.rs:86-89`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L86-L89):

```rust
pub RuntimeBlockWeights: BlockWeights = BlockWeights::with_sensible_defaults(
    Weight::from_parts(6u64 * WEIGHT_REF_TIME_PER_SECOND, u64::MAX),
    NORMAL_DISPATCH_RATIO,
);
```

`with_sensible_defaults` sets `max_total` and `reserved` per class and leaves `base_extrinsic` at the builder default, which is `frame_support::weights::constants::ExtrinsicBaseWeight`. There is no override anywhere in `runtime/` or `node/`. The PoC reads the value back out of `RuntimeBlockWeights` at runtime and confirms it is 108,157 ns.

The provenance of that number is in the generated header of `frame-support/src/weights/extrinsic_weights.rs`: `CPU: Intel(R) Xeon(R) CPU @ 2.60GHz`, benchmarking `System::remark` on a stock Substrate development chain.
{% endstep %}

{% step %}

## Signature verification is charged to `base_extrinsic` and nothing else

In `frame-executive`, `do_apply_extrinsic` runs the signature check first:

```rust
// Verify that the signature is good.
let xt = check(uxt, &Context::default())?;

let dispatch_info = xt.get_dispatch_info();
```

`check` precedes `get_dispatch_info()` and precedes the whole `TxExtension` pipeline, so it precedes `CheckWeight`. No extension weight can account for it. `base_extrinsic` is the budget.
{% endstep %}

{% step %}

## The work performed there

[`primitives/dilithium-crypto/src/traits.rs:152-185`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/dilithium-crypto/src/traits.rs#L152-L185) derives the account from the public key and then verifies:

```rust
Self::Dilithium87(sig_public) => {
    let account = sig_public.public().clone().into_account();
    if account != *signer { return false; }
    crate::verify_ml_dsa_87(...)
}
```

`into_account` is a Poseidon hash over the full public key ([`traits.rs:194-199`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/dilithium-crypto/src/traits.rs#L191-L200)). Each signed extrinsic pays for one ML-DSA-87 verification plus one Poseidon hash of 2592 bytes.
{% endstep %}

{% step %}

## Measured, same binary, same run

|                                                             | measured      |
| ----------------------------------------------------------- | ------------- |
| sr25519 verify, what the constant assumes                   | 21.90 us      |
| ML-DSA-87 verify                                            | 83.45 us      |
| Poseidon `hash_bytes` over the 2592-byte pubkey             | 58.21 us      |
| **`DilithiumSignatureScheme::verify`, the production call** | **143.33 us** |
| ratio to sr25519                                            | **6.5x**      |

Both figures come from the same binary on the same CPU, so the ratio does not depend on the hardware. The projects' own criterion benches agree independently: `qp-rusty-crystals` reports ML-DSA-87 verify at 82.6 us, and `qp-poseidon` reports `hash_bytes` at 45.35 us for 2048 bytes scaling at about 22.1 ns per byte, which interpolates to roughly 57 us at 2592 bytes.
{% endstep %}

{% step %}

## The undercharge exceeds 50%

Let `s` be the cost of sr25519 verification on the reference hardware the constant was measured on. The stock value decomposes as `108.157 = N + s`, where `N` is everything else in a `remark` extrinsic. The correct Quantus value on that same hardware is `N + 6.5s`, which is `108.157 + 5.5s`. The undercharge exceeds half the correct amount when `5.5s / (108.157 + 5.5s) > 0.5`, which reduces to `s > 19.66 us`.

sr25519 verification measured 21.90 us on an Apple M4. A Xeon at 2.60GHz is not faster than an M4, so `s` on the reference hardware is at least 21.90 us, which is above the 19.66 us threshold. The undercharge is therefore at least **52.7%**, and that figure is a floor. Using a realistic 45 us for sr25519 on that Xeon, the correct base is about 356 us against a declared 108 us, an undercharge of roughly 70%. No step in this derivation compares timings across two machines.
{% endstep %}
{% endstepper %}

## Impact

Every signed transaction is assigned a weight, and a fee derived from that weight, that understates its true execution cost by more than 50%. `base_extrinsic` feeds both block weight accounting in `CheckWeight` and fee computation in `ChargeTransactionPayment`, so the error lands on both.

The Quantus documentation states that ML-DSA verification runs at "\~2-3ms (vs \~0.5ms for ECDSA)" and that per-transaction overhead is "\~7,219 bytes" (<https://docs.quantus.com/deep-dives/pqc/>). The slowdown relative to a classical signature scheme is therefore known and published, while the weight constant that prices it was inherited from the classical scheme unchanged.

This maps to the Medium row in the program's Blockchain/DLT impact list:

> Inaccuracies to the weight (similar to gas) accounting system that under or overcharge by more than 50% of the correct amount (and are not intentional as indicated by the program's terms)

Both conditions in that row hold. The magnitude clears 50% under the floor derivation in section 5. The shortfall is not indicated as intentional in the program terms, the code, or the published documentation, and the last item of the Risk Breakdown gives positive evidence that it is an oversight.

This does not enable a block-production denial of service. A signed Quantus extrinsic carries a 2592-byte public key plus a 4627-byte signature, so roughly 7.3 KB. The normal-class block length limit is 5 MB at 75%, which is 3.93 MB, capping a block at about 537 signed extrinsics. Their real signature cost is roughly 77 ms against a 4,500 ms normal weight budget. The length limit binds about 24 times before the weight limit does, so the accounting error cannot be used to overfill a block. The reported defect is the accounting and fee inaccuracy, which is what the Medium row describes.

## Risk Breakdown

* **Affected component:** `runtime/src/configs/mod.rs` (`RuntimeBlockWeights`), affecting every signed extrinsic through `frame_system::CheckWeight` and `pallet_transaction_payment::ChargeTransactionPayment`.
* **Pre-conditions:** none. This is the default configuration for every signed transaction on the chain.
* **Who is affected:** all users and all block authors. No privileged role and no attacker is required.
* **Funds at risk:** none directly. The consequence is systematic fee underpricing and understated block weight accounting, not a loss of user funds.
* **Evidence that the shortfall is an oversight rather than a deliberate trade-off:** `runtime/src/configs/mod.rs` carries a "Transaction Fee Structure" section recording the project's deliberate weight decisions, including 21 lines of rationale for leaving `proof_size` unpriced plus a "when to revisit this decision" list. `base_extrinsic` appears nowhere in it. The published documentation contains no fees, weights or benchmarking page across its 12-page structure. Commit [`6f7094f`](https://github.com/Quantus-Network/chain/commit/6f7094f0679c89dbb80b575ec63531357c9d763e) ("cache Dilithium pubkeys on chain to enable signature-only transactions") adds a `PubkeyCacheVerifyWeight` surcharge of one database read and one write into `base_extrinsic`, with the stated reason that the work "runs in `UncheckedExtrinsic::check`, before any TxExtension weight/payment handling, so its Pubkeys read (and multi-KB first-use insert) escaped both the block weight limit and fees." That is the mechanism reported here. The roughly 25 us instance was corrected and the roughly 143 us instance in the same function was left in place.

## Recommendation

Stop inheriting the upstream constant and set a Quantus-specific one. `suggested-fix.diff` in the PoC gist replaces the `with_sensible_defaults` call with an explicit builder that keeps the existing `max_total` and `reserved` behaviour and overrides `base_extrinsic` for every dispatch class:

```rust
pub QuantusExtrinsicBaseWeight: Weight =
    Weight::from_parts(WEIGHT_REF_TIME_PER_NANOS.saturating_mul(230_427), 0);
```

The value is derived same-hardware so the two hardware frames are not mixed: `108_157 + (144_170 measured ML-DSA-87 and Poseidon, minus 21_900 measured sr25519)`. Treat it as a floor rather than a calibration, and replace it with the output of `benchmark overhead` run against this runtime on the project's own reference hardware.

The fix was applied to a working copy and the PoC re-run in both directions. At the scope pin the test fails with a 35 us shortfall. With the patch applied the configured value moves from 108.16 us to 230.43 us, the invariant holds, and the test passes.

ML-DSA-65 is also a live signature variant, and its verification measures 50.6 us. A single `base_extrinsic` has to cover the more expensive ML-DSA-87 path to be sound for both.

## References

* [`runtime/src/configs/mod.rs#L86-L89`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L86-L89)
* [`primitives/dilithium-crypto/src/traits.rs#L152-L185`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/dilithium-crypto/src/traits.rs#L152-L185)
* [`primitives/dilithium-crypto/src/traits.rs#L191-L200`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/dilithium-crypto/src/traits.rs#L191-L200)
* `frame-support` v47.0.0, `src/weights/extrinsic_weights.rs`, the generated provenance header for `ExtrinsicBaseWeight`
* `frame-system` v47.0.0, `src/limits.rs`, `BlockWeights::with_sensible_defaults`
* `frame-executive` v47.0.0, `src/lib.rs`, `do_apply_extrinsic`
* [Quantus-Network/chain `6f7094f`](https://github.com/Quantus-Network/chain/commit/6f7094f0679c89dbb80b575ec63531357c9d763e), the project's own fix for the same mechanism at smaller scale
* <https://docs.quantus.com/deep-dives/pqc/>, the project's published ML-DSA cost and size figures

## Link to Proof of Concept

<https://gist.github.com/chipsforbeer/f06a12bd0b0f2848d2594e0384aba441>

## Proof of Concept

The test reads the configured `base_extrinsic` back out of `RuntimeBlockWeights`, confirming the live value. It then times the production entry point `<DilithiumSignatureScheme as Verify>::verify`, which includes the Poseidon account derivation, and times sr25519 in the same binary for the hardware-independent ratio. The final assertion is the invariant that should hold: `base_extrinsic` must cover the signature verification it is the only budget for.

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-chain
cd audit-comp-quantus-chain && git checkout 3b243b870a54
cp base_extrinsic_weight_poc.rs runtime/tests/
cargo test --release -p quantus-runtime --test base_extrinsic_weight_poc -- --nocapture
```

At the scope pin this fails. With `suggested-fix.diff` applied it passes. `protoc` must be on PATH or `litep2p` fails to build.
