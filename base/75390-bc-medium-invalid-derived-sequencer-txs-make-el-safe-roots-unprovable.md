# 75390 bc medium invalid derived sequencer txs make el safe roots unprovable

**Submitted on Apr 28th 2026 at 22:00:15 UTC by @QED for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75390
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Unintended permanent chain split requiring hard fork (network partition requiring hard fork)

## Description

## Brief/Intro

`base-consensus` turns batcher-authenticated L1 data into exact `BasePayloadAttributes.transactions` with `no_tx_pool=true`. The execution payload builder still treats execution-invalid sequencer transactions as skippable candidates: it skips `InvalidTx` and continues. The proof executor does not; it executes the full derived transaction list strictly, and the Holocene proof driver retries failed payloads as deposit-only.

The trigger is batcher-authenticated L1 data containing an execution-invalid sequencer transaction before a later valid one. The same L1 input is reproducibly interpreted two ways by shipped components: EL imports a skip-and-continue safe block, while the proof path rejects the full list and computes a deposit-only replacement root. An honest proof client cannot validate the canonical EL root: the computed proof root differs from the claimed EL root and `Epilogue::validate` returns `InvalidClaim`. Even if the deposit-only proof root were surfaced to the proposer, the proposer's JIT equality check rejects it as `RootMismatch` against the EL canonical output root. In the localnet PoC, the later valid transaction calls `L2ToL1MessagePasser` and creates an EL-only withdrawal message that the deposit-only proof lineage does not contain.

## Vulnerability Details

### Derived batches become exact payload attributes

The derivation queue appends batch transactions verbatim and forces `no_tx_pool=true`:

```rust
// crates/consensus/derive/src/stages/attributes_queue.rs
let tx_count = batch.transactions.len();
let mut attributes = self.builder.prepare_payload_attributes(parent, batch.epoch()).await?;
attributes.no_tx_pool = Some(true);
match attributes.transactions {
    Some(ref mut txs) => txs.extend(batch.transactions),
    None => {
        if !batch.transactions.is_empty() {
            attributes.transactions = Some(batch.transactions);
        }
    }
}
```

### Root cause - EL skips invalid derived sequencer txs

`OpPayloadBuilderCtx::execute_sequencer_transactions()` treats attribute-supplied sequencer transactions with a mempool-style skip policy. `InvalidTx` is not fatal:

```rust
// crates/execution/payload/src/builder.rs
let gas_used = match builder.execute_transaction(sequencer_tx.clone()) {
    Ok(gas_used) => gas_used,
    Err(BlockExecutionError::Validation(BlockValidationError::InvalidTx {
        error,
        ..
    })) => {
        trace!(target: "payload_builder", %error, ?sequencer_tx, "Error in sequencer transaction, skipping.");
        continue;
    }
    Err(err) => {
        return Err(PayloadBuilderError::EvmExecutionError(Box::new(err)));
    }
};
```

When `no_tx_pool` is set, the builder freezes the result as deterministic:

```rust
// crates/execution/payload/src/builder.rs
if no_tx_pool {
    Ok(BuildOutcomeKind::Freeze(payload))
} else {
    Ok(BuildOutcomeKind::Better { payload })
}
```

This is incompatible with derived payloads. The transaction list is consensus input, not an optional candidate list.

### Proof path rejects the same list

The proof executor recovers and executes the full attribute list:

```rust
// crates/proof/executor/src/builder/core.rs
let transactions = attrs
    .recovered_transactions_with_encoded()
    .collect::<Result<Vec<_>, RecoveryError>>()
    .map_err(ExecutorError::Recovery)?;
let ex_result = executor.execute_block(transactions.iter())?;
```

The proof driver applies the Holocene invalid-payload rule by flushing the channel and retrying with only deposits:

