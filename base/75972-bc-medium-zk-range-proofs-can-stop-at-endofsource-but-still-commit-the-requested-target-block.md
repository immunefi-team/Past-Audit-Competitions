# 75972 bc medium zk range proofs can stop at endofsource but still commit the requested target block allowing a short range proof to finalize an invalid longer range output root

**Submitted on May 2nd 2026 at 00:13:49 UTC by @Oxastronatey for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75972
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Direct loss to Base or users ≥ 10% of funds held within Bridge.

## Description

### Immunefi Impact

**Impact ID:** SC-C1\
**Description:** Forging or bypassing TEE or ZK proof verification in `AggregateVerifier` to finalize an invalid state root on L1\
**Severity:** Critical

***

## Summary

The ZK range program is supposed to prove that the claimed L2 output root corresponds to `claimed_l2_block_number`. Instead, the guest can halt derivation on `PipelineError::EndOfSource`, check only that the output root equals `claimed_l2_output_root`, and then commit `claimed_l2_block_number` from untrusted boot input into the public values. `AggregateVerifier` later hashes that claimed block number into the ZK journal and accepts the SP1 proof, so a valid proof for a shorter derivation range can satisfy a 600-block onchain proposal.

The exploit shape is: prove only 200 L2 blocks, emit 20 intermediate roots at the ZK guest's hardcoded 10-block interval, submit those same 20 roots as the contract's expected 600/30 roots, and set the root claim to the actual 200-block output root. The contract sees the right number of roots and the last root equals `rootClaim`, while the ZK public value falsely says the ending L2 sequence is `start + 600`.

This is a proof statement integrity bug across the in-scope `base/base` v0.8.0-rc.28 offchain ZK guest programs and the in-scope `base/contracts` v8.1.0 `AggregateVerifier`, where the offchain guest commits a claimed block number that it never actually derived to, and the onchain contract trusts it.

***

## Root Cause

### 1) The range guest uses a hardcoded 10-block intermediate-root interval, not the on-chain `INTERMEDIATE_BLOCK_INTERVAL`

The offchain ZK witness generator hardcodes a 10-block interval for intermediate root recording, independent of the on-chain `INTERMEDIATE_BLOCK_INTERVAL` parameter.

