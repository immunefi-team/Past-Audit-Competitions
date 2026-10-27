# 76454 bc medium range program endofsource guard dead enabling trivial proof finalization halt

**Submitted on May 4th 2026 at 14:00:56 UTC by @Blobism for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76454
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

`crates/succinct/utils/client/src/client.rs:106` calls `cfg.is_isthmus_active(driver.cursor.read().l2_safe_head().block_info.number)`. `is_isthmus_active` is a *timestamp* predicate; live-chain Isthmus timestamps are \~1.7×10⁹, while L2 block numbers are \~10⁷, so the guard is dead on every live chain. The `EndOfSource` arm therefore silently lowers `target` to the current safe head and returns `Ok((safe_head_at_N, agreed_root, vec![]))`, which lets the SP1 range program publicly commit a `BootInfoStruct` with attacker-controlled `l2BlockNumber = N+M` and `l2PreRoot == l2PostRoot`. Any unprivileged user can mint such a trivial ZK proof and pair it against a legitimate ZK proposal at the same block number; the soundness-alert mechanism auto-nullifies both, halting finalization.

## Vulnerability Details

`crates/succinct/utils/client/src/client.rs:90-115`:

```rust
let mut attributes = match driver.pipeline.produce_payload(tip_cursor.l2_safe_head).await {
    Ok(attrs) => attrs.take_inner(),
    Err(PipelineErrorKind::Critical(PipelineError::EndOfSource)) => {
        if target.is_some() {
            target = Some(tip_cursor.l2_safe_head.block_info.number);
        };
        // Intended hard-fail post-Isthmus — but reads `block_info.number`
        // where `is_isthmus_active` expects a timestamp.
        if cfg.is_isthmus_active(driver.cursor.read().l2_safe_head().block_info.number) {
            return Err(PipelineError::EndOfSource.crit().into());
        }
        continue;
    }
    ...
};
```

`is_isthmus_active` (`crates/consensus/genesis/src/rollup.rs:217`):

```rust
pub fn is_isthmus_active(&self, timestamp: u64) -> bool {
    self.hardforks.isthmus_time.is_some_and(|t| timestamp >= t)
        || self.is_jovian_active(timestamp)
}
```

Live chain Isthmus timestamps (`crates/common/chains/src/config.rs`): mainnet `1_746_806_401`, sepolia `1_744_905_600`, alpha `1_744_300_800` — all ≫ any plausible L2 block number. The guard never fires; the loop falls through to `continue;` and on the next iteration returns `Ok((safe_head_at_N, agreed_root, vec![]))` with zero blocks executed.

The witness-executor epilogue (`crates/succinct/utils/client/src/witness/executor.rs:172`) then validates only `output_root == boot.claimed_l2_output_root`, with no `safe_head.block_info.number == boot.claimed_l2_block_number` check. With attacker-set `claimed = agreed = A`, the equality holds. `BootInfoStruct::new` (`crates/succinct/utils/client/src/boot.rs:36-58`) copies `boot_info.claimed_l2_block_number` straight into the public values, so the SP1 proof publicly commits `l2PreRoot == l2PostRoot == A`, `l2PreBlockNumber = N`, `l2BlockNumber = N+M`.

## Attack path

{% stepper %}
{% step %}

### Pick the target proposal

Choose `M > 0` targeting the next legitimate proposal at block `N+M`.
{% endstep %}

{% step %}

### Force `EndOfSource` immediately