```rust
// crates/proof/driver/src/core.rs
if cfg.is_holocene_active(attributes.payload_attributes.timestamp) {
    self.pipeline.signal(Signal::FlushChannel).await?;

    attributes.transactions = attributes.transactions.map(|txs| {
        txs.into_iter()
            .filter(|tx| !tx.is_empty() && tx[0] == OpTxType::Deposit as u8)
            .collect::<Vec<_>>()
    });

    self.executor.update_safe_head(tip_cursor.l2_safe_head_header.clone());
    match self.executor.execute_payload(attributes.clone()).await {
```

### Proposer rejects the resulting proof root

The proposer performs a just-in-time equality check before submission:

```rust
// crates/proof/proposer/src/pipeline.rs
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

If the EL canonical output is the skip-and-continue block while the proof output is the Holocene deposit-only block, this check deterministically rejects honest proofs for the affected range.

### Proof client cannot validate the EL claim

The proof client also performs the final computed-root versus claimed-root check in `Epilogue::validate()`:

```rust
// crates/proof/client/src/epilogue.rs
if self.output_root != self.claimed_output_root {
    return Err(Box::new(FaultProofProgramError::InvalidClaim {
        computed: self.output_root,
        claimed: self.claimed_output_root,
    }));
}
```

For the affected block, the canonical EL output root is the claimed root. The honest proof computation produces the deposit-only root instead, so the proof client rejects the canonical EL claim before it can be honestly attested.

## Impact Details

**Severity: Critical - EL safe-state and honest proof output roots can deterministically diverge for the same canonical L1 batch data, halting finalization for the affected range unless operators coordinate a rule-changing repair/rederivation.**

Base is a single-sequencer L2, so a classical multi-validator "network partition" does not apply. The relevant axis is whether the EL safe state and the proof output root produced from the same canonical L1 batch data can be reconciled under the existing rules. They cannot: the EL skip-and-continue rule and the proof strict-execution + Holocene deposit-only rule disagree on the same input and produce different output roots. Under the program taxonomy, this maps to "unintended permanent chain split requiring hard fork" because both branches are reproducible from the same L1 input under different shipped components, and the affected range cannot be honestly finalized without changing one component's consensus rule or performing a controlled rederivation under patched rules.

A batcher-controlled or faulty batch stream can put an execution-invalid sequencer transaction before a later valid transaction. From the same L1 data:

1. The EL derived build path skips the invalid transaction and imports a safe block containing the later valid transaction.
2. The proof path rejects the full list, flushes the channel, and computes a deposit-only replacement block.
3. The two paths compute different block hashes and Base output roots.
4. The proof client rejects the canonical EL root as `InvalidClaim`; even if the deposit-only proof output were surfaced to the proposer, the proposer rejects it as `RootMismatch`. The affected EL output cannot be finalized by the honest proof pipeline.

The localnet PoC makes the later valid transaction a real withdrawal message: the EL safe block emits `MessagePassed` from the `L2ToL1MessagePasser` predeploy and stores `sentMessages[withdrawalHash] = true`. The deposit-only proof lineage strips the same user transaction, so that withdrawal message is absent from the proof lineage. This demonstrates concrete bridge-relevant state divergence, not only a transaction-list mismatch.

The direct trigger requires the active batcher key or an equivalent batcher fault. That precondition is still security-relevant: batcher-authenticated L1 data is the canonical derivation input, and the EL and proof stacks must make the same accept/reject decision for that input. Because the invalid transaction is committed in L1 batch data, repeated proof derivation reproduces the deposit-only lineage; repeated vulnerable EL derivation reproduces the skip-and-continue lineage.

This report does not claim TEE/ZK signature forgery. The failure mode is not merely local execution inconsistency: the divergent EL state is the canonical safe state observed by the node, while the honest proof pipeline cannot attest to its output root. If operators choose the proof-derived lineage, withdrawals created only in the EL lineage are unprovable. If operators choose the vulnerable EL lineage, the proof system cannot honestly attest to that output root without changing the proof execution rule.

## References

Audit base: `github.com/base/base` at `v0.8.0-rc.28` (`e3467a20`).

* `crates/consensus/derive/src/stages/attributes_queue.rs` - derived batches are appended as payload transactions and `no_tx_pool=true` is set.
* `crates/execution/payload/src/builder.rs` - EL builder skips `BlockValidationError::InvalidTx` in sequencer transactions.
* `crates/proof/executor/src/builder/core.rs` - proof executor executes the full derived transaction list strictly.
* `crates/proof/driver/src/core.rs` - Holocene proof fallback flushes the channel and retries deposit-only.
* `crates/proof/client/src/epilogue.rs` - proof client returns `InvalidClaim` when the computed proof output root differs from the claimed output root.
* `crates/proof/proposer/src/pipeline.rs` - proposer returns `SubmitAction::RootMismatch` when the proof output root differs from the EL canonical output root.

## Link to Proof of Concept

<https://gist.github.com/a-qedaudit/c18a901965f00f590f30ad12801150ae>

## Proof of Concept

A single localnet PoC is included. It drives the live L1 -> derivation -> EL safe-head path, then replays the same localnet payload through Base's production stateless proof executor and proof-client epilogue validation.

**Runnable artifacts (secret gist):** <https://gist.github.com/a-qedaudit/c18a901965f00f590f30ad12801150ae>

Contains only the localnet reproduction kit and instructions: `README.md`, `run_localnet_poc.sh`, `check_localnet_poc.sh`, and `instrumentation.patch`. Clone the gist and follow the reproduction steps below.

Run:

```sh
git clone https://gist.github.com/a-qedaudit/c18a901965f00f590f30ad12801150ae bij27aqs-poc
cd bij27aqs-poc
if [ ! -d ../bij27aqs-base/.git ]; then
  git clone https://github.com/base/base.git ../bij27aqs-base
