# #43312 \[BC-Medium] get\_state\_proof() is called with the current version leading to the epoch\_changes of the StateProof always being empty

**Submitted on Apr 4th 2025 at 14:00:01 UTC by @HollaDieWaldfee for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43312
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Brief/Intro

When executing a block, `get_state_proof()` is called but with the wrong version. As a result, the epoch\_changes of the StateProof is always going to be empty, producing an invalid block commitment.

## Vulnerability Details

After a block is executed, `get_state_proof()` is called with the version being the `state_compute.version()` after the new block is executed. This version is the current version and as a result, when `get_state_proof_with_ledger_info()` is called, it always enters the else-block which returns a StateProof with empty epoch\_changes:

```rust
    fn get_state_proof_with_ledger_info(
        &self,
        known_version: u64,
        ledger_info_with_sigs: LedgerInfoWithSignatures,
    ) -> Result<StateProof> {
        gauged_api("get_state_proof_with_ledger_info", || {
            let ledger_info = ledger_info_with_sigs.ledger_info();
            ensure!(
                known_version <= ledger_info.version(),
                "Client known_version {} larger than ledger version {}.",
                known_version,
                ledger_info.version(),
            );
            let known_epoch = self.ledger_db.metadata_db().get_epoch(known_version)?;
            let end_epoch = ledger_info.next_block_epoch();
            let epoch_change_proof = if known_epoch < end_epoch {
                let (ledger_infos_with_sigs, more) =
                    self.get_epoch_ending_ledger_infos(known_epoch, end_epoch)?;
                EpochChangeProof::new(ledger_infos_with_sigs, more)
@>            } else {
                EpochChangeProof::new(vec![], /* more = */ false)
            };

            Ok(StateProof::new(ledger_info_with_sigs, epoch_change_proof))
        })
    }
```

Instead, `get_state_proof()` should be called with the version BEFORE the block is executed such that a change in the epoch can be detected in the if-case.

## Impact Details

The epoch\_changes of the state proof after executing a block is always going to be empty.\
As a result, the block commitment that is calculated from the state proof can be invalid:

```rust
let commitment = Commitment::digest_state_proof(&proof);
Ok(BlockCommitment::new(block_height.into(), Id::new(*block_id.clone()), commitment))
```

## References

(1): <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/executor/execution.rs#L63-L89>

(2): <https://github.com/immunefi-team/attackathon-movement-aptos-core/blob/627b4f9e0b63c33746fa5dae6cd672cbee3d8631/storage/aptosdb/src/db/include/aptosdb\\_reader.rs#L399-L424>

## Proof of Concept

## Proof of Concept

1. During the block execution flow, `execute_block()` is called. After the block is executed, `get_state_proof()` is called. However, this function is called with the wrong version, namely the version after the block is executed (reference (1)).
2. As a result, `get_state_proof_with_ledger_info()` always returns a StateProof with empty epoch\_changes since `known_epoch == end_epoch` and the function always enters the else-block.
