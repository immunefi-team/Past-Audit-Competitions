# #42112 \[BC-Critical] Using \`blob.GetAll\` instead of \`blob.Get\` for Celestia DA opens full nodes to fraudulent block attacks

**Submitted on Mar 20th 2025 at 21:10:55 UTC by @usmannk for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42112
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/providers/celestia>
* **Impacts:**
  * Unintended permanent chain split requiring hard fork (network partition requiring hard fork)
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

The Movement full node relies on the Celestia blob.GetAll function to retrieve DA blobs from a given height range. Because Celestia namespaces are not permissioned zones, an attacker can insert their own blocks into this range. They can then add blocks that were not sequenced by the sequencer or consume excess amounts of resources.

## Vulnerability Details

Movement does not verify the veracity of blocks posted to Celestia. When executing a block from DA, a block metadata transaction is constructed for it: <https://github.com/immunefi-team/attackathon-movement/blob/main/networks/movement/movement-full-node/src/node/tasks/execute\\_settle.rs#L233>.

Then, the block is executed and the aptos executor adds this block as the parent block id for the future: <https://github.com/immunefi-team/attackathon-movement/blob/main/protocol-units/execution/maptos/opt-executor/src/executor/execution.rs#L53>

## Impact Details

If an attacker adds many blocks to a single Celestia height or multiple heights, non-sequencer nodes will attempt to process these and could be forced to consume excess resources, falling behind the chain.

However, the primary impact is that the full nodes that consume these blocks will attempt to follow their chain. The sequencer node will go on without these blocks, causing a chain split between the sequencer and all other nodes.

To remedy this, only use the Get function from Celestia, providing commitment hashes that are drawn from the trusted sequencer.

## Proof of Concept

## Proof of Concept

The PoC for this is quite simple. An attacker need only add a blob to Celestia in the Movement namespace containing a valid empty block. The node will add in a metadata tx automatically when executing.

After this, full nodes following the Celestia DA will fork on their next `Get` call, straying from the sequencer state. They will not be able to recover as their chains have permanently split.
