# 76045 bc medium malformed batch will be retried as invalid payload attributes forever

**Submitted on May 2nd 2026 at 12:01:14 UTC by @pks271 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76045
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

### Bug Description

A malformed ordinary transaction byte string can survive batch validation because `SingleBatch::check_batch()` only rejects empty txs, deposit txs, and pre-Isthmus `7702` txs; it never fully decodes ordinary envelopes. `AttributesQueue` then appends the malformed bytes unchanged into payload attributes. When the engine tries to start a safe build, the attr-bearing `engine_forkchoiceUpdated` call can legitimately return `PayloadStatus::Invalid` for an invalid transaction or invalid state transition, which the bundled derivation spec explicitly treats as a normal payload-processing failure that must cause the attrs to be dropped. That means this path is not limited to a malicious operator: it can also happen if the authenticated batch source is simply buggy, version-skewed, or emits malformed data during a normal upgrade or serialization mistake. Instead, the current implementation classifies that `Invalid` result as `Temporary`, so `EngineTask::execute()` retries the same task in-place forever with only `yield_now()`. Because `EngineProcessor` drains the task queue before it accepts the next request, the poisoned safe-build task starves later engine requests behind it.

Code path:

`crates/consensus/protocol/src/batch/single.rs:168#SingleBatch::check_batch` -> malformed tx bytes are accepted without envelope decode -> `crates/consensus/derive/src/stages/attributes_queue.rs:106#AttributesQueue::create_next_attributes` -> malformed bytes are forwarded unchanged into payload attributes -> `crates/consensus/engine/src/task_queue/tasks/build/task.rs:87#BuildTask::start_build` -> attr-bearing `engine_forkchoiceUpdated` returns `PayloadStatusEnum::Invalid` -> `crates/consensus/engine/src/task_queue/tasks/build/error.rs:58#BuildTaskError::severity` -> `EngineBuildError::InvalidPayload(_)` is classified as `Temporary` -> `crates/consensus/engine/src/task_queue/tasks/task.rs:248#EngineTask::execute` -> `Temporary` => `yield_now()` + `continue` on the same task -> `crates/consensus/service/src/actors/engine/engine_request_processor.rs:512#EngineProcessor::start` -> drains the queue before receiving the next request, so later engine work is starved

This also drifts from the bundled Base/OP derivation spec, which still requires invalid batch-derived payload attributes to be dropped rather than retried in place:

* `docs/specs/pages/protocol/consensus/derivation.md:821`

```md
If a payload attributes created from a batch cannot be inserted into the chain because of a validation error (i.e. there
was an invalid transaction or state transition in the block) the batch should be dropped & the safe head should not be
advanced.
```

* `docs/specs/pages/protocol/consensus/derivation.md:843`

```md
- On RPC-type errors the payload attributes processing should be re-attempted in a future step.
- On payload processing errors the attributes must be dropped, and the forkchoice state must be left unchanged.
```

## Impact

A malicious, compromised, or simply buggy authenticated batch source can publish one malformed batch that survives derivation validation and pins the consensus node on a single poisoned safe-build task. This does not require an actively malicious operator; the same behavior can arise from a serializer bug, version skew, or upgrade-edge batch that the EL rejects as invalid. The node keeps re-sending the same attr-bearing forkchoice update without drop or backoff, amplifying CPU work and preventing later engine requests from progressing.

## Recommendation

Fully decode ordinary batch transaction envelopes during batch validation and treat attr-bearing `InvalidPayload` responses as payload-processing failures rather than temporary retry conditions. Drop the bad attrs, keep forkchoice unchanged, and continue with the next batch or deposits-only fallback; add a regression test that invalid batch attrs are not retried in place.

## Proof of Concept

File: `base-base/crates/consensus/service/tests/actors/batch_invalid_attrs.rs`

Run the test: `cargo test --config 'target.x86_64-unknown-linux-gnu.rustflags=["-C","link-arg=-fuse-ld=lld"]' -p base-consensus-node --test integration authenticated_malformed_batch_transactions_reach_el_as_invalid_attrs_and_starve_engine -- --nocapture`

