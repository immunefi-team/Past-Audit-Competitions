# #37568 \[BC-Insight] missing specification logic

## #37568 \[BC-Insight] Missing Specification Logic

**Submitted on Dec 9th 2024 at 10:53:20 UTC by @Pig46940 for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37568
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/chainsafe/lodestar>
* **Impacts:**
  * (Specifications) A bug in specifications with no direct impact on client implementations

### Description

### Brief/Intro

Lacking the logic which the length of KZG commitments is less than or equal to the limitation defined in Consensus Layer

### Vulnerability Details

Lacking the logic of the following.

The type of the payload of this topic changes to the (modified) SignedBeaconBlock found in Deneb.

\[Modified in Deneb:EIP4844]

New validation:

\[REJECT] The length of KZG commitments is less than or equal to the limitation defined in Consensus Layer -- i.e. validate that len(body.signed\_beacon\_block.message.blob\_kzg\_commitments) <= MAX\_BLOBS\_PER\_BLOCK

### Impact Details

A bug in specifications with no direct impact on client implementations

### References

## Specs

<https://github.com/ethereum/consensus-specs/blob/dev/specs/deneb/p2p-interface.md#beacon\\_block>

## Codes

* <https://github.com/ChainSafe/lodestar/blob/dad9037e7739d5bcbccfe627e715ef40e9ba935b/packages/beacon-node/src/chain/blocks/verifyBlock.ts>
* <https://github.com/ChainSafe/lodestar/blob/dad9037e7739d5bcbccfe627e715ef40e9ba935b/packages/beacon-node/src/chain/blocks/verifyBlocksDataAvailability.ts>

### Proof of Concept

### Proof of Concept

To adhere to the specification, you would need an explicit check like:

if (numBlobs > MAX\_BLOBS\_PER\_BLOCK) {\
throw new BlockError(block0, {\
code: BlockErrorCode.INVALID\_BLOB\_COUNT,\
error: new Error(`Block contains ${numBlobs} blobs, exceeding MAX_BLOBS_PER_BLOCK=${MAX_BLOBS_PER_BLOCK}`),\
});\
}

While the code references and interacts with blobKzgCommitments, it does not implement the specification's requirement to validate the blob count against MAX\_BLOBS\_PER\_BLOCK. Adding the explicit validation step is necessary to ensure compliance.
