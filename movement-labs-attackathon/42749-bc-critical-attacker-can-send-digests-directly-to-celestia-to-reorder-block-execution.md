# #42749 \[BC-Critical] Attacker can send digests directly to Celestia to reorder block execution

**Submitted on Mar 25th 2025 at 17:55:18 UTC by @HollaDieWaldfee for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42749
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

## Brief/Intro

When digest blobs are submitted to Celestia, an attacker can re-send them in a different order and with a higher transaction fee than the DA light node so that they are reordered and streamed to Movement in a different order. As a result, blocks will be executed in a different order than the one that Movement sequenced them in.

## Vulnerability Details

Movement sequences blocks in a specific order and submits the digests to Celestia. On execution, Movement streams the blobs from Celestia and executes the blocks in the streamed order. However, an attacker can re-send digests to Celestia in a different order and with a higher transaction fee so that they are streamed to Movement in a different order and, as a result, executed in a different order. By submitting blobs with higher transaction fees they are included before the blobs sent by the DA light node.

The blob digests submitted to Celestia by the DA light node are then skipped because the execution logic contains checks that blocks can only be executed once.

## Impact Details

Blocks are not executed in the specific order that Movement sequenced them in. Maintaining the sequenced order is important because transactions are ordered according to their application priority that is assigned by the full node mempool. If transactions are not executed in this order, for example transactions with lower gas prices can be moved in front of transactions with higher gas prices.

This issue breaks the fundamental mechanism of the mempool which is to prioritize transactions with higher fees. This has a lot of downstream consequences, for example:

* Inefficient allocation of block space
* MEV is not possible, leading to further inefficiencies
* Malicious actors can reorder blocks for a financial advantage

Overall, the impact is high, corresponding to the following impact from the in-scope impacts "Causing network processing nodes to process transactions from the mempool beyond set parameters" since transactions are not processed to their application priorities that they get assigned by the mempool.

## References

(1): <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/sequencing/memseq/sequencer/src/lib.rs#L58-L67>

(2): <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/providers/digest-store/src/da/mod.rs#L63-L85>

(3): <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/networks/movement/movement-full-node/src/node/tasks/execute\\_settle.rs#L73-L80>

## Proof of Concept

## Proof of Concept

1. Multiple blocks are built and sequenced in a specific order since every block has a parent block. This can be verified by taking a look at reference (1).
2. The digest blobs are submitted to Celestia while the actual data is stored in the local database (reference (2)).
3. An attacker submits the digest blobs in a different order and with a higher transaction fee than the DA light node, hence they will be streamed in a different order.
4. When `execute_settle__run()` is called, the blobs are streamed from Celestia (reference (3)). The blocks will then be executed in a different order.