fi
git -C ../bij27aqs-base checkout e3467a2048881213b56739a54a876efb9c6ea103
BASE_REPO="$PWD/../bij27aqs-base" ./run_localnet_poc.sh
./check_localnet_poc.sh
```

The localnet harness applies `instrumentation.patch` to the audit checkout. The patch adds a devnet test, one test-only `DevnetBuilder` knob that makes the background batcher sign with a non-authorized key, and proof-replay plumbing inside the test. `run_localnet_poc.sh` also checks that `BASE_REPO` is exactly `e3467a2048881213b56739a54a876efb9c6ea103`. It does not modify derivation validation, payload construction, transaction execution, output-root construction, or proof-client validation. The malicious batch is still submitted by the configured devnet batcher key to the real batch inbox.

Observed localnet evidence:

```
bij27aqs localnet E2E: authorized L1 batch -> real derivation -> real EL build
configured batcher address         : 0x976EA74026E726554dB657fA54763abd0C3a0aa9
normal in-process batcher signer   : 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
manual malicious batcher signer    : 0x976EA74026E726554dB657fA54763abd0C3a0aa9
batch inbox                        : 0x00bcC8554867297685A924F26b643A0146De823E
client safe before malicious batch : #0 0x47a89ecf2137d6f281417f0435895115179b4bcc22fec4e7d0ffb8d0271192c9
parent L1 origin                   : #29 0x044a81f20308735868c6fa8699ece00e38aef322bc41a7c33de09a932b3b16d0
sender nonce before L1 batch       : 0
derived L1 tx[0] nonce/hash        : 1 / 0x17a4308217ffb21e9b0005d80f2470f8952fbc50b4bf1d89b0a5f4357c60cac5
derived L1 tx[1] nonce/hash        : 0 / 0xab86ab6574769e585e6a9a1e954bd3ed181f780f42fb3753d1e29bcdcd8ce719
derived L1 tx[1] call target       : L2ToL1MessagePasser
proof strict first-tx outcome      : NonceTooHigh(tx=1,state=0)
proof Holocene retry user tx count : 0
proof Holocene retry withdrawal    : absent
authorized L1 calldata length      : 412 bytes
authorized L1 batch tx             : 0x5200b73f503208e1a7c5eaa8560bae0b8ce727c6596c63e87731914efd123fb7
client safe after malicious batch  : #1 0x400b2fd17f79b21014ae38a000d11e38b52b617dc41554e1ba719e02827ac17e
safe target block                  : #1 0x400b2fd17f79b21014ae38a000d11e38b52b617dc41554e1ba719e02827ac17e
safe target tx count               : 2
safe target deposit tx count       : 1
safe block contains nonce=1 tx     : false
safe block contains nonce=0 tx     : true
sender nonce after safe block      : 1
EL withdrawal tx status            : true
EL MessagePassed log count         : 1
EL withdrawal hash                 : 0xc287811beec2080e2b8cbbcfcef5fe32927457733001f0a19164cd1403636ba4
EL sentMessages[withdrawalHash]    : true
canonical EL output root           : 0x63851e69abb7ddead290a69deb79dc4306d7abc04e50c24ce34a867d601e46db
production proof strict result     : ExecutionError(Validation(InvalidTx { hash: 0x17a4308217ffb21e9b0005d80f2470f8952fbc50b4bf1d89b0a5f4357c60cac5, error: Base(NonceTooHigh { tx: 1, state: 0 }) }))
production proof Holocene tx count : 1
production proof block hash        : 0xbe7118ccef48c9e44868af2ffef81c2d9562853016bd3087649dc261f75cf43f
production proof output root       : 0xe4388d01aeca1b54571bc9830077e6b0664111919cb7026fa9fab4bd5e95f185
proof/EL output roots diverge      : true
proof client Epilogue::validate    : InvalidClaim { computed: 0xe4388d01aeca1b54571bc9830077e6b0664111919cb7026fa9fab4bd5e95f185, claimed: 0x63851e69abb7ddead290a69deb79dc4306d7abc04e50c24ce34a867d601e46db }
proof retry sentMessages entry     : false (deposit-only retry strips tx[1])
VERDICT: LOCALNET_EL_ONLY_WITHDRAWAL_MESSAGE_CREATED
VERDICT: EL_SAFE_ROOT_UNPROVABLE_BY_PRODUCTION_PROOF_CLIENT
```

The Holocene retry contains one deposit transaction and zero user transactions; the nonce-0 withdrawal transaction is stripped, which is why `proof Holocene retry user tx count` is 0 while `production proof Holocene tx count` is 1.

This drives the live ingress and EL side end to end: real L1, real Base builder EL, real Base validator EL, real Base consensus nodes, real authorized L1 batcher calldata, real derivation into payload attributes, and live `optimism_syncStatus.safe_l2` movement. It then applies the same Holocene deposit-only transformation used by the proof driver and replays the resulting payload through Base's production stateless proof executor and proof-client epilogue check, using the live EL safe block's canonical output root as the claimed root.

The result shows that the validator safe block is built from an authorized batch containing two user transactions while the resulting safe block executes only the later valid transaction. That transaction creates a withdrawal message in `L2ToL1MessagePasser` storage on the EL lineage. The production proof executor rejects the full derived list with `NonceTooHigh`, the Holocene deposit-only retry computes a different output root, and `Epilogue::validate` rejects the canonical EL root as `InvalidClaim`. This demonstrates an EL safe root that the honest proof client cannot attest, not only a transaction-list mismatch.

## Mitigation

Treat any execution-time `InvalidTx` from attribute-supplied sequencer transactions as a fatal derived-payload error when `no_tx_pool=true`. The EL derived build path should follow the same Holocene invalid-payload rule as the proof driver: reject the full list, flush the channel where applicable, and retry deposit-only. Keep skip-and-continue behavior only for optional mempool transactions.

Add a regression test with a derived transaction list `[nonce=1, nonce=0]` from the same sender. The EL and proof paths must produce the same transaction set, block hash, and output root.
