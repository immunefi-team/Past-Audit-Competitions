# #42557 \[BC-Low] Remote signing methods can fail which will turn off the light node block proposer

**Submitted on Mar 24th 2025 at 16:13:01 UTC by @Franfran for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42557
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/protocol/light-node>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

The movement sequencer ensure DA by posting blobs to Celestia.\
Here, the issue is that the block proposer fully trusts the remote signing availability if enabled, which cannot be guaranteed.\
If it ever fails, the block proposer will crash and the sequencer won't be able to post new blocks to the DA chain.

## Vulnerability Details

HashCorp vaults are not yet supported, because the signing method `load` function which is ran at the initialization of the signer returns an error: <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/util/signing/util/loader/src/lib.rs#L104>, but AWS KMS is a remote signing method that is supported.

```rust
#[async_trait::async_trait]
impl Load<Secp256k1> for SignerIdentifier {
	async fn load(&self) -> Result<LoadedSigner<Secp256k1>, LoaderError> {
		info!("loading a secp256k1 signer {:?}", self);
		match self {
			SignerIdentifier::Local(local) => {
				let signer = movement_signer_local::signer::LocalSigner::from_signing_key_hex(
					&local.private_key_hex_bytes,
				)
				.map_err(|e| LoaderError::InvalidSigner(e.into()))?;
				Ok(LoadedSigner::new(
					Arc::new(signer) as Arc<dyn Signing<Secp256k1> + Send + Sync>,
					self.clone(),
				))
			}
			SignerIdentifier::AwsKms(aws_kms) => {
				let builder =
					movement_signer_aws_kms::hsm::key::Builder::new().create_key(aws_kms.create);
				let key = aws_kms.key.clone();
				let signer =
					builder.build(key).await.map_err(|e| LoaderError::InvalidSigner(e.into()))?;
				Ok(LoadedSigner::new(
					Arc::new(signer) as Arc<dyn Signing<Secp256k1> + Send + Sync>,
					self.clone(),
				))
			}
			SignerIdentifier::HashiCorpVault(_hashi_corp_vault) => Err(LoaderError::InvalidCurve),
		}
	}
}
```

It can be enabled by setting the [`MOVEMENT_CELESTIA_DA_KEY_IDENTIFIER`](https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/util/src/config/da_light_node.rs#L72-L76) variable to [`aws_kms`](https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/util/signing/util/loader/src/identifiers/mod.rs#L50-L52), but the local signing method using a private key stored in memory is by default.

When the block proposer produces blocks in order to post them on Celestia as a DA solution, it must sign the blob.\
<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/light-node/src/sequencer.rs#L168>

```rust
	async fn submit_blocks(&self, blocks: Vec<Block>) -> Result<(), anyhow::Error> {
		for block in blocks {
			let data: InnerSignedBlobV1Data<C> = block.try_into()?;
			let blob = data.try_to_sign(&self.pass_through.signer).await?;
			self.pass_through.da.submit_blob(blob.into()).await?;
		}


		Ok(())
	}
```

The issue is that the [`try_to_sign`](https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/light-node/src/sequencer.rs#L168) function is faillible, especially if using a remote solution such as AWS KMS which might have a downtime at this moment.\
If it returns an error, the error will bubble up until to the [`run_block_publisher`](https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/light-node/src/sequencer.rs#L270) function which will crash the block proposer and then will crash the movement-celestia-da-light-node binary.\
The sequencer will be unable to propose new blocks which will freeze the chain.

## Impact Details

Shut down of the sequencer block proposer, which is critical for the operation of the Movement network.

## References

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/light-node/src/sequencer.rs#L168>

## Proof of Concept

## Proof of Concept

1. Write an artificial error in order to simulate an error returned from `try_to_sign`
2. Run the movement network with the Celestia da light client
3. Observe the light client going off, and see that it cannot propose any blocks anymore. Transactions sent to the full node won't make it crash, but no blocks are produced anymore on the DA chain.

```diff
diff --git a/protocol-units/da/movement/protocol/light-node/src/sequencer.rs b/protocol-units/da/movement/protocol/light-node/src/sequencer.rs
index 6cecd37..099dfeb 100644
--- a/protocol-units/da/movement/protocol/light-node/src/sequencer.rs
+++ b/protocol-units/da/movement/protocol/light-node/src/sequencer.rs
@@ -167,6 +167,7 @@ where
 			let data: InnerSignedBlobV1Data<C> = block.try_into()?;
 			let blob = data.try_to_sign(&self.pass_through.signer).await?;
 			self.pass_through.da.submit_blob(blob.into()).await?;
+			return Err(anyhow::anyhow!("artificial error"));
 		}
 
 		Ok(())
```

![Image](https://github.com/user-attachments/assets/813a2b65-468a-4891-a339-2e54b311eca2)
