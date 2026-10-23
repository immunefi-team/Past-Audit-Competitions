# 89247 bc high zero value wormhole exits bypass every fee floor letting a free settlement consume 83 of a block and permanently occupy 7 nullifier slots per spend

**Submitted on Aug 18th 2026 at 08:16:41 UTC by @ilov3c4ndy for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89247
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Block stuffing
  * Modification of transaction fees outside of design parameters
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours
  * Inaccuracies to the weight (similar to gas) accounting system that under or overcharge by more than 50% of the correct amount (and are not intentional as indicated by the program's terms)

## Description

**Target:** Substrate L1 node, `pallet-wormhole`, with the `qp-zk-circuits` private-batch aggregator\
**Severity:** High. Unbounded free growth of consensus-critical state, plus block-capacity consumption at roughly 1/17 of the fee cost.

## Summary

Wormhole exit settlement is an unsigned, `Pays::No` extrinsic. Its only economic gate is the volume fee, and that fee is levied on minted exit value rather than on the compute and storage the settlement consumes.

A leaf proof with both output amounts set to zero is a valid, fully bound, non-dummy spend. The leaf circuit's dummy sentinel is `block_hash == 0 AND outputs == 0`, so with a genuine block hash every binding stays enforced: nullifier, block hash, and ZK-tree root. Nothing requires a non-zero output, and the in-circuit fee relation

```
(output_1 + output_2) * 10000  <=  input_amount * (10000 - volume_fee_bps)
```

holds trivially at `output = 0` for any input, including an input of 0, which is what a 1-planck deposit quantizes to.

Settling that proof mints nothing, so `volume_fee_for_exit` returns 0. No burn, no miner fee, no transaction fee. The settlement is still charged the full worst-case declared weight, and still writes one permanent `UsedNullifiers` entry per leaf slot in the batch.

There is currently no floor of any kind on the cost of a wormhole settlement.

## Root cause

Two gaps compose.

### The fee is a function of value, not of resources

```rust
pub(crate) fn volume_fee_for_exit(minted_exit: u128, fee_bps: u32) -> u128 {
    (minted_exit / crate::SCALE_DOWN_FACTOR)
        .saturating_mul(fee_bps)
        .div_ceil(denominator)
        .saturating_mul(crate::SCALE_DOWN_FACTOR)
}
```

`minted_exit == 0` gives a fee of 0. The success path then returns `Ok(PostDispatchInfo { actual_weight: None, pays_fee: Pays::No })`, charging the block the full declared weight and collecting nothing.

### Dummy leaf padding costs full permanent storage

The private-batch circuit replaces dummy nullifiers with the hash of a prover-chosen preimage, deliberately, so padding stays indistinguishable from real slots:

```rust
let dummy_null_i = hash_dummy_nullifier_pre_image(builder, targets.dummy_nullifier_pre_images[i]);
selected_nullifiers.push([builder.select(is_dummy_i, dummy_null_i[j], real_null_i[j]), ...]);
```

`H(H(p))` is never zero, so `process_exit_bundle` persists every slot:

```rust
if nullifier_bytes == [0u8; 32] { continue; }
UsedNullifiers::<T>::insert(nullifier_bytes, true);
```

An attacker anchors a batch with one real leaf and pads with six dummies, giving 7 permanent entries per real spend. `UsedNullifiers` is the double-spend boundary and is never pruned.

## Three shipped invariants this breaks

Each landed after the last audit, and each is asserted in its own PR.

### PR #632 (30 Jul 2026)

PR #632 removed `MinimumTransferAmount` entirely, config type, error and exit check, on the strength of this replacement invariant:

> "Dust exits are still priced by the circuit's ceil rounding: at least 0.01 QUAN per nonzero exit."

The reasoning and its test plan cover only nonzero exits. The zero case pays nothing, and the on-chain check that would have caught it was deleted.

### PR #618 (22 Jul 2026)

PR #618 states as a design property: *"Zero nullifiers are never written to `UsedNullifiers`."* That holds only for dummy inner batches inside a public batch. A dummy leaf inside a real private batch gets a non-zero `H(H(p))` nullifier and it is written. The `segment_validity` comment repeats the error verbatim.

### PR #641 (6 Aug 2026)

PR #641 fixed "exit-bundle fees settled from unminted value" by routing all fee math through `minted_exit_amount`, *"only value that was actually credited."* Correct in isolation, and it is what drives the fee to exactly zero when nothing is credited. Together with #632, the two changes remove the last two cost floors.

## Impact

Deployed config from `pallets/wormhole/build.rs` defaults, confirmed in build output: `num_leaf_proofs = 7`, `num_private_batch_proofs = 53`. The runtime uses `RocksDbWeight` (25 us read, 100 us write) and `IdentityFee`. Block compute budget is 6 s at 75% `NORMAL_DISPATCH_RATIO`, so 4.5 s of `ref_time` per block, and `TARGET_BLOCK_TIME_MS = 12_000` gives 7,200 blocks per day.

| Extrinsic                         | Declared ref\_time | % of block |           Fee |
| --------------------------------- | -----------------: | ---------: | ------------: |
| `verify_public_batch`             |           3,744 ms |      83.2% |             0 |
| `verify_private_batch`            |              86 ms |       1.9% |             0 |
| 1-planck deposit (anchoring leaf) |            4.88 ms |      0.11% | \~0.0049 UNIT |

`verify_public_batch` is priced at `storage_tail(public_batch_max_exits())`, a fixed worst case of 742 exits that does not depend on actual content. A public batch of one real inner (1 real leaf, 6 dummy leaves) plus 52 all-dummy inners is charged that same 3,744 ms.

**One 1-planck deposit costing 4.88 ms of fee-bearing weight buys one fee-free extrinsic charged 3,744 ms, 83% of the block's entire Normal dispatch budget. Amplification is roughly 768x.** Two such transactions exceed a block, so a single attacker transaction per block displaces essentially all other traffic.

The sustained variant is cheaper to produce. Private batches are sized for mobile proving ("7 leaves: fits in `degree_bits=15` (\~1.5 GB peak memory for mobile)"), giving 17.6x amplification with 52 free settlements filling a block. In steady state the attacker pays for 5.37% of block capacity and occupies the rest, around 0.24 UNIT per block, or 1,740 UNIT per day to saturate the chain.

The more durable harm is state growth. At 52 settlements per block the attacker writes 366 `UsedNullifiers` entries per block, 314 of them for spends that never happened. At `max_size: 49` under a 48-byte `Blake2_128Concat` key, roughly 100 B per entry with trie overhead, that is about 37 KB per block or 264 MB per day of consensus state that is never pruned, plus one ZK-tree leaf per deposit. This burdens every node permanently and does not recover when the attack stops.

The underlying defect is a deleted security control justified by an invariant that does not hold. The fix is a validation change, not rate limiting.

## Not a duplicate

Checked against every public source. Several adjacent items exist and none of them is this bug.

### Known Issues, all three

Referendum vote removal, plaintext miner-to-node link, and zk-tree depth eventually exceeding the maximum from organic volume. None relate.

### V12 autonomous audit, both runs, 226 findings disclosed in #647 and #617

The nearest miss is #181420, "Invalid unsigned proofs trigger unpaid ZK verification". That is the invalid-proof variant, already mitigated by #631 moving the ZK verify off `validate_unsigned` onto `pre_dispatch`. This report is the valid zero-value proof variant: the proof verifies, the settlement succeeds, and the fee is zero because nothing was minted. V12 auto-triaged #181420 as invalid, and the team's 79-finding re-assessment never reached this case.

### Internal review issue #411, "Wormhole Pallet Review"

It contains three items that look close and are not:

* Item 3, garbage proof blobs passing `validate_unsigned`, is the invalid-proof DoS again, fixed by #631.
* Item 5, `MinimumTransferAmount` applied to the batch total rather than per exit, concerns the old minimum that #632 later deleted outright. This report is about the absence of any floor after that deletion.
* Item 7, "Fixed weight for variable work", raises the opposite direction, that declared weight might under-count real work. Here declared weight massively over-counts an inert settlement, which is exactly what lets an attacker consume budget without performing or paying for the work it was priced at. Later weight work in #631 and #641 addressed item 7 and did not touch this.

### Closed PR #568

Nullifier write ordering was argued non-exploitable because `#[pallet::call]` bodies are auto-wrapped in `with_storage_layer`. Unrelated.

### No public fix exists

The chain PR list refreshed on 18 Aug 2026 shows nothing merged after 15 Aug that touches this, and the only open chain PR is #660, warp sync.

## The code is unaudited

| Audit                    | Completed   |
| ------------------------ | ----------- |
| Equilibrium, ZK Circuits | 20 Mar 2026 |
| Equilibrium, Chain       | 13 May 2026 |

Every composing change landed after the last chain audit: `verify_public_batch` and the segment/dummy model in #618 on 22 Jul, the metering and unsigned-exit gate in #631 on 29 Jul, the removal of `MinimumTransferAmount` in #632 on 30 Jul, and the minted-value fee fix in #641 on 6 Aug.

## Existing mitigations do not cover this

`MAX_PROOF_BYTES` and the canonical-framing check bound per-proof bytes, not count or value. Deferring the ZK verify to `pre_dispatch` bounds pool-admission cost, not block-inclusion cost. A constant `UNSIGNED_EXIT_PRIORITY` prevents priority usurpation but does not gate volume. `MinimumTransferAmount`, the one check that did gate this, was removed in #632, and the circuit's ceil-rounded fee floor that replaced it evaluates to 0 at zero output. Miners have no fee incentive to include these settlements and no cost either, so they fill otherwise idle capacity.

## Remediation

{% stepper %}
{% step %}

## Reject economically inert bundles

In `validate_exit_bundle_common`, reject with `NoValidSegments` when the sum of `summed_output_amount` across all valid segments is zero. A settlement that mints nothing can only be griefing. This is the smallest targeted fix.
{% endstep %}

{% step %}

## Floor the fee per settled segment

Floor the fee per settled segment rather than per unit of value, at minimum one quantum per non-inert segment, or per non-zero nullifier persisted, so cost tracks the storage and compute caused rather than the value moved. This restores the guarantee #632 assumed.
{% endstep %}

{% step %}

## Refund unused weight

Refund unused weight with `actual_weight: Some(..)` on the success path, scaled to exits actually minted, so an inert settlement stops consuming budget it never used.
{% endstep %}

{% step %}

## Correct the dummy-nullifier documentation

Correct the dummy-nullifier documentation in `segment_validity` and #618. Dummy leaf padding inside a real private batch yields non-zero `H(H(p))` nullifiers that are persisted. As written, both hide the 7x state-growth multiplier from reviewers.
{% endstep %}
{% endstepper %}

## Link to Proof of Concept

<https://gist.github.com/joksdz/5fb2bd46906cc8b3564311f7f10da17d>

## Proof of Concept

Two passing tests, one per half of the system. Both ship in the attached gist.

### The circuit proves a zero-output, non-dummy leaf

```bash
cargo test --release -p tests zero_output_leaf_with_real_block_hash_is_provable -- --nocapture
```

```
PROVED: non-dummy leaf, block_hash=0xd5ab66b8cd05ec6f55d8c6e2cace9b08e3421c552c3ce8999645dda36f662995,
        outputs=(0,0), nullifier=0x4e7059e77c7213121dc74a104573270c18f64900abde406794fc93a323f2abe0
test prover::prover_tests::zero_output_leaf_with_real_block_hash_is_provable ... ok
```

It takes `CircuitInputs::test_inputs_0()`, replaces the dummy sentinel with the genuine header-derived block hash, asserts that hash is non-zero, zeroes both outputs, and proves. Because `block_hash != 0` the proof is not a dummy and every binding is enforced. This is a real spend that exits nothing.

### The pallet settles it for free and persists all 7 nullifiers

```bash
cargo test -p pallet-wormhole --lib poc_zero_value_exit
```

```
test tests::poc_zero_value_exit::zero_value_exit_pays_no_fee_but_permanently_occupies_every_nullifier_slot ... ok
```

It drives `process_exit_bundle` with one segment carrying 7 non-zero nullifiers (1 real, 6 dummy replacements) and 14 exit slots all at zero, then asserts the bundle settles, `total_issuance` is unchanged so nothing was burned and no miner fee paid, `volume_fee_for_exit(0, bps) == 0`, and all 7 nullifiers are present in `UsedNullifiers`.
