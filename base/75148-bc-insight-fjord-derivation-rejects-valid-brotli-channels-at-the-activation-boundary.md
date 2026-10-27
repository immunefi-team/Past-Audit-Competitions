# 75148 bc insight fjord derivation rejects valid brotli channels at the activation boundary

## 75148 \[BC-Insight] Fjord derivation rejects valid Brotli channels at the activation boundary

Submitted on Apr 27th 2026 at 13:59:56 UTC by @BoatmanKharon for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75148
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.24>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

### Description

## Fjord derivation rejects valid Brotli channels at the activation boundary

### Summary

The Fjord upgrade activates derivation changes from the timestamp of the L1 block currently being processed. The channel reader applies that rule when it decides whether Fjord channel parameters should be used. `BatchReader::next_batch` then performs a second Brotli gate against the decoded batch's L2 timestamp instead of the L1 derivation context.

That mismatch causes valid post-Fjord Brotli channels to be discarded if they contain a batch whose L2 timestamp is still pre-Fjord.

### Affected Components

* `crates/consensus/derive/src/stages/channel/channel_reader.rs`
* `crates/consensus/protocol/src/batch/reader.rs`
* `docs/specs/pages/upgrades/fjord/derivation.md`

### Technical Detail

The derivation pipeline selects Fjord channel parameters from the current L1 origin:

```rust
let origin = self.prev.origin().ok_or(PipelineError::MissingOrigin.crit())?;
let max_rlp_bytes_per_channel = if self.cfg.is_fjord_active(origin.timestamp) {
    RollupConfig::MAX_RLP_BYTES_PER_CHANNEL_FJORD
} else {
    RollupConfig::MAX_RLP_BYTES_PER_CHANNEL_BEDROCK
};
```

That is consistent with the Fjord derivation specification: derivation changes apply when the pipeline is processing data from an L1 block whose timestamp is at or after the activation time.

The decoded channel is then passed into `BatchReader::next_batch`, which performs an additional Brotli check:

```rust
if self.brotli_used && !cfg.is_fjord_active(batch.timestamp()) {
    return None;
}
```

`batch.timestamp()` is the L2 timestamp embedded in the decoded batch. It is not the same value as the L1 timestamp that governs derivation activation. Around the fork boundary, those values can legitimately differ.

As a result, the node can enter the Fjord derivation path, accept Brotli channel framing, and then discard the channel solely because the decoded L2 batch timestamp has not yet crossed the same boundary.

### Impact

This is a protocol-transition derivation bug.

A node following this implementation can reject spec-valid Brotli channel data during the Fjord activation window. The direct consequence is liveness loss in derivation: valid channel data is dropped, the reader advances to the next channel, and the node waits for replacement data that is not actually required by the protocol.

I would rate this as a medium-severity issue. It is a concrete consensus/derivation correctness failure in in-scope code, but I am not claiming direct funds risk or a demonstrated network-wide outage.

### Observed on Current Public Code

This issue is present in the public `base/base` repository on `main` as of commit `dd3b5eb575be368280a5cd4e14be57b78c30712e` dated April 26, 2026.

### Proof of Concept

## Fjord Brotli activation mismatch poc

### Overview

This PoC demonstrates that the node accepts a Brotli channel once the decoded batch timestamp itself has crossed the Fjord boundary, but rejects the same class of channel when the L1 derivation context is already post-Fjord and the decoded batch timestamp is still pre-Fjord.

That is the condition that proves the activation check is using the wrong clock.

### Test Code

Insert the following into `crates/consensus/derive/src/stages/channel/channel_reader.rs` under the existing test module.

```rust
use alloy_rlp::Decodable;
use miniz_oxide::inflate::decompress_to_vec_zlib;

fn new_brotli_batch_data() -> Bytes {
    let raw = new_compressed_batch_data();
    let decompressed = decompress_to_vec_zlib(&raw).unwrap();

    let params = brotli::enc::BrotliEncoderParams::default();
    let mut compressed = vec![];
    let mut input = &decompressed[..];
    brotli::BrotliCompress(&mut input, &mut compressed, &params).unwrap();

    let mut out = vec![BatchReader::CHANNEL_VERSION_BROTLI];
    out.extend_from_slice(&compressed);
    out.into()
}

fn fixture_batch_timestamp() -> u64 {
    let raw = new_compressed_batch_data();
    let decompressed = decompress_to_vec_zlib(&raw).unwrap();
    let mut reader = decompressed.as_slice();
    let bytes = Bytes::decode(&mut reader).unwrap();
    let batch = Batch::decode(&mut bytes.as_ref(), &RollupConfig::default()).unwrap();
    batch.timestamp()
}

#[tokio::test]
async fn test_next_batch_accepts_brotli_channel_after_fjord_batch_timestamp() {
    let batch_timestamp = fixture_batch_timestamp();
    let fjord_time = batch_timestamp;
    let config = Arc::new(RollupConfig {
        hardforks: HardForkConfig { fjord_time: Some(fjord_time), ..Default::default() },
        ..Default::default()
    });
    let mut mock = TestChannelReaderProvider::new(vec![Ok(Some(new_brotli_batch_data()))]);
    mock.block_info = Some(BlockInfo { timestamp: fjord_time, ..Default::default() });

    let mut reader = ChannelReader::new(mock, config);
    let res = reader.next_batch().await.unwrap();
    assert!(matches!(res, Batch::Span(_)));
    assert!(reader.next_batch.is_some());
}

#[tokio::test]
async fn test_next_batch_rejects_post_fjord_origin_brotli_channel_if_batch_timestamp_is_pre_fjord(
) {
    let batch_timestamp = fixture_batch_timestamp();
    let fjord_time = batch_timestamp + 1;
    let config = Arc::new(RollupConfig {
        hardforks: HardForkConfig { fjord_time: Some(fjord_time), ..Default::default() },
        ..Default::default()
    });
    let mut mock = TestChannelReaderProvider::new(vec![Ok(Some(new_brotli_batch_data()))]);
    mock.block_info = Some(BlockInfo { timestamp: fjord_time, ..Default::default() });

    let mut reader = ChannelReader::new(mock, config);
    assert_eq!(reader.next_batch().await, Err(PipelineError::NotEnoughData.temp()));
    assert!(
        reader.next_batch.is_none(),
        "the reader drops the whole channel even though the L1 origin is already post-Fjord"
    );
}
```

### Commands

Run from the repository root:

```bash
cargo test -p base-consensus-derive test_next_batch_accepts_brotli_channel_after_fjord_batch_timestamp -- --nocapture
cargo test -p base-consensus-derive test_next_batch_rejects_post_fjord_origin_brotli_channel_if_batch_timestamp_is_pre_fjord -- --nocapture
```

### Expected Result

* `test_next_batch_accepts_brotli_channel_after_fjord_batch_timestamp` passes.
* `test_next_batch_rejects_post_fjord_origin_brotli_channel_if_batch_timestamp_is_pre_fjord` passes.

The second test is the proof of impact: the current implementation drops a Brotli channel even though the derivation pipeline is already operating in a post-Fjord L1 context.