Pick an `l1Head` too old to contain L2 batch data past block `N` (any L1 block at or before block `N`'s L1 origin). Derivation hits `EndOfSource` on the very first `produce_payload`. The aggregation program's `l1_heads_map` membership check (`crates/succinct/programs/aggregation/src/main.rs:55-75`) is satisfied by any ancestor of the latest L1 checkpoint.
{% endstep %}

{% step %}

### Build the witness and proof

Build a witness with `agreed = claimed = A`, `claimed_l2_block_number = N+M`, and the chosen `l1Head`. Run SP1 range + aggregation.
{% endstep %}

{% step %}

### Submit the competing dispute game

Submit a competing dispute game `(rootClaim = A, l2BlockNumber = N+M)` with the proof; `AggregateVerifier._verifyZkProof` (`contracts/src/multiproof/AggregateVerifier.sol:917-932`) accepts it.
{% endstep %}

{% step %}

### Trigger soundness-alert nullification

The legitimate proposer's game `(rootClaim = B, l2BlockNumber = N+M)` has a real ZK proof. Two valid ZK proofs of the same parent and ending block number with different output roots → soundness-alert nullification. Repeat per legitimate proposal; finalization stalls indefinitely.
{% endstep %}
{% endstepper %}

The TEE flow is not vulnerable (`crates/proof/tee/nitro-enclave/src/server.rs::prove` derives `ending_l2_block` from actually-executed blocks), but the soundness alert treats two valid ZK proofs as conclusively contradictory, which is enough.

## Impact Details

Severity: **Critical — total network shutdown.** Permissionless, indefinitely repeatable, halts L2 finalization and L1 withdrawals. Recovery requires a code patch + operator deployment.

## References

`base/base` commit: `e3467a2048881213b56739a54a876efb9c6ea103` (`v0.8.0-rc.28`)

* Vulnerable guard: `crates/succinct/utils/client/src/client.rs:106`.
* Timestamp predicate: `crates/consensus/genesis/src/rollup.rs:217-220`.
* `BlockInfo` struct (separate `number`/`timestamp`): `crates/consensus/protocol/src/block.rs:27-36`.
* Output-root-only epilogue check: `crates/succinct/utils/client/src/witness/executor.rs:172`.
* `BootInfoStruct::new` copies attacker-controlled `claimed_l2_block_number`: `crates/succinct/utils/client/src/boot.rs:36-58`.
* On-chain ZK verifier journal layout: `contracts/src/multiproof/AggregateVerifier.sol:917-932`.
* Per-chain Isthmus timestamps: `crates/common/chains/src/config.rs:166, 232, 289`.
* Soundness-alert nullification.

## Link to Proof of Concept

<https://gist.github.com/blobism/35c85ce1cee3fa7c519b6c0bc2d9aff0>

## Proof of Concept

**Note**: 142 is an arbitrary bug number that can be ignored.

Get the PoC Gist: <https://gist.github.com/blobism/35c85ce1cee3fa7c519b6c0bc2d9aff0>

```bash
git clone git@github.com:base/base.git --branch v0.8.0-rc.28
cd base

# apply patch
git apply poc.diff

cargo test -p base-succinct-client-utils --test poc_142_bug1 -- --nocapture
```

The branch adds one integration-test file (`crates/succinct/utils/client/tests/poc_142_bug1.rs`) and two dev-deps to `crates/succinct/utils/client/Cargo.toml` (`tokio`, `base-common-rpc-types-engine`). No production source files modified.

### Test 1 — `type_confusion_makes_isthmus_guard_dead_on_mainnet_shape`

Calls `RollupConfig::is_isthmus_active` directly on a mainnet-shaped config (`isthmus_time = Some(1_746_806_401)`).

* `is_isthmus_active(30_000_000) == false` — typical L2 tip block number is treated as pre-Isthmus.
* `is_isthmus_active(1_746_806_401) == true` — confirms the predicate is a timestamp predicate when fed a timestamp.

### Test 2 — `advance_to_target_returns_ok_on_end_of_source_with_unmet_target`

A `FakePipeline` whose `produce_payload` always returns `PipelineError::EndOfSource.crit()`, paired with a `FakeExecutor` whose four methods are `unreachable!()`. Cursor is seeded with safe head at `block_info.number = 30_000_000`, `block_info.timestamp = MAINNET_ISTHMUS_TIMESTAMP + 100_000` (post-Isthmus). Calls `advance_to_target(&mut driver, &cfg, Some(N + 1_000), 1).await`.

Asserted: result is `Ok`, returned safe head is `N` (not the target `N + 1_000`), output root is the agreed root, intermediate roots is empty, and `FakeExecutor`'s methods were never reached.

```
proved: advance_to_target returned Ok(safe_head=30000000, root=0xaa..aa,
  intermediate=[]) on EndOfSource with target=30001000
test advance_to_target_returns_ok_on_end_of_source_with_unmet_target ... ok
```

### Test 3 — `boot_info_struct_commits_false_claim`

Builds a `BootInfo` with `agreed = claimed = A`, `claimed_l2_block_number = N + 1_000`, feeds it to `BootInfoStruct::new(boot, N, vec![])`. Asserts `l2PreRoot == l2PostRoot == A`, `l2PreBlockNumber = N`, `l2BlockNumber = N + 1_000` — the public values the SP1 range program would commit, attesting to zero state change across 1000 blocks.

```
proved: BootInfoStruct{ l2PreRoot=0xaa..aa, l2PostRoot=0xaa..aa,
  l2PreBlockNumber=30000000, l2BlockNumber=30001000 } — false claim committed
test boot_info_struct_commits_false_claim ... ok
```
