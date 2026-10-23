# 75378 bc low non exact eip 2718 comparison lets safe head consolidation diverge from exact l1 derivation

**Submitted on Apr 28th 2026 at 20:10:49 UTC by @QED for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75378
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Unintended chain split (network partition)

## Description

## Brief/Intro

`base-consensus` safe-head consolidation compares L1-derived transaction bytes to an unsafe block by decoding the attribute bytes with non-exact EIP-2718 decoding. A derived transaction byte string `encode_2718(T) || trailing_bytes` compares equal to an unsafe block containing only `T`. The real `ConsolidateTask::execute` path then advances the safe head to the clean unsafe block even though the exact payload-building path rejects the same L1-derived bytes with `UnexpectedLength`.

## Vulnerability Details

### Invariant the code claims

The Base specs define unsafe block consolidation as an exact match between L1-derived payload attributes and the oldest unsafe block:

```
docs/specs/pages/reference/glossary.md:696-700
Unsafe block consolidation is the process through which the rollup node attempts
to move the safe L2 head a block forward...
the node verifies that the payload attributes derived from the L1 chain match
the oldest unsafe L2 block exactly.
```

The derivation spec is explicit for transactions:

```
docs/specs/pages/protocol/consensus/derivation.md:792-800
The following fields ... are checked for equality with the L2 block:
  transactions_list (first length, then equality of each of the encoded transactions, including deposits)
```

The execution spec also says a non-empty payload-attribute transaction list must be used exactly:

```
docs/specs/pages/protocol/execution/index.md:206-211
If present and non-empty: the payload MUST be produced starting with this exact list of transactions.
```

### Root cause - semantic tx equality instead of exact byte equality

Single batches carry user transactions as opaque `Bytes`.

```rust
// crates/consensus/protocol/src/batch/single.rs:16-29
pub struct SingleBatch {
    ...
    pub transactions: Vec<Bytes>,
}
```

`SingleBatch::check_batch()` rejects empty transactions, deposits, and pre-Isthmus 7702 transactions, but it does not exact-decode each transaction byte string.

```rust
// crates/consensus/protocol/src/batch/single.rs:169-183
for tx in &self.transactions {
    if tx.is_empty() { return BatchValidity::Drop(...); }
    if tx.as_ref().first() == Some(&(OpTxType::Deposit as u8)) { return BatchValidity::Drop(...); }
    if !cfg.is_isthmus_active(self.timestamp)
        && tx.as_ref().first() == Some(&(OpTxType::Eip7702 as u8))
    {
        return BatchValidity::Drop(...);
    }
}
```

The attributes queue then copies those opaque bytes directly into payload attributes:

```rust
// crates/consensus/derive/src/stages/attributes_queue.rs:124-133
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

The vulnerable comparator decodes the bytes with `decode_2718` and never checks that the input slice was fully consumed.

```rust
// crates/consensus/engine/src/attributes.rs:143-170
for (attr_tx_bytes, block_tx) in attributes_txs.iter().zip(block_txs) {
    let Ok(attr_tx) = BaseTxEnvelope::decode_2718(&mut &attr_tx_bytes[..]) else {
        return AttributesMismatch::MalformedAttributesTransaction.into();
    };

    if &attr_tx != block_tx.inner.inner.inner() {
        return AttributesMismatch::TransactionContent(
            attr_tx.tx_hash(),
            block_tx.tx_hash(),
        )
        .into();
    }
}

Self::Match
```

That is a semantic transaction comparison. It is not the spec-required exact encoded transaction comparison.

### Exact payload paths reject the same bytes

The shared RPC attribute helper checks for unread bytes:

```rust
// crates/common/rpc-types-engine/src/attributes.rs:152-156
let mut buf = tx_bytes.as_ref();
let tx = BaseTxEnvelope::decode_2718(&mut buf).map_err(alloy_rlp::Error::from)?;
if !buf.is_empty() {
    return Err(alloy_rlp::Error::UnexpectedLength.into());
}
```

The payload builder also uses exact decoding:

```rust
// crates/execution/payload/src/payload.rs:105-113
let transactions = attributes
    .transactions
    .unwrap_or_default()
    .into_iter()
    .map(|data| {
        Decodable2718::decode_2718_exact(data.as_ref()).map(|tx| WithEncoded::new(data, tx))
    })
    .collect::<Result<_, _>>()?;