```rust
use std::{
    sync::{Arc, Mutex},
    time::Duration,
};

use alloy_consensus::Header;
use alloy_genesis::ChainConfig;
use alloy_eips::{BlockId, BlockNumHash, BlockNumberOrTag, eip2718::Decodable2718};
use alloy_primitives::{Address, B256, Bloom, FixedBytes, StorageKey, U256};
use alloy_provider::{EthGetBlock, Network, RpcWithBlock, network::Ethereum};
use alloy_rpc_types_engine::{
    ExecutionPayloadBodiesV1, ExecutionPayloadEnvelopeV2, ExecutionPayloadInputV2,
    ExecutionPayloadV1, ExecutionPayloadV3, ForkchoiceState, ForkchoiceUpdated, PayloadId,
    PayloadStatus, PayloadStatusEnum,
};
use alloy_rpc_types_eth::{Block as RpcBlock, EIP1186AccountProofResponse};
use alloy_transport::TransportResult;
use alloy_transport_http::Http;
use async_trait::async_trait;
use base_common_consensus::BaseTxEnvelope;
use base_common_network::Base;
use base_common_provider::BaseEngineApi;
use base_common_rpc_types::Transaction as OpTransaction;
use base_common_rpc_types_engine::{
    BaseExecutionPayload, BaseExecutionPayloadEnvelope, BaseExecutionPayloadEnvelopeV3,
    BaseExecutionPayloadEnvelopeV4, BaseExecutionPayloadEnvelopeV5, BaseExecutionPayloadV4,
    BasePayloadAttributes,
};
use base_consensus_derive::{
    AttributesQueue, Pipeline, PipelineError, PipelineErrorKind, Signal, SignalReceiver,
    StatefulAttributesBuilder, StepResult,
    test_utils::{TestChainProvider, TestSystemConfigL2Fetcher, new_test_attributes_provider},
};
use base_consensus_engine::{
    Engine,
    test_utils::{TestEngineStateBuilder, test_engine_client_builder},
};
use base_consensus_genesis::{RollupConfig, SystemConfig};
use base_consensus_node::{
    DerivationActor, DerivationActorRequest, DerivationClientError, EngineActorRequest,
    EngineDerivationClient, EngineProcessingRequest, EngineProcessor, EngineRequestReceiver, NodeActor,
    QueuedDerivationEngineClient,
};
use base_consensus_safedb::DisabledSafeDB;
use base_protocol::{AttributesWithParent, BatchValidity, BlockInfo, L1BlockInfoBedrock, L2BlockInfo, SingleBatch};
use tokio::sync::{mpsc, watch};
use tokio::time::timeout;
use tokio_util::sync::CancellationToken;

#[derive(Debug, Default)]
struct NoopDerivationClient;

#[async_trait]
impl EngineDerivationClient for NoopDerivationClient {
    async fn notify_sync_completed(&self, _: L2BlockInfo) -> Result<(), DerivationClientError> {
        Ok(())
    }

    async fn send_new_engine_safe_head(
        &self,
        _: L2BlockInfo,
    ) -> Result<(), DerivationClientError> {
        Ok(())
    }

    async fn send_signal(&self, _: Signal) -> Result<(), DerivationClientError> {
        Ok(())
    }
}

fn valid_fcu() -> ForkchoiceUpdated {
    ForkchoiceUpdated {
        payload_status: PayloadStatus {
            status: PayloadStatusEnum::Valid,
            latest_valid_hash: Some(FixedBytes([0x22; 32])),
        },
        payload_id: None,
    }
}

fn invalid_build_fcu() -> ForkchoiceUpdated {
    ForkchoiceUpdated {
        payload_status: PayloadStatus {
            status: PayloadStatusEnum::Invalid {
                validation_error: "invalid derived attrs".to_string(),
            },
            latest_valid_hash: Some(FixedBytes([0x33; 32])),
        },
        payload_id: Some(PayloadId::new([0x44; 8])),
    }
}

fn l1_info_deposit_tx() -> Vec<u8> {
    use alloy_eips::eip2718::Encodable2718;

    BaseTxEnvelope::from(base_common_consensus::TxDeposit {
        input: L1BlockInfoBedrock::default().encode_calldata(),
        ..Default::default()
    })
    .encoded_2718()
}

fn bedrock_payload(block_number: u64, parent_hash: B256) -> BaseExecutionPayload {
    BaseExecutionPayload::V1(ExecutionPayloadV1 {
        parent_hash,
        fee_recipient: Address::ZERO,
        state_root: FixedBytes::ZERO,
        receipts_root: FixedBytes::ZERO,
        logs_bloom: Bloom::ZERO,
        prev_randao: FixedBytes::ZERO,
        block_number,
        gas_limit: 30_000_000,
        gas_used: 0,
        timestamp: block_number,
        extra_data: Default::default(),
        base_fee_per_gas: U256::ZERO,
        block_hash: B256::with_last_byte(block_number as u8),
        transactions: vec![l1_info_deposit_tx().into()],
    })
}

fn parent_block(epoch_hash: B256) -> L2BlockInfo {
    L2BlockInfo {
        block_info: BlockInfo {
            number: 10,
            hash: B256::from([0x10; 32]),
            parent_hash: B256::from([0x0f; 32]),
            timestamp: 20,
        },
        l1_origin: BlockNumHash { number: 0, hash: epoch_hash },
        seq_num: 0,
    }
}

#[derive(Debug)]
struct OneShotPipeline {
    cfg: Arc<RollupConfig>,
    derived: Option<AttributesWithParent>,
    origin: BlockInfo,
}

impl OneShotPipeline {
    fn new(cfg: Arc<RollupConfig>, derived: AttributesWithParent, origin: BlockInfo) -> Self {
        Self { cfg, derived: Some(derived), origin }
    }
}

impl Iterator for OneShotPipeline {
    type Item = AttributesWithParent;

    fn next(&mut self) -> Option<Self::Item> {
        self.derived.take()
    }
}

impl base_consensus_derive::OriginProvider for OneShotPipeline {
    fn origin(&self) -> Option<BlockInfo> {
        Some(self.origin)
    }
}

#[async_trait]
impl SignalReceiver for OneShotPipeline {
    async fn signal(&mut self, _signal: Signal) -> Result<(), PipelineErrorKind> {
        Ok(())
    }
}

#[async_trait]
impl Pipeline for OneShotPipeline {
    fn peek(&self) -> Option<&AttributesWithParent> {
        self.derived.as_ref()
    }

    async fn step(&mut self, _cursor: L2BlockInfo) -> StepResult {
        if self.derived.is_some() {
            StepResult::PreparedAttributes
        } else {
            StepResult::StepFailed(PipelineError::NotEnoughData.temp())
        }
    }

    fn rollup_config(&self) -> &RollupConfig {
        self.cfg.as_ref()
    }

    async fn system_config_by_number(
        &mut self,
        _number: u64,
    ) -> Result<SystemConfig, PipelineErrorKind> {
        Ok(SystemConfig::default())
    }
}

#[derive(Debug, Clone)]
struct RecordingInvalidPayloadEngineClient {
    inner: base_consensus_engine::test_utils::MockEngineClient,
    recorded_payload_attributes: Arc<Mutex<Vec<BasePayloadAttributes>>>,
}

impl RecordingInvalidPayloadEngineClient {
    fn new(cfg: Arc<RollupConfig>, head: L2BlockInfo) -> Self {
        let inner = test_engine_client_builder()
            .with_config(cfg)
            .with_block_info_by_tag(BlockNumberOrTag::Latest, head)
            .with_fork_choice_updated_v2_response(valid_fcu())
            .with_fork_choice_updated_v3_response(valid_fcu())
            .with_new_payload_v1_response(PayloadStatus {
                status: PayloadStatusEnum::Valid,
                latest_valid_hash: Some(FixedBytes([0x55; 32])),
            })
            .build();

        Self { inner, recorded_payload_attributes: Arc::new(Mutex::new(Vec::new())) }
    }

    fn recorded_payload_attributes(&self) -> Vec<BasePayloadAttributes> {
        self.recorded_payload_attributes.lock().unwrap().clone()
    }

    async fn new_payload_v1_calls(&self) -> usize {
        self.inner.new_payload_v1_calls().await
    }
}

#[async_trait]
impl base_consensus_engine::EngineClient for RecordingInvalidPayloadEngineClient {
    fn cfg(&self) -> &RollupConfig {
        self.inner.cfg()
    }

    fn get_l1_block(
        &self,
        block: BlockId,
    ) -> EthGetBlock<<Ethereum as Network>::BlockResponse> {
        self.inner.get_l1_block(block)
    }

    fn get_l2_block(
        &self,
        block: BlockId,
    ) -> EthGetBlock<<Base as Network>::BlockResponse> {
        self.inner.get_l2_block(block)
    }

    fn get_proof(
        &self,
        address: Address,
        keys: Vec<StorageKey>,
    ) -> RpcWithBlock<(Address, Vec<StorageKey>), EIP1186AccountProofResponse> {
        self.inner.get_proof(address, keys)
    }

    async fn new_payload_v1(&self, payload: ExecutionPayloadV1) -> TransportResult<PayloadStatus> {
        self.inner.new_payload_v1(payload).await
    }

    async fn l2_block_by_label(
        &self,
        numtag: BlockNumberOrTag,
    ) -> Result<Option<RpcBlock<OpTransaction>>, base_consensus_engine::EngineClientError> {
        self.inner.l2_block_by_label(numtag).await
    }

    async fn l2_block_info_by_label(
        &self,
        numtag: BlockNumberOrTag,
    ) -> Result<Option<L2BlockInfo>, base_consensus_engine::EngineClientError> {
        self.inner.l2_block_info_by_label(numtag).await
    }
}

#[async_trait]
impl BaseEngineApi<Base, Http<base_consensus_engine::HyperAuthClient>>
    for RecordingInvalidPayloadEngineClient
{
    async fn new_payload_v2(
        &self,
        payload: ExecutionPayloadInputV2,
    ) -> TransportResult<PayloadStatus> {
        self.inner.new_payload_v2(payload).await
    }

    async fn new_payload_v3(
        &self,
        payload: ExecutionPayloadV3,
        parent_beacon_block_root: B256,
    ) -> TransportResult<PayloadStatus> {
        self.inner.new_payload_v3(payload, parent_beacon_block_root).await
    }

    async fn new_payload_v4(
        &self,
        payload: BaseExecutionPayloadV4,
        parent_beacon_block_root: B256,
    ) -> TransportResult<PayloadStatus> {
        self.inner.new_payload_v4(payload, parent_beacon_block_root).await
    }

    async fn fork_choice_updated_v2(
        &self,
        fork_choice_state: ForkchoiceState,
        payload_attributes: Option<BasePayloadAttributes>,
    ) -> TransportResult<ForkchoiceUpdated> {
        if let Some(attrs) = payload_attributes {
            self.recorded_payload_attributes.lock().unwrap().push(attrs);
            Ok(invalid_build_fcu())
        } else {
            self.inner.fork_choice_updated_v2(fork_choice_state, None).await
        }
    }

    async fn fork_choice_updated_v3(
        &self,
        fork_choice_state: ForkchoiceState,
        payload_attributes: Option<BasePayloadAttributes>,
    ) -> TransportResult<ForkchoiceUpdated> {
        if let Some(attrs) = payload_attributes {
            self.recorded_payload_attributes.lock().unwrap().push(attrs);
            Ok(invalid_build_fcu())
        } else {
            self.inner.fork_choice_updated_v3(fork_choice_state, None).await
        }
    }

    async fn get_payload_v2(
        &self,
        payload_id: PayloadId,
    ) -> TransportResult<ExecutionPayloadEnvelopeV2> {
        self.inner.get_payload_v2(payload_id).await
    }

    async fn get_payload_v3(
        &self,
        payload_id: PayloadId,
    ) -> TransportResult<BaseExecutionPayloadEnvelopeV3> {
        self.inner.get_payload_v3(payload_id).await
    }

    async fn get_payload_v4(
        &self,
        payload_id: PayloadId,
    ) -> TransportResult<BaseExecutionPayloadEnvelopeV4> {
        self.inner.get_payload_v4(payload_id).await
    }

    async fn get_payload_v5(
        &self,
        payload_id: PayloadId,
    ) -> TransportResult<BaseExecutionPayloadEnvelopeV5> {
        self.inner.get_payload_v5(payload_id).await
    }

    async fn get_payload_bodies_by_hash_v1(
        &self,
        block_hashes: Vec<B256>,
    ) -> TransportResult<ExecutionPayloadBodiesV1> {
        self.inner.get_payload_bodies_by_hash_v1(block_hashes).await
    }

    async fn get_payload_bodies_by_range_v1(
        &self,
        start: u64,
        count: u64,
    ) -> TransportResult<ExecutionPayloadBodiesV1> {
        self.inner.get_payload_bodies_by_range_v1(start, count).await
    }

    async fn get_client_version_v1(
        &self,
        client_version: alloy_rpc_types_engine::ClientVersionV1,
    ) -> TransportResult<Vec<alloy_rpc_types_engine::ClientVersionV1>> {
        self.inner.get_client_version_v1(client_version).await
    }

    async fn exchange_capabilities(
        &self,
        capabilities: Vec<String>,
    ) -> TransportResult<Vec<String>> {
        self.inner.exchange_capabilities(capabilities).await
    }
}

#[tokio::test(flavor = "multi_thread")]
async fn authenticated_malformed_batch_transactions_reach_el_as_invalid_attrs_and_starve_engine() {
    // Step 1: build a valid derivation context and authenticated batch envelope.
    let cfg = Arc::new({
        let mut cfg = RollupConfig::default();
        cfg.block_time = 2;
        cfg.seq_window_size = 3600;
        cfg.max_sequencer_drift = 600;
        cfg.hardforks.ecotone_time = Some(23);
        cfg
    });

    let epoch_header = Header { timestamp: 0, ..Default::default() };
    let epoch_hash = epoch_header.hash_slow();
    let parent = parent_block(epoch_hash);
    let epoch_block = BlockInfo {
        number: 0,
        hash: epoch_hash,
        parent_hash: B256::ZERO,
        timestamp: 0,
    };
    let inclusion_block = BlockInfo {
        number: 1,
        hash: B256::from([0x77; 32]),
        parent_hash: epoch_hash,
        timestamp: 0,
    };

    // Step 2: create malformed tx bytes that fail envelope decoding, then prove
    // they still pass batch-level validation.
    let malformed_batch_tx = alloy_primitives::Bytes::from(vec![0x01]);
    assert!(
        BaseTxEnvelope::decode_2718(&mut &malformed_batch_tx[..]).is_err(),
        "tx bytes must be malformed as a typed envelope"
    );

    let batch = SingleBatch {
        parent_hash: parent.block_info.hash,
        epoch_num: epoch_block.number,
        epoch_hash,
        timestamp: parent.block_info.timestamp + cfg.block_time,
        transactions: vec![malformed_batch_tx.clone()],
    };

    assert_eq!(
        batch.check_batch(cfg.as_ref(), &[epoch_block], parent, &inclusion_block),
        BatchValidity::Accept,
        "authenticated batch data should pass batch-level validation before EL decoding"
    );

    // Step 3: derive payload attributes from that batch and prove the malformed
    // bytes are forwarded unchanged into the EL-facing attrs.
    let mut system_config_fetcher = TestSystemConfigL2Fetcher::default();
    system_config_fetcher.insert(parent.block_info.number, SystemConfig::default());

    let mut chain_provider = TestChainProvider::default();
    chain_provider.insert_header(epoch_hash, epoch_header);

    let attributes_builder = StatefulAttributesBuilder::new(
        Arc::clone(&cfg),
        Arc::new(ChainConfig::default()),
        system_config_fetcher,
        chain_provider,
    );
    let prev = new_test_attributes_provider(Some(inclusion_block), vec![]);
    let mut attributes_queue = AttributesQueue::new(Arc::clone(&cfg), prev, attributes_builder);

    let payload_attributes = attributes_queue
        .create_next_attributes(batch.clone(), parent)
        .await
        .expect("batch should be transformed into payload attributes");

    let txs = payload_attributes
        .transactions
        .clone()
        .expect("derived payload attributes must contain transactions");
    assert_eq!(txs.len(), 2, "expected l1-info tx plus malformed user tx");
    assert_eq!(txs[1], malformed_batch_tx, "malformed batch tx bytes must be forwarded unchanged");

    // Step 4: start the engine actor and derivation actor, then enqueue the
    // poisoned safe signal followed by a normal unsafe payload.
    let derived_attributes =
        AttributesWithParent::new(payload_attributes.clone(), parent, Some(inclusion_block), true);
    let derivation_pipeline =
        OneShotPipeline::new(Arc::clone(&cfg), derived_attributes, inclusion_block);

    let client = Arc::new(RecordingInvalidPayloadEngineClient::new(Arc::clone(&cfg), parent));
    let initial_state = TestEngineStateBuilder::new()
        .with_unsafe_head(parent)
        .with_safe_head(parent)
        .with_finalized_head(parent)
        .build();
    let (state_tx, state_rx) = watch::channel(initial_state);
    let (queue_tx, _) = watch::channel(0usize);
    let engine = Engine::new(initial_state, state_tx, queue_tx);

    let processor = EngineProcessor::new(
        Arc::clone(&client),
        Arc::clone(&cfg),
        NoopDerivationClient,
        engine,
        None,
        None,
        false,
    );
    let (req_tx, req_rx) = mpsc::channel(128);
    let processor_handle = processor.start(req_rx);

    state_rx
        .clone()
        .wait_for(|state| state.sync_state.unsafe_head().block_info.number == 10)
        .await
        .expect("bootstrap did not seed unsafe head");

    let (derivation_req_tx, derivation_req_rx) = mpsc::channel(8);
    let (engine_actor_req_tx, mut engine_actor_req_rx) = mpsc::channel(128);
    let derivation_client = QueuedDerivationEngineClient::new(engine_actor_req_tx);
    let derivation_actor = DerivationActor::new(
        derivation_client,
        CancellationToken::new(),
        derivation_req_rx,
        derivation_pipeline,
        Arc::new(DisabledSafeDB),
    );
    let derivation_handle = tokio::spawn(async move { derivation_actor.start(()).await });

    derivation_req_tx
        .send(DerivationActorRequest::ProcessEngineSyncCompletionRequest(Box::new(parent)))
        .await
        .expect("failed to prime derivation actor");

    let safe_signal = timeout(Duration::from_secs(1), async {
        loop {
            let req = engine_actor_req_rx.recv().await.expect("engine actor channel closed");
            if let EngineActorRequest::ProcessSafeL2SignalRequest(signal) = req {
                return signal;
            }
        }
    })
    .await
    .expect("derivation did not emit safe signal in time");

    req_tx
        .send(EngineProcessingRequest::ProcessSafeL2Signal(safe_signal))
        .await
        .expect("failed to enqueue derivation-produced safe signal");

    req_tx
        .send(EngineProcessingRequest::ProcessUnsafeL2Block(Box::new(
            BaseExecutionPayloadEnvelope {
                parent_beacon_block_root: None,
                execution_payload: bedrock_payload(11, parent.block_info.hash),
            },
        )))
        .await
        .expect("failed to queue follow-up unsafe payload");

    // Step 5: prove the poisoned safe task starves the queue so the follow-up
    // unsafe payload never reaches engine_newPayload.
    let advanced = timeout(Duration::from_millis(200), async {
        let mut rx = state_rx.clone();
        rx.wait_for(|state| state.sync_state.unsafe_head().block_info.number == 11)
            .await
            .ok();
    })
    .await;
    assert!(
        advanced.is_err(),
        "unsafe payload should remain unprocessed because invalid derived attrs are retried forever"
    );
    assert_eq!(
        client.new_payload_v1_calls().await,
        0,
        "follow-up unsafe insert should never reach engine_newPayload while drain() is stuck"
    );

    // Step 6: prove the same invalid attrs are retried in-place at least five
    // times and print the observed retry count.
    let recorded = timeout(Duration::from_secs(1), async {
        loop {
            let recorded = client.recorded_payload_attributes();
            if recorded.len() >= 5 {
                return recorded;
            }
            tokio::task::yield_now().await;
        }
    })
    .await
    .expect("attribute-bearing FCU should be retried at least 5 times after EL returns Invalid");
    println!("recorded_invalid_attr_retries={}", recorded.len());
    assert!(
        recorded.len() >= 5,
        "attribute-bearing FCU should be retried at least 5 times after EL returns Invalid"
    );
    let first = recorded.first().expect("at least one FCU-with-attrs call should be recorded");
    let first_txs = first
        .transactions
        .clone()
        .expect("recorded payload attributes must include transactions");
    assert_eq!(first_txs[1], batch.transactions[0], "EL must receive the malformed batch tx bytes");

    drop(derivation_req_tx);
    drop(req_tx);
    derivation_handle.abort();
    processor_handle.abort();
}
```

Expected result:

```
running 1 test
recorded_invalid_attr_retries=51685
test actors::batch_invalid_attrs::authenticated_malformed_batch_transactions_reach_el_as_invalid_attrs_and_starve_engine ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 14 filtered out; finished in 0.21s
```
