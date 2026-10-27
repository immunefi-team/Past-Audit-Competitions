# 88485 bc high unverified nullifier public inputs bypass wormhole semantic txpool deduplication

**Submitted on Aug 13th 2026 at 12:35:10 UTC by @CarlosMB for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88485
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

## Summary

Wormhole exit transactions are unsigned, fee-free, and propagated through the transaction pool.

Quantus deliberately moved expensive Plonky2 verification out of `validate_unsigned()` and into `pre_dispatch()` to prevent every gossiped junk proof from immediately forcing expensive ZK verification.

Because proofs are therefore not cryptographically verified at pool-admission time, Quantus introduced a second mitigation: different representations of the same logical exit are deduplicated using a `provides` tag derived from the proof's nullifier set.

The vulnerability is that these nullifiers are themselves **unverified public inputs** at the moment the deduplication key is generated.

An attacker can therefore take **one existing serialized Wormhole proof**, modify only one nullifier public-input field, and re-serialize it.

Each mutation:

1. remains canonically serialized;
2. passes the cheap Wormhole pool checks;
3. receives a different `provides` tag;
4. is accepted as an independent transaction-pool entry;
5. is propagatable;
6. requires no account, signature, balance, or transaction fee;
7. and is rejected only later in `pre_dispatch()`, after full Plonky2 verification has executed.

A runnable regression PoC confirms that **one unchanged cryptographic proof can generate 32 independently admissible pool identities**, while every variant subsequently fails full ZK verification.

This directly bypasses the nullifier-based semantic deduplication introduced to prevent fee-free junk-proof amplification.

***

## Root Cause

The transaction pool uses attacker-controlled, **cryptographically unauthenticated nullifier public inputs** as the semantic identity of unsigned Wormhole exit transactions.

The nullifier is trusted for deduplication before the ZK verifier establishes that the nullifier actually belongs to the submitted proof.

As a result, the attacker controls the field that determines whether two fee-free proof submissions are treated as duplicates or as independent transactions.

***

## Vulnerability Details

### 1. Full ZK verification is intentionally deferred

`validate_unsigned()` performs only cheap pre-validation:

```rust
let (_, _, bundle, _validity) =
    Self::pre_validate_private_batch_proof(proof_bytes)
        .map_err(|_| InvalidTransaction::Call)?;

ValidTransaction::with_tag_prefix("WormholePrivateBatch")
    .and_provides(Self::exit_bundle_provides_tag(&bundle))
    .priority(crate::UNSIGNED_EXIT_PRIORITY)
    .longevity(5)
    .propagate(true)
    .build()
```

This is intentional.

Quantus previously performed full Plonky2 verification during unsigned transaction-pool validation. The security hardening commit that changed this behavior explains that unsigned, fee-free gossip could otherwise force approximately 17–25 ms of verification CPU for each junk proof variant.

The new design therefore:

* performs cheap pre-validation in `validate_unsigned()`;
* performs full verification in `pre_dispatch()`;
* and relies on nullifier-based semantic deduplication to prevent multiple representations of one logical proof from creating independent pool work.

Full cryptographic verification occurs here:

```rust
fn pre_dispatch(
    call: &Self::Call,
) -> Result<(), TransactionValidityError> {
    match call {
        Call::verify_private_batch { proof_bytes } =>
            Self::validate_private_batch_proof(proof_bytes)
                .map(|_| ())
                .map_err(|_| InvalidTransaction::Call.into()),
        // ...
    }
}
```

and eventually:

```rust
verifier.verify(proof)
    .map_err(|_| Error::<T>::ProofVerificationFailed)?;
```

Therefore, any values extracted from `proof.public_inputs` during `validate_unsigned()` have **not yet been cryptographically authenticated**.

### 2. The semantic deduplication key is entirely derived from unverified nullifiers

Wormhole generates the txpool `provides` tag using:

```rust
fn exit_bundle_provides_tag(bundle: &ExitBundle) -> [u8; 32] {
    let mut preimage = Vec::new();

    for segment in &bundle.segments {
        for nullifier in &segment.nullifiers {
            preimage.extend_from_slice(nullifier.as_ref());
        }
    }

    sp_io::hashing::blake2_256(&preimage)
}
```

The code describes this as semantic transaction-pool deduplication whose objective is effectively:

```
one logical exit
    ↓
one txpool identity
```

rather than:

```
one byte representation
    ↓
one txpool identity
```

However, the nullifiers used in this hash come directly from the still-unverified proof public inputs.

### 3. Cheap validation only checks whether the forged nullifier looks usable

Before ZK verification, the nullifier is checked for properties such as:

* correct layout;
* non-zero value;
* no duplicate inside the bundle;
* not already present in `UsedNullifiers`;
* not already claimed elsewhere in the bundle.

These conditions only establish that the **attacker-declared value is fresh**.

They do not establish:

```
submitted nullifier
        ==
nullifier cryptographically committed by this proof
```

Only Plonky2 verification establishes that property, and that happens later.

Therefore the attacker can select another fresh canonical nullifier value and pass all cheap checks.

