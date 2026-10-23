# 88633 bc medium extrinsicbaseweight does not account for ml dsa 87 verification every signed extrinsic undercharges node cpu by at least 55 7&#x20;

**Submitted on Aug 14th 2026 at 09:10:46 UTC by @rev3rse for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88633
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Inaccuracies to the weight (similar to gas) accounting system that under or overcharge by more than 50% of the correct amount (and are not intentional as indicated by the program's terms)

## Description

## Brief/Intro

`ExtrinsicBaseWeight` is the fixed per-extrinsic overhead the runtime charges for every signed extrinsic, and it is meant to cover signature verification. The shipped value is still Parity's — 108,157 ns, measured on 2024-11-08 on a Parity CI runner against a chain that verifies **sr25519 in a native host function**.

Quantus replaced all elliptic-curve cryptography with ML-DSA-87 and did neither of the two things that substitution requires. There is no host function — verification runs in-runtime in WASM. And the constant was never regenerated: `git log -- frame/support/src/weights/` returns exactly one commit, the vendoring commit.

I measured both schemes on one machine. The ML-DSA/sr25519 **ratio is 4.45×**, and because both are timed on the same host that ratio carries no hardware assumption. Substituting ML-DSA for sr25519 in the shipped constant gives a true per-extrinsic base weight of **at least 168,938 ns — an undercharge of at least 55.7%**, the worst figure across 14 runs under any build configuration. No hardware normalization is applied and no WASM execution penalty is assumed, even though ML-DSA runs in WASM while sr25519 ran as a native host function.

Separately and with no derivation at all, **`BlockExecutionWeight` is stale by +29.4%** — 431,614 ns shipped against 558,354 ns measured by the project's own `benchmark overhead` on the Quantus runtime.

Any signed extrinsic therefore obtains node CPU below its real cost. The cheapest signed calls, which are also the natural spam vehicle, are the most under-priced.

## Vulnerability Details

**Component:** `runtime/src/configs/mod.rs:86` (Quantus-original) and `pallets/frame-system/src/limits.rs:350` (forked, modified) — the in-scope code that **adopts** the constant. `primitives/dilithium-crypto/src/scheme_macro.rs:493` is the in-WASM verify it fails to price.

### Scope — the defect is in the code that adopts the constant, not the vendored file

I want to be exact here, because the number itself lives in a file that is **not** modified. `frame/support/src/weights/extrinsic_weights.rs` has exactly **one** commit — the vendoring commit — so by the program's "forked Substrate is in scope only at the diff" rule, that file on its own is out of scope. **I am not claiming a defect in vendored upstream code.**

The claim is against Quantus's own in-scope configuration, which adopts that constant unchanged for a chain whose signature scheme it replaced:

| anchor                                                                                             | commits | scope                                          |
| -------------------------------------------------------------------------------------------------- | ------- | ---------------------------------------------- |
| `runtime/src/configs/mod.rs:86` — builds `RuntimeBlockWeights` via `with_sensible_defaults`        | 91      | Quantus-original, **in scope in full**         |
| `pallets/frame-system/src/limits.rs:350` — `base_extrinsic: constants::ExtrinsicBaseWeight::get()` | 2       | forked **and modified** ⇒ in scope at the diff |
| `primitives/dilithium-crypto/src/scheme_macro.rs:493` — the in-WASM ML-DSA verify                  | —       | Quantus-original                               |
| `scripts/regenerate_weights.sh` — Quantus's own regeneration process, which omits this step        | —       | Quantus-original                               |

The decision that is Quantus's own, and wrong, is *adopting a signature-verification-derived constant measured against sr25519 for a chain that verifies ML-DSA-87 in WASM* — and never regenerating it while regenerating every pallet weight around it.

**Asset:** `immunefi-team/audit-comp-quantus-chain` (Blockchain/DLT)\
**Bug class:** Weight/gas accounting inaccuracy

### Root cause

#### 1. Verification runs in WASM — no host function exists

`grep -rn "runtime_interface\|sp_io::crypto\|host_function" primitives/dilithium-crypto/` returns nothing. `primitives/dilithium-crypto/src/scheme_macro.rs:493`:

```rust
pub fn $verify_fn(pub_key: &[u8], msg: &[u8], sig: &[u8]) -> bool {
    match qp_rusty_crystals_dilithium::$module::PublicKey::from_bytes(pub_key) {
        Ok(pk) => pk.verify(msg, sig, None),
        Err(e) => { log::warn!("public key failed to deserialize {:?}", e); false },
    }
}
```

#### 2. The constant is still Parity's and was never regenerated

`frame/support/src/weights/extrinsic_weights.rs`:

```
//! DATE: 2024-11-08 (Y/M/D)
//! HOSTNAME: `runner-wiukf8gn-project-674-concurrent-0`, CPU: `Intel(R) Xeon(R) CPU @ 2.60GHz`
//! SHORT-NAME: `extrinsic`, LONG-NAME: `ExtrinsicBase`, RUNTIME: `Development`
    Weight::from_parts(WEIGHT_REF_TIME_PER_NANOS.saturating_mul(108_157), 0);
```

#### 3. Nothing else charges for it

`runtime/src/lib.rs:154-176` — `ReversibleTransactionExtension::weight` and `WormholeProofRecorderExtension::weight` each account only for their own work.

## Impact Details

Every signed extrinsic is charged 108,157 ns of base weight for work that costs at least 168,938 ns. The block-weight budget is consequently overcommitted: a block filled with cheap signed extrinsics consumes materially more CPU than the runtime believes it has spent, and an attacker acquires that CPU below cost. The cheapest signed calls — the natural spam vehicle — are the most underpriced.

This maps to the program's named payable impacts:

* **Medium** — *"Inaccuracies to the weight (similar to gas) accounting system that under or overcharge by more than 50% of the correct amount"*: satisfied at **55.7%**, my worst measured run, with no hardware normalization and no assumed WASM penalty.
* **Low** — the same row under 50%: this is the floor I would fall back to only if triage rejects substituting my measured sr25519 for `s_p`, which would require the Xeon to be 1.133× faster than an M4 Pro.
* **Low** — `BlockExecutionWeight` at **+29.4%**, independently and with zero derivation.

I claim **Medium**. The figure carries no WASM penalty even though ML-DSA runs **in WASM** while sr25519 ran as a **native host function** (`grep -rn "runtime_interface\|sp_io::crypto" primitives/dilithium-crypto/` returns nothing), so the real gap is larger than what I claim.

## References

* `frame/support/src/weights/extrinsic_weights.rs` — the shipped 108,157 ns constant and its Parity provenance header
* `frame/support/src/weights/block_weights.rs` — the shipped 431,614 ns constant, itself stale by +29.4%
* `primitives/dilithium-crypto/src/scheme_macro.rs:493` — `$verify_fn`, the in-WASM verification path
* `node/src/service.rs:559,599` — executor constructed with `sp_io::SubstrateHostFunctions` only
* `node/src/benchmarking.rs` — `RemarkBuilder` / `create_benchmark_extrinsic`, signing with `Dilithium87Pair`
* `runtime/src/lib.rs:154-176` — the transaction extensions, none of which charge for verification
* `runtime/src/configs/mod.rs:483,486` — `WeightToFee = IdentityFee`, `LengthToFee = LengthToFeeMultiplier`
* Commit `75d32f8` "Inline frame core (#602)" — the only commit ever to touch `frame/support/src/weights/`
* FIPS 204 (ML-DSA) — ML-DSA-87 signature 4,627 B, public key 2,592 B; confirmed against `qp_rusty_crystals_dilithium::ml_dsa_87::{SIGNBYTES, PUBLICKEYBYTES}`
* Substrate weights reference: 1,000 weight units = 1 ns on reference hardware
* Prior audits reviewed for duplication: `quantus-substrate-audit-20260513.pdf` (Equilibrium; scope was treasury/multisig/scheduler/reversible-transfers at commit `0467a304`, with "other pallets and runtime modules not listed above" explicitly out of scope), `Proof-of-Work and Poseidon Security Review 1.1.pdf`, `Neodyme-Audit.pdf`, `EigerWormholeAudit.pdf`. None discusses `ExtrinsicBaseWeight`, `BlockExecutionWeight`, or the weighting of signature verification.
* `chain/scripts/regenerate_weights.sh` — Quantus's own weight-regeneration script; loops `benchmark pallet` over nine pallets and never invokes `benchmark overhead` (zero repo-wide hits)
* `chain/pallets/balances/src/weights.rs:22,24` — pallet weights regenerated 2026-08-08 on `minisqulenik.local`, versus the 2024-11-08 Parity CI provenance of both overhead constants
* `chain/frame/support/src/weights/extrinsic_weights.rs:67` — the only test on the constant; an upstream generic 10 µs–1 ms smoke test that a 9× error would pass

## Link to Proof of Concept

<https://gist.github.com/almog-shachar/1ad3106e0814ccbe860e41527ad531d8>

## Proof of Concept

{% stepper %}
{% step %}

## Apply and run the harness

```bash
git apply f6-poc.patch
cd poc/f6-verify-cost
cargo run --release
```

{% endstep %}

{% step %}

## Review the measured output

The harness times both schemes over 7 rounds of 300 iterations, discards the cold first round, and reports medians. The ML-DSA path is `from_bytes` + `verify`, exactly what `scheme_macro.rs:493` does per extrinsic. Measured output (full log in `evidence/f6-poc-output.txt`):

```
=== F6: signature verification cost, same machine (hardware-independent) ===
ML-DSA-87 from_bytes+verify :        89023 ns
sr25519 verify              :        18756 ns
ratio r                     :       4.7465x   <-- the load-bearing measurement

=== substituting ML-DSA for sr25519 in the shipped base weight ===
shipped ExtrinsicBaseWeight :       108157 ns
added cost (r-1)*sr25519    :        70268 ns
true base for this chain    :       178425 ns
UNDERCHARGE                 :        65.0%

=== sensitivity (the only assumption is s_p) ===
sr25519 cost on Parity HW needed to fall BELOW 50% :      14434 ns
we measured (Apple M4 Pro)                         :      18756 ns
=> the 2.60GHz Xeon would have to be 1.299x FASTER than an M4 Pro at sr25519
```

{% endstep %}
{% endstepper %}

### The derivation, and its single assumption

The shipped 108,157 ns covers decode, transaction-extension checks, dispatch **and** one sr25519 verify, measured on Parity's hardware. Quantus performs the same work with ML-DSA-87 instead. Writing `s_p` for sr25519's cost on Parity's hardware and `r` for the ML-DSA/sr25519 ratio:

```
true_base   = 108,157 - s_p + r·s_p  =  108,157 + (r - 1)·s_p
undercharge = (r - 1)·s_p / 108,157
```

`r` is measured **on one machine**, so it is hardware-independent — that is the load-bearing number and it carries no assumption. The only unknown is `s_p`, and I substitute my own measured sr25519 cost for it. **The sensitivity is stated rather than hidden:** for this to fall below the 50% row, `s_p` would have to be under 16,606 ns, i.e. Parity's 2.60 GHz Xeon would have to verify sr25519 **1.133× faster than an Apple M4 Pro**. It does not.

**Across 14 independent runs spanning two build configurations the undercharge ranged 55.7% – 64.4% (the crate as shipped in the patch measures 58.5% – 60.9%, median 59.4%). I claim 55.7% — the worst figure I have measured under any configuration.** Per-run figures are in `evidence/f6-run-spread.txt`.

### A correction to my own method

An earlier draft of this report normalized both timings by `BlockExecutionWeight` (558,354 / 431,614 = 1.294×), treating it as a pure hardware factor because it involves no signature verification, and derived +47.8%. **That normalizer is invalid and I withdraw it.** Parity measured `BlockExecutionWeight` on `substrate-node --chain=dev`; Quantus's own `benchmark overhead` measures the Heisenberg runtime, whose per-block hooks (QPoW, mining-rewards, zk-tree) are additional **work**, not slower hardware. Dividing by it asserted that a 2.60 GHz Xeon verifies sr25519 29% faster than an M4 Pro, which is not credible. The corrected derivation above needs no hardware normalization and no WASM penalty.

## A second, independent defect in the same commit-shaped gap

`BlockExecutionWeight` is itself stale, and this one needs **no** derivation:

|                                                                       |                                |
| --------------------------------------------------------------------- | ------------------------------ |
| Shipped (`block_weights.rs`, Parity, 2024-11-08)                      | **431,614 ns**                 |
| Measured by Quantus's own `benchmark overhead` on the Quantus runtime | **558,354 ns**                 |
| Undercharge                                                           | **+29.4%** (min sample +26.0%) |

The workload difference that disqualifies this number as a *normalizer* is exactly what makes it a *defect*: `BlockExecutionWeight` is meant to represent per-block overhead **for this runtime**, and it was measured on a different one.

Reproduce with:

```bash
quantus-node benchmark overhead \
  --runtime target/release/wbuild/quantus-runtime/quantus_runtime.compact.compressed.wasm \
  --genesis-builder runtime --genesis-builder-preset dev \
  --warmup 5 --repeat 20 --weight-path <dir>
```

(output in `evidence/regenerated-block_weights.rs`)

## The undercharge is not intentional, and the repository proves it

The Low and Medium rows both carry the qualifier *"and are not intentional as indicated by the program's terms."* Nothing in the program's terms mentions weights at all. The repository is affirmative on the other side:

1. **Quantus has its own weight-regeneration script that omits this step.**\
   `scripts/regenerate_weights.sh` loops `benchmark pallet` over nine pallets. It contains exactly one benchmark invocation, and **`benchmark overhead` appears nowhere in the repository** — zero hits repo-wide. The overhead constants are simply outside the process.
2. **The provenance headers disagree with each other.** Pallet weights were regenerated on Quantus hardware; the overhead constants were not:

   | file                                             | DATE       | HOSTNAME                                          |
   | ------------------------------------------------ | ---------- | ------------------------------------------------- |
   | `pallets/balances/src/weights.rs`                | 2026-08-08 | `minisqulenik.local`                              |
   | `frame/support/src/weights/extrinsic_weights.rs` | 2024-11-08 | `runner-wiukf8gn-project-674-concurrent-0` (Xeon) |
   | `frame/support/src/weights/block_weights.rs`     | 2024-11-08 | `runner-wiukf8gn-project-674-concurrent-0` (Xeon) |
3. **`git log -- frame/support/src/weights/` returns one commit** — the vendoring commit.
4. The only test on the constant (`extrinsic_weights.rs:67`) is upstream's generic smoke test asserting the value lies between 10 µs and 1 ms — a 100× window, which a 9× error would pass. It validates nothing about this chain.

Adding the git patch link here again: <https://gist.github.com/almog-shachar/1ad3106e0814ccbe860e41527ad531d8>
