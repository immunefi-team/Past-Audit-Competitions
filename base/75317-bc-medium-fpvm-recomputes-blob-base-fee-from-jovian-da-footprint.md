# 75317 bc medium fpvm recomputes blob base fee from jovian da footprint

**Submitted on Apr 28th 2026 at 13:39:30 UTC by @iam0x04 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75317
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Vulnerability Details

The stateless fault-proof executor builds its `BlockEnv` by deriving the next block's excess blob gas from the parent header:

`crates/proof/executor/src/builder/env.rs`:

```rust
let (params, fraction) = if spec_id.is_enabled_in(OpSpecId::ISTHMUS) {
    (Some(BlobParams::prague()), BLOB_BASE_FEE_UPDATE_FRACTION_PRAGUE)
} else if spec_id.is_enabled_in(OpSpecId::ECOTONE) {
    (Some(BlobParams::cancun()), BLOB_BASE_FEE_UPDATE_FRACTION_CANCUN)
} else {
    (None, 0)
};

let blob_excess_gas_and_price = parent_header
    .maybe_next_block_excess_blob_gas(params)
    .or_else(|| spec_id.is_enabled_in(OpSpecId::ECOTONE).then_some(0))
    .map(|excess| BlobExcessGasAndPrice::new(excess, fraction));
```

That is no longer valid after Jovian. Base repurposes `header.blob_gas_used` to store the DA footprint, while consensus keeps `excess_blob_gas` fixed at zero. The production execution node reflects this by hard-coding every post-Cancun `BlockEnv` to:

```
excess_blob_gas = 0
blob_gasprice = 1
```

in both block execution and Engine API payload validation:

* `crates/execution/evm/src/lib.rs:81-84`
* `crates/execution/evm/src/lib.rs:112-115`
* `crates/execution/evm/src/lib.rs:292-295`

<https://github.com/base/base/blob/4b389f03b4963ec7bbb4d024445af6281a90d9a8/crates/execution/evm/src/lib.rs>

The proof executor instead feeds `parent.blob_gas_used` into Ethereum's `maybe_next_block_excess_blob_gas` calculation. For a Jovian parent with a high DA footprint, the FPVM executes the next block with a non-zero excess blob gas and a `BLOBBASEFEE` greater than `1`, while real Base nodes execute the same block with `BLOBBASEFEE == 1`.

## Impact Details

This is a proof soundness break. A proposer can construct a valid L2 parent with high Jovian DA footprint, then include a child transaction or contract path that reads the `BLOBBASEFEE` opcode. The production node computes state using `BLOBBASEFEE == 1`, while the fault-proof program computes a different state root/output root from the same L1 data.

An invalid output root matching the buggy FPVM execution can survive challenge because challengers and on-chain fault proofs use the same divergent execution logic. Conversely, valid canonical outputs for affected blocks become unprovable by this FPVM.

## References

<https://github.com/base/base/blob/4b389f03b4963ec7bbb4d024445af6281a90d9a8/crates/proof/executor/src/builder/env.rs#L95-L116>

## Proof of Concept

Put this under `crates/proof/executor/src/builder/env.rs`. and uses the same RollupConfig import as production.

```rust
#[test]
fn jovian_parent_da_footprint_changes_fpvm_blobbasefee() {
    let mut config = RollupConfig { block_time: 2, ..Default::default() };
    config.hardforks.jovian_time = Some(1);

    let parent = Header {
        timestamp: 3,
        number: 10,
        gas_limit: 30_000_000,
        base_fee_per_gas: Some(1),
        // After Jovian this is DA footprint, not Ethereum blob gas.
        blob_gas_used: Some(10_000_000),
        excess_blob_gas: Some(0),
        ..Default::default()
    };

    let mut attrs = BasePayloadAttributes::default();
    attrs.payload_attributes.timestamp = 5;
    attrs.payload_attributes.suggested_fee_recipient = Address::ZERO;
    attrs.payload_attributes.prev_randao = B256::ZERO;
    attrs.gas_limit = Some(parent.gas_limit);

    let executor = StatelessL2Builder::new(
        &config,
        BaseEvmFactory::default(),
        NoopTrieDBProvider,
        NoopTrieHinter,
        parent.clone().seal_slow(),
    );

    let block_env = executor
        .prepare_block_env(OpSpecId::JOVIAN, &parent, &attrs, &BaseFeeParams::optimism(), 0)
        .unwrap();

    let blob_env = block_env.blob_excess_gas_and_price.unwrap();

    // Production Base execution hardcodes post-Cancun blob_gasprice to 1.
    // The FPVM incorrectly derives it from Jovian DA footprint.
    assert_ne!(
        blob_env.blob_gasprice, 1,
        "Base node uses BLOBBASEFEE=1, but FPVM derived it from Jovian DA footprint"
    );
    assert!(
        blob_env.blob_gasprice > 1,
        "vulnerable FPVM derives BLOBBASEFEE from Jovian DA footprint"
    );
}
```

Expected behavior:\
After Jovian, Base execution treats header.blob\_gas\_used as DA footprint and keeps the EVM blob gas price fixed at 1. The FPVM should therefore build the child `BlockEnv` with `BLOBBASEFEE = 1`.

Actual behavior:\
The FPVM passes the Jovian parent header into `maybe_next_block_excess_blob_gas()`. Because `parent.blob_gas_used` now stores DA footprint, a high-DA parent makes the FPVM compute `BLOBBASEFEE > 1`.

Impact:\
Any child transaction that reads `BLOBBASEFEE` can execute differently in the FPVM than on production Base nodes, producing different state/output roots from the same L1 data.

```bash
cargo test -p base-proof-executor jovian_parent_da_footprint_changes_fpvm_blobbasefee --lib --features std -- --nocapture
```