```

So the consolidation shortcut accepts bytes that the exact build/import path rejects.

### Safe-head transition

`ConsolidateTask` uses `AttributesMatch::check(...).is_match()` as the gate:

```rust
// crates/consensus/engine/src/task_queue/tasks/consolidate/task.rs:47-54
fn is_consistent_with_block(&self, cfg: &RollupConfig, block: &Block<Transaction>) -> bool {
    match self {
        Self::Attributes(attributes) => {
            crate::AttributesMatch::check(cfg, attributes, block).is_match()
        }
        ...
    }
}
```

If it matches, the task updates the safe head from the already-imported unsafe block:

```rust
// crates/consensus/engine/src/task_queue/tasks/consolidate/task.rs:190-221
if self.input.is_consistent_with_block(&self.cfg, &block) {
    match L2BlockInfo::from_block_and_genesis(...) {
        Ok(block_info) if !self.input.is_attributes_last_in_span() => {
            state.sync_state = state.sync_state.apply_update(EngineSyncStateUpdate {
                safe_head: Some(block_info),
                ..Default::default()
            });
            return Ok(());
        }
```

## Impact Details

**Severity: High - Unintended chain split (network partition).**

The bug causes two honest nodes running the same implementation to make different `safe_l2` decisions from the same canonical L1 batch data, depending only on whether the matching clean unsafe block was already locally available.

1. A node that already imported the clean unsafe block `T` through the unsafe block path / P2P gossip runs safe-head consolidation. The non-exact comparator accepts L1-derived bytes `encode_2718(T) || trailing_bytes`, and `ConsolidateTask` advances the safe head to the clean unsafe block.
2. A node that did not have that unsafe block, or a node rebuilding after reset, processes the same L1-derived attributes through the payload-building path. Exact EIP-2718 decoding rejects the transaction bytes with `UnexpectedLength`, so that node cannot derive and mark the same block safe from the same L1 data.

This is not merely acceptance of a malformed transaction in one helper. It violates the core consolidation invariant: the safe chain should be a deterministic function of authenticated L1-derived payload attributes. Here, the result also depends on local unsafe-block availability. The two-node localnet PoC below demonstrates this directly: node A has the clean unsafe block and advances `safe_l2` to the poisoned target, while node B is isolated from unsafe gossip, observes the same L1 batch block, and remains unable to import the target block.

The required attacker capability is control of the active batcher-authenticated L1 batch stream, or an equivalent batcher/signing fault. This limits ordinary external exploitability, but the affected input is still consensus input that honest rollup nodes must validate identically after batcher authentication. The vulnerability is that validation is not identical across the consolidation and exact payload-building paths.

## Link to Proof of Concept

<https://gist.github.com/a-qedaudit/c4969a4bf3dd323979c44dc1bcae413e>

## Proof of Concept

Two PoCs are included. All reproduction files are in the unlisted gist below, and the only external source needed is the public `base/base` repository.

### Consolidation harness

Unlisted gist:

```
https://gist.github.com/a-qedaudit/c4969a4bf3dd323979c44dc1bcae413e
```

Run:

```sh
git clone https://gist.github.com/a-qedaudit/c4969a4bf3dd323979c44dc1bcae413e cipsveaa-poc
cd cipsveaa-poc
cargo +1.93.0 run --release
```

The harness `Cargo.toml` depends on `https://github.com/base/base.git` at `v0.8.0-rc.28`, so it does not require a local audit checkout.

Observed output:

```
cipsveaa PoC: Base safe-head consolidation accepts non-exact tx bytes
audit repo: github.com/base/base @ v0.8.0-rc.28 / e3467a2048881213b56739a54a876efb9c6ea103

clean encoded tx length    : 55
poisoned attr tx length    : 67
trailing bytes appended    : 0xdeadbeef5341464548454144

BaseTxEnvelope::decode_2718 accepts poisoned bytes             : true
decoded tx equals clean block tx                               : true
bytes left unconsumed by non-exact decode                      : 0xdeadbeef5341464548454144
BaseTxEnvelope::decode_2718_exact on poisoned bytes            : Err(RlpError(UnexpectedLength))

real AttributesMatch::check(poisoned attrs, clean unsafe block): Match
BasePayloadAttributes::decoded_transactions on same bytes      : Err(RlpError(UnexpectedLength))

=== vulnerable consolidation decision ===
ConsolidateTask precheck AttributesMatch                     : Match
ConsolidateTask precheck L2BlockInfo                         : Ok(L2BlockInfo { ... number: 1 ... })
safe head before ConsolidateTask::execute: BlockInfo { ... number: 0 ... }
safe head after  ConsolidateTask::execute: BlockInfo { ... number: 1 ... }
safe head advanced by task               : true

control: exact attrs vs clean unsafe block                     : Match

PoC result: vulnerable match accepted malformed L1-derived bytes that the
exact payload decoder rejects. A safe-head consolidation can therefore mark
a clean unsafe block as safe even when the authenticated L1 attributes encode
different raw transaction bytes.
```

The consolidation harness uses Base's real `ConsolidateTask::execute`, `EngineState`, `AttributesMatch::check`, `BasePayloadAttributes::decoded_transactions`, and `BaseTxEnvelope` decoders. The only mock is Base's own `MockEngineClient`, used to return the already-imported unsafe block to the consolidation task.

### Localnet E2E

Run:

```sh
git clone https://gist.github.com/a-qedaudit/c4969a4bf3dd323979c44dc1bcae413e cipsveaa-poc
cd cipsveaa-poc
git clone https://github.com/base/base.git base
git -C base checkout e3467a2048881213b56739a54a876efb9c6ea103
BASE_REPO="$PWD/base" ./run_localnet_poc.sh
./check_localnet_poc.sh
```

The localnet harness applies `instrumentation.patch` to the audit checkout. The patch adds a devnet test, one test-only `DevnetBuilder` knob that makes the background batcher sign with a non-authorized key, and one test-only isolated second validator that is not connected to builder EL p2p, flashblocks, or consensus unsafe-block gossip. The patch does not modify `AttributesMatch`, derivation validation, payload construction, safe-head update logic, transaction decoding, or Engine API behavior. The malicious batch is still submitted by the configured devnet batcher key to the real batch inbox.

`run_localnet_poc.sh` writes the evidence excerpt to `localnet.out` and keeps the full raw service log at `localnet.out.raw`.

Observed localnet evidence:

```
cipsveaa two-node localnet E2E: real L1 + builder + two Base validators
normal in-process batcher signer   : 0x...
manual malicious batcher signer    : 0x...
node_a validator                   : connected to unsafe block gossip
node_b validator                   : isolated from unsafe block gossip
node_b initial unsafe/safe         : unsafe #0 safe #0
unsafe target block                : #17 0x...
clean user tx included             : 0x...
node_a had clean unsafe before L1  : yes
node_b had clean unsafe before L1  : no
node_a safe before malicious batch : #0 0x...
node_b safe before malicious batch : #0 0x...
poisoned target batch tx bytes     : ... -> ...
exact payload decoder on poison    : Err(RlpError(UnexpectedLength))
authorized L1 batch tx             : 0x...
authorized L1 batch block          : #48
node_a safe after malicious batch  : #17 0x...
node_b L1 head after malicious     : #59 0x...
node_b current L1 after malicious  : #58 0x...
node_b safe after malicious batch  : #0 0x...
node_b target block after batch    : missing
VERDICT: SAME_L1_DATA_DIFFERENT_SAFE_HEADS
VERDICT: LOCALNET_SAFE_HEAD_ADVANCED_ON_NON_EXACT_L1_TX_BYTES
```

This drives the full local path: real L1, real Base builder EL, two real Base validator EL/CL pairs, real L2 unsafe block gossip/import for node A, no unsafe-block gossip for node B, real authorized L1 batcher calldata, and live `optimism_syncStatus.safe_l2` divergence from the same L1 batch.

## Mitigation

Make safe-head transaction comparison exact.

The smallest fix is to replace the non-exact decode in `AttributesMatch::check_transactions()` with exact decoding:

```rust
let Ok(attr_tx) = BaseTxEnvelope::decode_2718_exact(attr_tx_bytes.as_ref()) else {
    return AttributesMismatch::MalformedAttributesTransaction.into();
};
```

An even stricter fix is to compare raw encoded bytes: re-encode the block transaction with `encode_2718` and require byte-for-byte equality with `attr_tx_bytes`. That directly implements the spec wording: equality of each encoded transaction.

Add a regression test where `attributes.transactions = [encode_2718(T) || trailing_bytes]` and `block.transactions = [T]`. Expected result: `AttributesMatch::check()` must return `MalformedAttributesTransaction` or `TransactionContent`, never `Match`, and `ConsolidateTask` must not advance the safe head via L1 consolidation.