### 4. Existing Quantus tests already confirm public inputs can be modified before verification

The existing test:

```
validate_unsigned_priority_ignores_unverified_amounts
```

mutates a proof public input, re-serializes the proof and confirms that the resulting cryptographically invalid proof still passes cheap pool validation.

The test explicitly describes this as:

```
"amount-inflated junk still passes cheap pool checks"
```

This confirms that changing a public input does not inherently break:

* proof deserialization;
* canonical re-serialization;
* or `validate_unsigned()`.

That existing mitigation is safe because changing an amount does not change the nullifier-based `provides` tag.

The vulnerability appears when the attacker modifies **the nullifier public input itself**, because this is simultaneously:

1. still unauthenticated; and
2. the input controlling semantic deduplication.

***

## Deterministic Attack Path

```
Existing Wormhole proof
        |
        | mutate one nullifier PI
        v
Canonical proof bytes
        |
        v
validate_unsigned()
        |
        |-- deserialize proof                     PASS
        |-- canonical encoding                    PASS
        |-- parse public inputs                   PASS
        |-- block/fee/asset checks                PASS
        |-- forged nullifier is fresh             PASS
        |
        v
exit_bundle_provides_tag(forged_nullifier)
        |
        v
NEW provides tag
        |
        v
Transaction independently enters txpool
        |
        v
propagate = true
        |
        v
block proposer selects transaction
        |
        v
BlockBuilder::push()
        |
        v
ValidateUnsigned::pre_dispatch()
        |
        v
validate_private_batch_proof()
        |
        v
Plonky2 verifier.verify(proof)
        |
        v
FAIL: modified nullifier is not authenticated by proof
        |
        v
transaction marked invalid
```

The attacker then changes the nullifier again:

```
same proof + nullifier B
        ↓
different extrinsic hash
        ↓
different provides tag
        ↓
another independently processable transaction
```

No new ZK proof needs to be generated.

## Proof of Concept

The following regression test was added to:

```
pallets/wormhole/src/tests.rs
```

```rust
#[test]
fn mutated_nullifiers_bypass_semantic_pool_dedup() {
    use qp_plonky2_verifier::field::types::Field;
    use sp_runtime::{
        traits::ValidateUnsigned,
        transaction_validity::TransactionSource,
    };
    use std::collections::BTreeSet;

    new_test_ext().execute_with(|| {
        setup_valid_block_state_for_test_proof();

        let n = crate::circuit_config::NUM_LEAF_PROOFS;

        // Private-batch PI layout:
        // 8 header felts
        // + 2*N exit slots * 5 felts
        // + N nullifiers * 4 felts
        let nullifier_start = 8 + (2 * n * 5);

        let mut tags = BTreeSet::new();

        for i in 1..=32u64 {
            let mut proof = deserialize_test_proof();

            // Only mutate one public-input felt belonging to the
            // first nullifier. The cryptographic proof is unchanged.
            proof.public_inputs[nullifier_start] =
                F::from_canonical_u64(1_000_000 + i);

            let call = crate::Call::<Test>::verify_private_batch {
                proof_bytes: proof.to_bytes(),
            };

            let validity =
                <Wormhole as ValidateUnsigned>::validate_unsigned(
                    TransactionSource::External,
                    &call,
                )
                .expect(
                    "forged nullifier should pass cheap pool admission"
                );

            assert!(
                tags.insert(validity.provides[0].clone()),
                "every forged nullifier should obtain a distinct provides tag"
            );

            assert!(
                <Wormhole as ValidateUnsigned>::pre_dispatch(&call).is_err(),
                "the unchanged ZK proof must fail against the forged public input"
            );
        }

        assert_eq!(
            tags.len(),
            32,
            "one underlying proof generated 32 independent pool identities"
        );
    });
}
```

Run:

```bash
cargo test -p pallet-wormhole \
    mutated_nullifiers_bypass_semantic_pool_dedup \
    --release -- --nocapture
```

Result:

```
test ...mutated_nullifiers_bypass_semantic_pool_dedup ... ok

test result: ok. 1 passed; 0 failed; 71 filtered out
finished in 0.07s
```

### What the PoC proves

The PoC deterministically proves all vulnerability-specific properties:

```
1 unchanged cryptographic proof
        |
        +--> forged nullifier #1
        |       validate_unsigned = OK
        |       provides = A
        |       pre_dispatch = ERR
        |
        +--> forged nullifier #2
        |       validate_unsigned = OK
        |       provides = B
        |       pre_dispatch = ERR
        |
        ...
        |
        +--> forged nullifier #32
                validate_unsigned = OK
                provides = AF
                pre_dispatch = ERR
```

Specifically, it proves that:

1. modifying a nullifier PI preserves cheap pool validity;
2. each modification bypasses semantic `provides` deduplication;
3. each variant reaches the full-verification gate;
4. every variant fails only when the expensive cryptographic verification is executed.

### What the PoC does not claim

This regression test is **not a full-node networking benchmark**.

It does not claim to measure a specific block-delay percentage or to prove complete network halt.

