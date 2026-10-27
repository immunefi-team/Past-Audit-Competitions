# 75437 bc medium jovian min base fee omission lets consolidation promote a non canonical unsafe block as safe

**Submitted on Apr 29th 2026 at 05:42:48 UTC by @QED for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75437
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Unintended chain split (network partition)

## Description

### Brief/Intro

`base-consensus` safe-head consolidation ignores the Jovian `min_base_fee` field when comparing L1-derived payload attributes to an already-imported unsafe block. If the unsafe block header encodes the same EIP-1559 elasticity and denominator but a different Jovian `min_base_fee`, `AttributesMatch::check()` returns `Match`. The real `ConsolidateTask::execute()` path then advances the safe head to that unsafe block even though the header `extra_data` is not the one produced from the L1-derived attributes.

### Vulnerability Details

#### Jovian extra\_data has three fields

Jovian `extra_data` commits to elasticity, denominator, and `min_base_fee`.

```rust
// crates/common/consensus/src/extra/jovian.rs:26-41
pub fn decode(extra_data: &[u8]) -> Result<(u32, u32, u64), EIP1559ParamError> {
    if extra_data.len() != 17 {
        return Err(EIP1559ParamError::InvalidExtraDataLength);
    }
    if extra_data[0] != VERSION_BYTE {
        return Err(EIP1559ParamError::InvalidVersion(extra_data[0]));
    }
    let denominator: [u8; 4] = extra_data[1..5].try_into().expect("sufficient length");
    let elasticity: [u8; 4] = extra_data[5..9].try_into().expect("sufficient length");
    let min_base_fee: [u8; 8] = extra_data[9..17].try_into().expect("sufficient length");
    Ok((
        u32::from_be_bytes(elasticity),
        u32::from_be_bytes(denominator),
        u64::from_be_bytes(min_base_fee),
    ))
}
```

Derived payload attributes carry `min_base_fee` after Jovian activation.

```rust
// crates/consensus/derive/src/attributes/stateful.rs:217-221
min_base_fee: self
    .rollup_cfg
    .is_jovian_active(next_l2_time)
    .then(|| sys_config.min_base_fee.unwrap_or_default()),
```

The payload builder uses that value to encode the block header.

```rust
// crates/execution/payload/src/payload.rs:452-457
let extra_data = if chain_spec.is_jovian_active_at_timestamp(attributes.timestamp()) {
    attributes
        .get_jovian_extra_data(
            chain_spec.base_fee_params_at_timestamp(attributes.timestamp()),
        )
        .map_err(PayloadBuilderError::other)?
```

#### Root cause - consolidation discards min\_base\_fee

`AttributesMatch::check_eip1559()` decodes Jovian `extra_data`, drops the third tuple field, and compares only elasticity and denominator.

```rust
// crates/consensus/engine/src/attributes.rs:213-219
let extra_data_decoded = if config.is_jovian_active(block.header.timestamp) {
    JovianExtraData::decode(&block.header.extra_data).map(|(be, bd, _)| (be, bd))
} else if config.is_holocene_active(block.header.timestamp) {
    HoloceneExtraData::decode(&block.header.extra_data)
} else {
    return AttributesMismatch::MissingBlockEIP1559.into();
};
```

The only comparison that follows is:

```rust
// crates/consensus/engine/src/attributes.rs:245-253
if ae != be || ad != bd {
    return AttributesMismatch::EIP1559Parameters(
        BaseFeeParams { max_change_denominator: ad, elasticity_multiplier: ae },
        BaseFeeParams { max_change_denominator: bd, elasticity_multiplier: be },
    )
    .into();
}

Self::Match
```

There is no check that `attributes.min_base_fee == decoded_header_min_base_fee`.

#### Safe-head promotion path

`ConsolidateTask` uses `AttributesMatch::check(...).is_match()` as the gate.

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

When it returns `Match`, the task applies the fetched unsafe block as the new safe head.

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

#### The wrong header can affect later derivation

Base reconstructs Jovian system configuration from accepted block headers.

```rust
// crates/consensus/protocol/src/utils.rs:64-70
if rollup_config.is_jovian_active(block.header.timestamp) {
    let (elasticity, denominator, min_base_fee) =
        JovianExtraData::decode(&block.header.extra_data)?;
    cfg.eip1559_denominator = Some(denominator);
    cfg.eip1559_elasticity = Some(elasticity);
    cfg.min_base_fee = Some(min_base_fee);
}
```

The execution chain spec also uses the parent header's Jovian `min_base_fee` to clamp the next block's base fee.

```rust
// crates/execution/chainspec/src/basefee.rs:51-72
let (elasticity, denominator, min_base_fee) = JovianExtraData::decode(parent.extra_data())?;
...
if next_base_fee < min_base_fee {
    return Ok(min_base_fee);
}
```

## Impact Details

**Severity: High - Unintended chain split (network partition).**

A signed unsafe block can differ from the L1-derived Jovian payload attributes only in the header `min_base_fee`. Nodes that previously imported that unsafe block can promote it to safe through consolidation, while nodes deriving or rebuilding from the L1 attributes build the same block with the canonical `min_base_fee` and therefore obtain a different header hash.

In a mixed network state, this can create divergent safe-chain hashes from the same L1 data: the consolidation path accepts `H(B)`, while the derivation/build path produces `H(A)`.