[`crates/succinct/utils/client/src/client.rs#L18-L19`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/succinct/utils/client/src/client.rs#L18-L19)

```rust
/// The default interval (in blocks) at which intermediate output roots are recorded.
pub const DEFAULT_INTERMEDIATE_ROOT_INTERVAL: u64 = 10;
// ^-- ZK witness generation uses a fixed 10-block root cadence, not the on-chain INTERMEDIATE_BLOCK_INTERVAL.
```

The witness generation service passes this hardcoded constant directly into SP1 stdin:

[`crates/proof/zk/service/src/backends/op_succinct/provider.rs#L100-L103`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/proof/zk/service/src/backends/op_succinct/provider.rs#L100-L103)

```rust
let stdin = self.host.witness_generator().get_sp1_stdin(
    witness,
    base_succinct_client_utils::client::DEFAULT_INTERMEDIATE_ROOT_INTERVAL,
    // ^-- hardcoded 10 is passed into the SP1 witness, independent of Sepolia's deployed interval of 30.
)?;
```

The Sepolia activation task sets `BLOCK_INTERVAL=600`, `INTERMEDIATE_BLOCK_INTERVAL=30`, and `PROOF_THRESHOLD=1`:

[`sepolia/2026-04-20-activate-multiproof/.env#L14-L16`](https://github.com/base/contract-deployments/blob/main/sepolia/2026-04-20-activate-multiproof/.env#L14-L16)

```
BLOCK_INTERVAL=600
INTERMEDIATE_BLOCK_INTERVAL=30
PROOF_THRESHOLD=1
```

That means the contract expects exactly `600 / 30 = 20` intermediate roots. A short proof of only 200 blocks at the hardcoded 10-block interval emits `200 / 10 = 20` roots — the same count.

***

### 2) On `EndOfSource`, the guest rewrites the target to the current safe head instead of failing the proof

When the derivation pipeline exhausts its L1 data source, the `advance_to_target` function silently downgrades the target block number to the current safe head and continues the loop. An intended Isthmus-mode guard exists but is called with a block number instead of a timestamp, rendering it ineffective.

[`crates/succinct/utils/client/src/client.rs#L80-L109`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/succinct/utils/client/src/client.rs#L80-L109)

```rust
if let Some(tb) = target
    && tip_cursor.l2_safe_head.block_info.number >= tb
{
    info!(target: "client", "Derivation complete, reached L2 safe head.");
    return Ok((
        tip_cursor.l2_safe_head,
        tip_cursor.l2_safe_head_output_root,
        intermediate_roots,
    ));
}

#[cfg(target_os = "zkvm")]
println!("cycle-tracker-report-start: payload-derivation");
let mut attributes = match driver.pipeline.produce_payload(tip_cursor.l2_safe_head).await {
    Ok(attrs) => attrs.take_inner(),
    Err(PipelineErrorKind::Critical(PipelineError::EndOfSource)) => {
        warn!(target: "client", "Exhausted data source; Halting derivation and using current safe head.");

        // Adjust the target block number to the current safe head, as no more blocks
        // can be produced.
        if target.is_some() {
            target = Some(tip_cursor.l2_safe_head.block_info.number);
            // ^-- attacker-controlled L1 data exhaustion downgrades the target
            //     from claimed_l2_block_number to the current safe head.
        };

        // If we are in interop mode, this error must be handled by the caller.
        // Otherwise, we continue the loop to halt derivation on the next iteration.
        if cfg.is_isthmus_active(driver.cursor.read().l2_safe_head().block_info.number) {
            // ^-- VULNERABLE: is_isthmus_active expects a timestamp, not an L2 block number.
            //     A Sepolia block number ~40,307,xxx is compared against hardfork timestamps
            //     in Unix seconds, so this condition is always false.
            return Err(PipelineError::EndOfSource.crit().into());
        }
        continue;
    }
```

The `is_isthmus_active` function operates on timestamps, confirming the type mismatch:

[`crates/consensus/genesis/src/rollup.rs#L216-L220`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/genesis/src/rollup.rs#L216-L220)

```rust
/// Returns true if Isthmus is active at the given timestamp.
pub fn is_isthmus_active(&self, timestamp: u64) -> bool {
    self.hardforks.isthmus_time.is_some_and(|t| timestamp >= t)
        || self.is_jovian_active(timestamp)
    // ^-- caller passed block_info.number (~40M), but this compares against hardfork
    //     activation timestamps in Unix seconds (~1.7B). The condition stays false,
    //     so the guest accepts early data exhaustion instead of failing the proof.
}
```

***

### 3) The proof only checks root equality, not that the claimed block number was actually reached

After `advance_to_target` returns, the witness executor checks that the output root matches the boot-claimed output root, but never checks that the safe head block number equals the boot-claimed block number.

[`crates/succinct/utils/client/src/witness/executor.rs#L158-L179`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/succinct/utils/client/src/witness/executor.rs#L158-L179)

```rust
let (safe_head, output_root, intermediate_roots) = advance_to_target(
    &mut driver,
    rollup_config.as_ref(),
    Some(boot.claimed_l2_block_number),
    // ^-- requested target is start + 600 in the attack.
    intermediate_root_interval,
)
.await?;

// ...

if output_root != boot.claimed_l2_output_root {
    return Err(anyhow!(
        "Failed to validate L2 block #{number} with claimed output root {claimed_output_root}. Got {output_root} instead",
        number = safe_head.block_info.number,
        output_root = output_root,
        claimed_output_root = boot.claimed_l2_output_root,
    ));
}
// ^-- MISSING CHECK: safe_head.block_info.number == boot.claimed_l2_block_number.
//     The proof accepts a root for block S+200 as satisfying a claim for block S+600.
```

The function then returns the original boot input, not a corrected version reflecting the actual safe head reached:

[`crates/succinct/utils/client/src/witness/executor.rs#L198`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/succinct/utils/client/src/witness/executor.rs#L198)

```rust
Ok((boot_clone, intermediate_roots))
// ^-- returns the original boot input (with claimed_l2_block_number = S + 600),
//     not a record corrected to the actual safe_head.number (S + 200).
```

***

### 4) The committed public values use the requested block number, not the actual reached block

The `BootInfoStruct` constructed for SP1 commitment uses `boot_info.claimed_l2_block_number` directly from the untrusted boot input, regardless of how many blocks were actually derived.

[`crates/succinct/utils/client/src/boot.rs#L35-L57`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/succinct/utils/client/src/boot.rs#L35-L57)

```rust
impl BootInfoStruct {
    /// Create from a [`BootInfo`] and intermediate state roots.
    pub fn new(
        boot_info: BootInfo,
        l2_pre_block_number: u64,
        intermediate_roots: Vec<B256>,
    ) -> Self {
        Self {
            l1Head: boot_info.l1_head,
            l2PreRoot: boot_info.agreed_l2_output_root,
            l2PostRoot: boot_info.claimed_l2_output_root,
            l2PreBlockNumber: l2_pre_block_number,
            l2BlockNumber: boot_info.claimed_l2_block_number,
            // ^-- public values claim the requested target (S + 600),
            //     even when derivation stopped early at S + 200.
            rollupConfigHash: hash_rollup_config(&boot_info.rollup_config),
            intermediateRoots: Bytes::from(
                intermediate_roots
                    .iter()
                    .flat_map(|root| root.as_slice())
                    .copied()
                    .collect::<Vec<u8>>(),
            ),
        }
    }
}
```

The range program commits this struct immediately after `executor.run` accepts the short derivation:

[`crates/succinct/programs/range/utils/src/lib.rs#L63-L71`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/succinct/programs/range/utils/src/lib.rs#L63-L71)

```rust
            executor
                .run(boot_info, pipeline, cursor, l2_provider, intermediate_root_interval)
                .await
                .unwrap();
        // ...
    };

    sp1_zkvm::io::commit(&BootInfoStruct::new(boot_info, l2_pre_block_number, intermediate_roots));
    // ^-- the range proof commits the forged BootInfoStruct after executor.run
    //     accepts the short derivation without checking the reached block number.
```

***

### 5) Aggregation preserves the forged claimed ending block and produces the on-chain journal digest

The aggregation program verifies each range proof's public values, then consolidates them. It does not independently validate that the claimed block numbers match actual derivation — it trusts the range proofs' committed values.

[`crates/succinct/programs/aggregation/src/main.rs#L44-L52`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/succinct/programs/aggregation/src/main.rs#L44-L52)

```rust
agg_inputs.boot_infos.iter().for_each(|boot_info| {
    // In the range program, the public values digest is just the hash of the ABI encoded
    // boot info.
    let serialized_boot_info = bincode::serialize(&boot_info).unwrap();
    let pv_digest = Sha256::digest(serialized_boot_info);

    sp1_lib::verify::verify_sp1_proof(&agg_inputs.multi_block_vkey, &pv_digest.into());
    // ^-- aggregation verifies the range proof over the forged BootInfoStruct,
    //     not over actual safe_head metadata.
});
```

The consolidated boot info propagates the forged end block:

[`crates/succinct/programs/aggregation/src/main.rs#L80-L97`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/succinct/programs/aggregation/src/main.rs#L80-L97)

```rust
let intermediate_roots: Bytes = agg_inputs
    .boot_infos
    .iter()
    .flat_map(|boot_info| boot_info.intermediateRoots.iter().copied())
    .collect::<Vec<u8>>()
    .into();

let final_boot_info = BootInfoStruct {
    l2PreRoot: first_boot_info.l2PreRoot,
    l2PreBlockNumber: first_boot_info.l2PreBlockNumber,
    l2BlockNumber: last_boot_info.l2BlockNumber,
    // ^-- forged claimed end block (S + 600) is propagated into the aggregate proof.
    l2PostRoot: last_boot_info.l2PostRoot,
    l1Head: agg_inputs.latest_l1_checkpoint_head,
    rollupConfigHash: last_boot_info.rollupConfigHash,
    intermediateRoots: intermediate_roots,
};
```

The aggregate outputs become the on-chain journal digest:

[`crates/succinct/programs/aggregation/src/main.rs#L102-L118`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/succinct/programs/aggregation/src/main.rs#L102-L118)

```rust
let agg_outputs = AggregationOutputs {
    proverAddress: agg_inputs.prover_address,
    l1Head: final_boot_info.l1Head,
    l2PreRoot: final_boot_info.l2PreRoot,
    startingL2SequenceNumber: final_boot_info.l2PreBlockNumber,
    l2PostRoot: final_boot_info.l2PostRoot,
    endingL2SequenceNumber: final_boot_info.l2BlockNumber,
    // ^-- this becomes the on-chain AggregateVerifier endingL2SequenceNumber.
    intermediateRoots: final_boot_info.intermediateRoots,
    rollupConfigHash: final_boot_info.rollupConfigHash,
    imageHash: multi_block_vkey_b256,
};

let packed = agg_outputs.abi_encode_packed();
let digest = keccak256(&packed);
sp1_zkvm::io::commit_slice(digest.as_ref());
// ^-- commits exactly the digest ZKVerifier later checks as public values.
```

***

### 6) `AggregateVerifier` only checks the intermediate-root count/endpoint indirectly and then trusts the ZK proof

The contract checks that the last intermediate root matches `rootClaim`, but does not independently verify the number of intermediate roots against the expected `BLOCK_INTERVAL / INTERMEDIATE_BLOCK_INTERVAL`:

[`src/multiproof/AggregateVerifier.sol#L344-L349`](https://github.com/base/contracts/blob/v8.1.0/src/multiproof/AggregateVerifier.sol#L344-L349)

```solidity
// Last intermediate root has to match the proposal's claim
if (intermediateOutputRoot(intermediateOutputRootsCount() - 1) != rootClaim().raw()) {
    revert IntermediateRootMismatch(
        intermediateOutputRoot(intermediateOutputRootsCount() - 1), rootClaim().raw()
    );
}
// ^-- only the last root is checked; roots from a 200-block/10-step proof
//     can satisfy a 600-block/30-step proposal as long as the count matches.
```

The contract enforces that `l2SequenceNumber()` equals `startingOutputRoot.l2SequenceNumber + BLOCK_INTERVAL`, then passes this to `_verifyProof`:

[`src/multiproof/AggregateVerifier.sol#L367-L403`](https://github.com/base/contracts/blob/v8.1.0/src/multiproof/AggregateVerifier.sol#L367-L403)

```solidity
// The block number must be BLOCK_INTERVAL blocks after the starting block number.
if (l2SequenceNumber() != startingOutputRoot.l2SequenceNumber + BLOCK_INTERVAL) {
    revert UnexpectedBlockNumber(startingOutputRoot.l2SequenceNumber + BLOCK_INTERVAL, l2SequenceNumber());
}
// ^-- contract believes the proposal is for start + 600.

// ...

_verifyProof(
    proof[65:],
    proofType,
    gameCreator(),
    l1OriginHash,
    startingOutputRoot.root.raw(),
    uint64(startingOutputRoot.l2SequenceNumber),
    rootClaim().raw(),
    uint64(l2SequenceNumber()),
    // ^-- forged aggregate public value uses this 600-block ending sequence number.
    intermediateOutputRoots()
);
```

The ZK journal hash and verification accept the forged values:

[`src/multiproof/AggregateVerifier.sol#L904-L933`](https://github.com/base/contracts/blob/v8.1.0/src/multiproof/AggregateVerifier.sol#L904-L933)

```solidity
bytes32 journal = keccak256(
    abi.encodePacked(
        proposer,
        l1OriginHash,
        startingRoot,
        startingL2SequenceNumber,
        endingRoot,
        endingL2SequenceNumber,
        intermediateRoots,
        CONFIG_HASH,
        ZK_RANGE_HASH
    )
);

// Validate the proof.
if (!ZK_VERIFIER.verify(proofBytes, ZK_AGGREGATE_HASH, journal)) revert InvalidProof();
// ^-- the ZK proof verifies because the aggregate program committed this forged journal digest.
```

[`src/multiproof/zk/ZKVerifier.sol#L32-L45`](https://github.com/base/contracts/blob/v8.1.0/src/multiproof/zk/ZKVerifier.sol#L32-L45)

```solidity
function verify(
    bytes calldata proofBytes,
    bytes32 imageId,
    bytes32 journal
)
    external
    view
    override
    notNullified
    returns (bool)
{
    SP1_VERIFIER.verifyProof(imageId, abi.encodePacked(journal), proofBytes);
    // ^-- only validates SP1 proof/public value consistency;
    //     it cannot detect that the guest stopped early.
    return true;
}
```

***

### 7) With `PROOF_THRESHOLD=1`, the invalid proposal can resolve as `DEFENDER_WINS`

[`src/multiproof/AggregateVerifier.sol#L456-L466`](https://github.com/base/contracts/blob/v8.1.0/src/multiproof/AggregateVerifier.sol#L456-L466)

```solidity
// Game must be completed with a valid proof and enough proofs.
if (!gameOver()) revert GameNotOver();
if (proofCount < PROOF_THRESHOLD) revert NotEnoughProofs();
// ^-- Sepolia task sets PROOF_THRESHOLD=1, so the initial forged ZK proof is sufficient.

// If the game is challenged, reward the challenger.
if (counteredByIntermediateRootIndexPlusOne > 0) {
    status = GameStatus.CHALLENGER_WINS;
    bondRecipient = proofTypeToProver[ProofType.ZK];
} else {
    status = GameStatus.DEFENDER_WINS;
    // ^-- unchallenged invalid root resolves as valid after the slow delay.
}
```

***

## Internal Pre-conditions

1. The active `AggregateVerifier` implementation uses the deployed Sepolia-style intervals: `BLOCK_INTERVAL=600`, `INTERMEDIATE_BLOCK_INTERVAL=30`, `PROOF_THRESHOLD=1`.
2. The ZK range/aggregate hashes correspond to the scoped programs above.
3. The attacker can obtain or generate a valid SP1 aggregate proof for custom boot inputs.
4. The chosen L1 head has enough data to derive a shorter range (e.g. 200 blocks) but not the full 600 blocks.

## External Pre-conditions

1. The attacker can pay the proposal bond and proof-generation cost. No owner, manager, Security Council, sequencer, TEE registrar, or Base infrastructure compromise is required.
2. The L1 origin blockhash used in the proof prefix is still within the `AggregateVerifier` blockhash/EIP-2935 verification window.
3. No honest challenger submits a valid counterproof before finalization. This is the same dispute window assumption the Critical impact definition is testing; the bug is that the proof system accepts a false statement as valid.

## Attack Path

{% stepper %}
{% step %}

## Start from a valid anchor/root at L2 block `S`

For example, use `S = 40,307,663` from the Sepolia `.env`.
{% endstep %}

{% step %}

## Choose a real L1 head that only contains enough data to derive `S + 200`

It must not contain enough data to derive `S + 600`.
{% endstep %}

{% step %}

## Run the SP1 range proof with custom boot inputs

* `boot.claimed_l2_block_number = S + 600`
* `boot.claimed_l2_output_root = outputRoot(S + 200)`
* `intermediate_root_interval = 10`
  {% endstep %}

{% step %}

## `advance_to_target()` handles `EndOfSource` by rewriting the target

At `client.rs#L95`, it rewrites the target to `S + 200`. The `is_isthmus_active` guard at `client.rs#L106` fails because it receives a block number instead of a timestamp.
{% endstep %}

{% step %}

## The loop continues and returns the shorter safe head

The function returns at `client.rs#L80-L88` when `tip_cursor.l2_safe_head.block_info.number >= S + 200`.
{% endstep %}

{% step %}

## `WitnessExecutor::run()` accepts the short output root

It accepts because the returned root equals `boot.claimed_l2_output_root`; it never checks that the returned safe-head number is `S + 600`.
{% endstep %}

{% step %}

## `BootInfoStruct::new()` commits the forged block number

It commits `l2BlockNumber = S + 600` and `l2PostRoot = outputRoot(S + 200)`.
{% endstep %}

{% step %}

## The range proof and aggregate proof preserve the forged values

The range proof commits this at `lib.rs#L71`, and the aggregate proof propagates it at `main.rs#L92` and `main.rs#L108`.
{% endstep %}

{% step %}

## Create an `AggregateVerifier` game with mismatched root and sequence number

Use `rootClaim = outputRoot(S + 200)` while the clone immutable `l2SequenceNumber()` is `S + 600`.
{% endstep %}

{% step %}

## The root-count expectation still matches

`600 / 30 = 20`, exactly matching the `200 / 10 = 20` roots emitted by the ZK guest.
{% endstep %}

{% step %}

## `ZKVerifier.verify()` accepts the SP1 proof

`proofCount` becomes 1, meeting `PROOF_THRESHOLD`.
{% endstep %}

{% step %}

## The game resolves as `DEFENDER_WINS`

After the slow finalization delay, `resolve()` sees `proofCount >= PROOF_THRESHOLD` and sets `DEFENDER_WINS`.
{% endstep %}

{% step %}

## Result

An invalid L2 output root for block `S + 200` is finalized on L1 as the root for block `S + 600`.
{% endstep %}
{% endstepper %}

***

## Impact

* **Impact:** Critical - SC-C1 (bypassing ZK proof verification in `AggregateVerifier` to finalize an invalid state root on L1). Downstream, if bridge withdrawal logic consumes the finalized root, the same root-for-wrong-block finalization can become an SC-C2-style bridge-drain primitive: the attacker crafts a withdrawal proof against the finalized output root, which is valid for block `S + 200` but the bridge believes it covers through `S + 600`.
* **Likelihood:** Medium-high - the exploit requires generating a valid SP1 proof (nontrivial cost) but is fully permissionless. No trusted role compromise, no race condition, no brute-force. The attacker controls the L1 head selection and boot inputs.
* **Severity:** Critical.

***

## Negative Check

* **The official proposer/prover service does not save this** because onchain verification cannot assume the official host path was used. The attacker can run the public SP1 range program ([`crates/succinct/programs/range/utils/src/lib.rs`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/succinct/programs/range/utils/src/lib.rs#L63-L71)) and aggregate program ([`crates/succinct/programs/aggregation/src/main.rs`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/succinct/programs/aggregation/src/main.rs#L80-L118)) independently with custom boot inputs. `AggregateVerifier` only receives the resulting journal digest through [`_verifyZkProof()`](https://github.com/base/contracts/blob/v8.1.0/src/multiproof/AggregateVerifier.sol#L904-L933).
* **This is NOT the public SP1 soundness issue** (known issue #2 in the [Known Vulnerabilities in Base Azul](https://drive.google.com/file/d/1CwxsIZnRcjTXIkYFEw_xF4_rOwePQ55t/view?usp=sharing) document). The SP1 proof can be perfectly sound here; the bug is that the guest statement proves the wrong thing — it commits `claimed_l2_block_number` without verifying derivation actually reached that block. The known issue is the SP1 recursion-circuit advisory for versions v6.0.0 through v6.0.2 ([GHSA-63x8-x938-vx33](https://github.com/succinctlabs/sp1/security/advisories/GHSA-63x8-x938-vx33)), not this Base-specific public-value mismatch.
* **This is NOT known issue #5** ("No Block Range Validation on ProveBlock" in the [Known Vulnerabilities in Base Azul](https://drive.google.com/file/d/1CwxsIZnRcjTXIkYFEw_xF4_rOwePQ55t/view?usp=sharing) document). That known issue is about service request validation for zero/max/overflowing `number_of_blocks_to_prove` values. This candidate works with normal production-scale ranges: 600 contract blocks and a 200-block early derivation stop.
* **This is NOT GF-2** (chain-id binding: [base/base#2419](https://github.com/base/base/pull/2419), [commit `eba5b07a`](https://github.com/base/base/commit/eba5b07a14c4eff5975c03d95bdfc50b4680bcf6)). Even if chain ID is correctly bound, the guest can still commit `claimed_l2_block_number` while the actual derived safe head is lower.

***

## Test Suite Grounding

* **Existing test coverage:** [`WitnessExecutor::run`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/succinct/utils/client/src/witness/executor.rs#L158-L179) validates `output_root == boot.claimed_l2_output_root`, but no test verifies the stronger invariant `safe_head.block_info.number == boot.claimed_l2_block_number`.
* **Gap:** No test covers the case where [`advance_to_target`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/succinct/utils/client/src/client.rs#L80-L109) returns early due to `EndOfSource` while the boot input claims a higher target block. The [`is_isthmus_active`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/consensus/genesis/src/rollup.rs#L216-L220) guard at `client.rs#L106` is never tested with a block number input to confirm it correctly rejects the `EndOfSource` path.

***

## Mitigation

Patch the guest first; the on-chain contract cannot safely distinguish a sound proof of the wrong statement from a correct proof.

**1. In `WitnessExecutor::run()` — add block number check:**

```rust
if safe_head.block_info.number != boot.claimed_l2_block_number {
    return Err(anyhow!(
        "derived L2 block {actual} did not reach claimed target {claimed}",
        actual = safe_head.block_info.number,
        claimed = boot.claimed_l2_block_number,
    ));
    // ^-- enforce that the proof actually reaches the claimed target block.
}
```

**2. In `advance_to_target()` — fail closed on `EndOfSource` in proof mode:**

```rust
Err(PipelineErrorKind::Critical(PipelineError::EndOfSource)) => {
    return Err(PipelineError::EndOfSource.crit().into());
    // ^-- proof generation must fail when L1 data is insufficient for the requested target.
}
```

If the `EndOfSource` behavior must remain for non-proof clients, split it behind an explicit mode flag and make proof/zkVM execution fail closed.

**3. Fix the `is_isthmus_active` guard — pass timestamp, not block number:**

```rust
if cfg.is_isthmus_active(tip_cursor.l2_safe_head.block_info.timestamp) {
    return Err(PipelineError::EndOfSource.crit().into());
    // ^-- pass timestamp, not block number.
}
```

**4. Bind the intermediate-root interval into the proof statement.** The range guest, aggregate guest, and `AggregateVerifier` should agree on the same `INTERMEDIATE_BLOCK_INTERVAL`, or the aggregate public values should include the interval and the contract should reject mismatches.

## Proof of Concept

<details>

<summary>Click to expand Proof of Concept</summary>

**PoC type:** Rust executable integration PoC using the in-scope `base-succinct-client-utils` crate.

Create the following two files from the embedded snippets below in any temporary local directory:

* `run_poc.sh` — installs the executable example into the scoped crate and runs it with `cargo run --locked`.
* `endofsource_real_advance_poc.rs` — calls the real `base_succinct_client_utils::client::advance_to_target()` with a lightweight local `Driver` whose pipeline returns the real `PipelineErrorKind::Critical(PipelineError::EndOfSource)`.

**How to run:**

```bash
mkdir -p base_azul_endofsource_poc
cd base_azul_endofsource_poc

# Copy the run_poc.sh snippet below into ./run_poc.sh.
# Copy the Rust executable example below into ./endofsource_real_advance_poc.rs.

chmod +x ./run_poc.sh
BASE_AZUL_REPO="/absolute/path/to/base-v0.8.0-rc.28-offchain-workspace" ./run_poc.sh
```

`BASE_AZUL_REPO` must point to the root of the extracted Base Azul `base/base` offchain workspace, not to the PoC directory. That workspace root must contain `crates/succinct/utils/client/Cargo.toml`. The runner does not require an RPC URL, a private key, public Sepolia/mainnet interaction, or an SP1 proving cluster. It uses the reviewer-provided local workspace only.

**Expected successful output shape:**

The first runner paths will reflect the reviewer's local machine. The security-relevant output is that real `advance_to_target()` returns `START + 200` after being asked to reach `START + 600`, then the real public-value/journal structs commit `START + 600` with the `START + 200` output root.

```
[+] Installed PoC example at <BASE_AZUL_REPO>/crates/succinct/utils/client/examples/endofsource_real_advance_poc.rs
[+] Running: cargo run --locked -p base-succinct-client-utils --example endofsource_real_advance_poc
Base Azul real advance_to_target EndOfSource PoC
------------------------------------------------
START                                      = 40307663
requested target                           = START + 600 = 40308263
mock safe head before real call             = START + 200 = 40307863
real advance_to_target returned safe head   = 40307863
real advance_to_target returned root        = 0xa2d4156200c16d5b6ac597f37e785c9643143ffb42df924a01e163c4b326241d
produce_payload EndOfSource calls           = 1
executor update/execute calls               = 0/0
Isthmus active with timestamp               = true
Isthmus active with block number            = false
WitnessExecutor root-only epilogue accepts  = true
missing block-number equality would be      = false
BootInfoStruct.l2BlockNumber                = 40308263
BootInfoStruct.l2PostRoot                   = 0xa2d4156200c16d5b6ac597f37e785c9643143ffb42df924a01e163c4b326241d
ZK cadence roots                            = 200 / 10 = 20
AggregateVerifier expected roots            = 600 / 30 = 20
packed journal bytes                        = 836
rollupConfigHash                            = 0x12e9c45f19f9817c6d4385fad29e7a70c355502cf0883e76a9a7e478a85d1360
aggregation digest                          = 0x2b6cb2c3ed88fb4aa55e489b56eda9a8e4030e04677e157d16d033468f68eecb
AggregateVerifier-style digest              = 0x2b6cb2c3ed88fb4aa55e489b56eda9a8e4030e04677e157d16d033468f68eecb

PASS: real advance_to_target() returned START+200 after being requested to reach START+600
PASS: Isthmus guard is bypassed because the real code passes block number where timestamp is expected
PASS: WitnessExecutor-style root-only check accepts the returned START+200 root as the claimed root
PASS: real BootInfoStruct::new() commits START+600 with the START+200 output root
PASS: 20 short-range ZK roots match the contract's expected 20 intermediate roots
PASS: aggregation abi_encode_packed digest matches AggregateVerifier._verifyZkProof() journal layout
```

**What the PoC proves:**

1. It calls the real scoped `advance_to_target()` implementation, not a reimplementation.
2. The local pipeline returns the real `PipelineError::EndOfSource.crit()` error, causing the vulnerable target rewrite inside `advance_to_target()`.
3. The real function returns the existing safe head at `START + 200` even though the requested target was `START + 600`.
4. The real timestamp-based Isthmus API returns active for the safe-head timestamp but inactive when called with the block number, matching the vulnerable guard.
5. The `WitnessExecutor`-style epilogue accepts root equality while the missing block-number equality check would fail.
6. The real `BootInfoStruct::new()` commits `l2BlockNumber = START + 600` while using the `START + 200` output root.
7. The short-range 200-block proof cadence at the hardcoded ZK interval emits `200 / 10 = 20` roots, matching the contract's expected `600 / 30 = 20` roots.
8. The real `AggregationOutputs::abi_encode_packed()` digest matches the `AggregateVerifier._verifyZkProof()` journal layout.

### Runner

`base_azul_endofsource_poc/run_poc.sh`

### Executable Example

`base_azul_endofsource_poc/endofsource_real_advance_poc.rs`

**Dependencies:** Rust/Cargo compatible with the scoped Base Azul workspace. No RPC URL, no private key, no public Sepolia/mainnet interaction, no SP1 proving cluster needed.

</details>