Those properties are not required for the reported `BC-H03` impact.

The reported issue is that the anti-DoS processing bound can be bypassed, allowing one proof to create arbitrarily many independently processable fee-free mempool transactions.

The processing behavior after pool admission follows deterministically from the production proposer implementation described below.

***

## Why Each Variant Causes Independent Proposer Processing

Quantus uses `sc-basic-authorship` for proposal construction.

For each ready transaction, the proposer calls:

```rust
BlockBuilder::push(...)
```

A forged Wormhole transaction therefore reaches `pre_dispatch()`, where the Plonky2 verification executes.

When verification fails, the proposer treats the transaction as invalid and removes it.

Importantly, this failure path does **not** trigger:

```rust
MAX_SKIPPED_TRANSACTIONS = 8
```

That counter applies to resource-exhaustion/block-capacity cases.

For ordinary validity failures, the proposer performs the equivalent of:

```
try transaction
    ↓
pre_dispatch
    ↓
ZK verification
    ↓
invalid
    ↓
mark invalid
    ↓
continue with next ready transaction
```

Therefore a ready pool containing attacker-generated variants can result in:

```
variant #1 → Plonky2 verification → invalid
variant #2 → Plonky2 verification → invalid
variant #3 → Plonky2 verification → invalid
variant #4 → Plonky2 verification → invalid
...
```

until the iterator is exhausted or the proposal deadline is reached.

***

## Why Invalid-Transaction Banning Does Not Stop the Attack

After one variant fails, its exact transaction hash can be marked invalid/banned.

This does not prevent further variants because changing the nullifier changes:

1. the serialized proof bytes;
2. the extrinsic hash;
3. the semantic `provides` tag.

Therefore:

```
variant A
    ↓
hash A banned

variant B
    ↓
hash B != hash A
provides B != provides A
    ↓
accepted independently
```

The attacker does not need to replay an already-banned transaction.

***

## Why QPoW's Rebuild Rate Limit Does Not Stop the Attack

QPoW limits transaction-triggered candidate rebuilds to approximately one every two seconds.

This mitigation remains effective and is not being bypassed.

The attack does not require:

```
1 submitted transaction
=
1 candidate rebuild
```

A **single proposal build** iterates over multiple ready transactions.

Therefore one rebuild can itself process many forged Wormhole variants sequentially.

***

## Impact

This vulnerability bypasses the security mechanism specifically added to prevent fee-free invalid Wormhole proofs from creating independent transaction-processing work.

The relevant asymmetry is:

```
Attacker:
one existing proof
+
cheap public-input mutations

Processing node:
for each independent variant
→ pool handling
→ proposer handling
→ full Plonky2 verification
→ rejection
```

The production benchmark constants estimate private-batch ZK verification at approximately:

```
15.162 ms
```

per proof.

The private-batch fixture is approximately:

```
157.5 KB
```

and the default transaction-pool capacity is approximately:

```
256 MiB
```

Quantus provides approximately a 10-second proposal duration, with basic-authorship reserving roughly 10% for finalization.

As a **capacity estimate only**, approximately 560 private-proof variants correspond to:

```
560 × 15.162 ms ≈ 8.49 seconds of ZK verification
```

and approximately:

```
560 × 157.5 KB ≈ 88 MB
```

of proof data.

This is below the default pool byte limit.

These estimates are not necessary to establish the vulnerability and are not presented as an E2E timing measurement. They demonstrate that the amount of independently processable junk permitted by the bypass is meaningful relative to the configured processing window.

The attack is additionally:

* unauthenticated;
* unsigned;
* fee-free;
* propagatable;
* repeatable from one proof;
* and does not require generating new valid ZK proofs.

This maps directly to:

**`BC-H03 — Causing processing nodes to process mempool transactions beyond set parameters`**

***

## Why This Is Not Intended Behavior

Deferring full verification to `pre_dispatch()` is intentional.

Allowing arbitrary independent identities derived from one proof is not.

The security change that introduced the current architecture explicitly combined:

```
deferred ZK verification
+
nullifier-set deduplication
```

because either component alone does not protect against fee-free invalid-proof amplification.

The bypass breaks the second part of that security boundary.

The issue therefore is not:

> "invalid proofs can reach pre\_dispatch"

which is expected.

The issue is:

> "one underlying invalid proof can be transformed into arbitrarily many independently admissible transactions because the deduplication identity is itself attacker-controlled before verification."

That defeats the intended semantic deduplication property.

***

## Recommended Mitigation

Do not use cryptographically unauthenticated public inputs as the sole anti-DoS identity for fee-free Wormhole proof submissions.

Possible mitigations include:

* introducing a separately bounded queue for unverified Wormhole proofs;
* applying global/per-peer limits to unverified Wormhole transactions;
* requiring authentication, fees, or a refundable bond for proof submission;
* moving proof verification into a bounded admission service with verification-result caching;
* preventing unrestricted propagation of proofs that have not yet been cryptographically authenticated.

Regression tests should additionally verify that arbitrary mutations of public inputs cannot transform one underlying proof into unbounded independent transaction-pool identities.