This is not harmless metadata drift. Jovian `min_base_fee` is later read from accepted headers for system-config reconstruction and next-block base-fee computation, so the bad safe header can affect follow-on derivation.

The direct attacker precondition is control of the expected unsafe block signer, or an equivalent faulty signer path. The affected input is still consensus-critical because unsafe blocks are supposed to be independently checked against canonical L1-derived attributes before safe-head promotion.

## Link to Proof of Concept

<https://gist.github.com/a-qedaudit/fa9b320184d1e65de35506ff4d2dd685>

## Proof of Concept

A single localnet PoC is included.

**Runnable artifacts (secret gist):** <https://gist.github.com/a-qedaudit/fa9b320184d1e65de35506ff4d2dd685>

Contains only the localnet reproduction kit and instructions: `README.md`, `run_localnet_poc.sh`, `check_localnet_poc.sh`, and `instrumentation.patch`. Clone the gist and follow the reproduction steps below.

```sh
git clone https://gist.github.com/a-qedaudit/fa9b320184d1e65de35506ff4d2dd685 ozjekkr1-poc
cd ozjekkr1-poc
git clone https://github.com/base/base.git base
git -C base checkout e3467a2048881213b56739a54a876efb9c6ea103
BASE_REPO="$PWD/base" ./run_localnet_poc.sh
./check_localnet_poc.sh
```

The localnet PoC applies `instrumentation.patch` to `base/base@v0.8.0-rc.28` and starts the real Base devnet through `DevnetBuilder`. A test-only sequencer hook makes the real sequencer build and sign the first unsafe block with a non-canonical Jovian `min_base_fee = 9`; the live derived attributes for that block have `min_base_fee = 1000000000`. The hook only changes unsafe block production by the sequencer; validator derivation, L1 batch handling, `AttributesMatch::check`, and `ConsolidateTask::execute` are unmodified. The real validator imports the wrong unsafe block through normal unsafe propagation. The PoC then posts an authorized L1 batch for the same block and observes the validator consolidate that wrong unsafe hash as safe. `run_localnet_poc.sh` checks that `BASE_REPO` is exactly `e3467a2048881213b56739a54a876efb9c6ea103`.

Observed localnet evidence:

```
ozjekkr1 full-network E2E: real sequencer unsafe import -> authorized L1 derivation -> safe consolidation
audit target: base/base@v0.8.0-rc.28 / e3467a2048881213b56739a54a876efb9c6ea103
ozjekkr1 sequencer canonical min_base_fee : 1000000000
ozjekkr1 sequencer forced unsafe min_base_fee : 9
configured batcher address         : 0x976EA74026E726554dB657fA54763abd0C3a0aa9
normal in-process batcher signer   : 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
manual authorized batcher signer   : 0x976EA74026E726554dB657fA54763abd0C3a0aa9
batch inbox                        : 0x00bcC8554867297685A924F26b643A0146De823E
validator unsafe source            : normal sequencer unsafe propagation
unsafe target block                : #1 0xbad80cf84f7cbeaad5afdf7eb1f06b2acdb4a26c9c0bb6bdcbfa03a6614158a2
unsafe header min_base_fee         : 9
client safe before authorized L1   : #0 0x76e66b94edafd1238f47bbbd51c31c71877ee0db650c85ce96b1f3a83e82451f
target L1 batch tx count           : 0
authorized L1 calldata length      : 113 bytes
authorized L1 batch tx             : 0xf844e48ade8ef8a5c4c9f8f0e6d72afe151f449cb08e224088b026a979a34bfa
authorized L1 batch block          : #44
client safe after authorized L1    : #1 0xbad80cf84f7cbeaad5afdf7eb1f06b2acdb4a26c9c0bb6bdcbfa03a6614158a2
safe header min_base_fee           : 9
safe head equals unsafe hash       : true
safe min_base_fee equals unsafe    : true
VERDICT: FULL_NETWORK_JOVIAN_MIN_BASE_FEE_CONSOLIDATION_BYPASS
VERDICT: REAL_UNSAFE_BLOCK_IMPORTED_AND_MARKED_SAFE
```

This demonstrates the exact invariant break through the normal localnet path. The unsafe block is not manually supplied to `ConsolidateTask`: it is built and signed by the real sequencer, imported by the real validator through normal unsafe propagation, and later promoted to safe after real authorized L1 batch data is derived. The first sequencer build prints the canonical derived `min_base_fee = 1000000000` and the forced unsafe header value `9`; the live safe head then advances to the wrong unsafe hash with `min_base_fee = 9`.

The target block is intentionally transaction-empty (`target L1 batch tx count : 0`); the mismatch is in the Jovian header field committed by `extra_data`, so the block hash differs even with identical transaction contents.

## Mitigation

Compare Jovian `min_base_fee` in `AttributesMatch::check_eip1559()`.

One direct fix is to keep the full Jovian decode result and add an explicit comparison:

```rust
if config.is_jovian_active(block.header.timestamp) {
    let (be, bd, bm) = JovianExtraData::decode(&block.header.extra_data)?;
    if attributes.attributes().min_base_fee != Some(bm) {
        return AttributesMismatch::MinBaseFee(attributes.attributes().min_base_fee, bm).into();
    }
    ...
}
```

Add a regression test where Jovian attributes and block header have equal elasticity and denominator but different `min_base_fee`; `AttributesMatch::check()` must return a mismatch and `ConsolidateTask` must not advance the safe head via L1 consolidation.
