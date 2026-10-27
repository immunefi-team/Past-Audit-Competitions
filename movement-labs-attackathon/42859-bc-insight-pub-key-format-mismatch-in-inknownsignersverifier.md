# #42859 \[BC-Insight] Pub key format mismatch in \`InKnownSignersVerifier\`

**Submitted on Mar 27th 2025 at 18:41:05 UTC by @Rhaydden for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42859
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/protocol/verifier>
* **Impacts:**
  * Unintended chain split (network partition)

## Description

## Brief/Intro

In the `InKnownSignersVerifier` component, there's a mismatch between public key encoding formats (compressed vs. uncompressed SEC1) thatt can cause valid transactions to be rejected. This could lead to network partitioning where different nodes disagree on transaction validity, causing an unintended chain split.

## Vulnerability Details

In the verification logic in `InKnownSignersVerifier` class, which is responsible for verifying that a transaction's signer is in the set of known authorized signers, we are making a direct string comparison between differently formatted representations of the same pub key.

In `InKnownSignersVerifier::verify`, we directly compare hex strings without normalizing the format as seen here:

<https://github.com/immunefi-team/attackathon-movement//blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/verifier/src/signed/mod.rs#L72-L87>

```rust
	async fn verify(&self, blob: DaBlob<C>, height: u64) -> Result<Verified<DaBlob<C>>, Error> {
		let da_blob = self.inner_verifier.verify(blob, height).await?;
		let signer = da_blob.inner().signer_hex();
		if !self.known_signers_sec1_bytes_hex.contains(&signer) {
			return Err(Error::Validation(
				format!(
					"signer {} is not in the known signers set {:#?}",
					signer, self.known_signers_sec1_bytes_hex
				)
				.to_string(),
			));
		}

		Ok(da_blob)
	}
}
```

Now, let's take a look at the`signer_hex()` method in `DaBlob` simply hex-encodes the raw bytes without ensuring a consistent format:

<https://github.com/immunefi-team/attackathon-movement//blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/util/src/blob/ir/blob.rs#L99-L101>

```rust
	pub fn signer_hex(&self) -> String {
		hex::encode(self.signer())
	}
```

The `PublicKey` struct in Secp256k1 is defined but with a misleading comment:

<https://github.com/immunefi-team/attackathon-movement//blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/util/signing/interface/src/cryptography/secp256k1.rs#L14>

```rust
fixed_size!(pub struct PublicKey([u8; 65])); // Compressed public key
```

This comment is incorrect as 65 bytes corresponds to an uncompressed SEC1 format (uncompressed keys are 65 bytes, compressed are 33 bytes).

Now the issue here's that the same public key can have two different valid representations as shown below:

1. Uncompressed format (65 bytes): `0x04 + X-coordinate + Y-coordinate`
2. Compressed format (33 bytes): `0x02/0x03 + X-coordinate`

When the `known_signers_sec1_bytes_hex` set contains keys in one format, but `signer_hex()` returns keys in another format, valid signatures will be rejected even though they represent the same cryptographic key.

## Impact Details

Falls under "Unintended chain split (network partition)" as it causes different nodes to disagree on the validity of txns without requiring a hard fork to fix.

The network could split into multiple incompatible chains, with nodes that use compressed public key formats diverging from those using uncompressed formats.\
Different nodes may accept or reject the same valid txns based on how their known signers are configured.

## References

<https://github.com/immunefi-team/attackathon-movement//blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/verifier/src/signed/mod.rs#L72-L87>

## Proof of Concept

## Proof of Concept

We could have a secp256k1 public key with:

* X coordinate: 0x79BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798
* Y coordinate: 0x483ADA7726A3C4655DA4FBFC0E1108A8FD17B448A68554199C47D08FFB10D4B8

In the uncompressed format (65 bytes), it's going to be:

```
0x04 + X + Y = 0x0479BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798483ADA7726A3C4655DA4FBFC0E1108A8FD17B448A68554199C47D08FFB10D4B8
```

In compressed format (33 bytes) assuming Y is even:

```
0x02 + X = 0x0279BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798
```

Now let's consider this steps:

#### Setup

The `known_signers_sec1_bytes_hex` set contains the compressed format:

```
known_signers_sec1_bytes_hex = {"0279BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798"}
```

#### Incoming blob

A blob is received with a valid signature from the same key, but stored in uncompressed format:

```
blob.signer = [0x04, 0x79, 0xBE, ...] // 65 bytes total
```

#### Verification process

* `signer_hex()` returns: `"0479BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798483ADA7726A3C4655DA4FBFC0E1108A8FD17B448A68554199C47D08FFB10D4B8"`
* The verifier checks: `known_signers_sec1_bytes_hex.contains(&signer)`
* Result: `false` because the hex strings don't match, even though they represent the same public key

#### Outcome

The verification fails with an error like:

```
Error::Validation("signer 0479BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798483ADA7726A3C4655DA4FBFC0E1108A8FD17B448A68554199C47D08FFB10D4B8 is not in the known signers set {"0279BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798"}")
```

## Fix

We need to ensure a consistent pub key format handling by normalizing the format before comparison. From the top of my mind, there are 2 ways i think we can achieve this:

1. by modify `DaBlob::signer_hex()` to always return a consistent format. something like this:

```rust
pub fn signer_hex(&self) -> String {
    let signer_bytes = self.signer();
    
    // Convert to a consistent format (e.g., always compressed)
    if signer_bytes.len() == 65 || signer_bytes.len() == 33 {
        let verifying_key = ecdsa::VerifyingKey::from_sec1_bytes(signer_bytes)
            .expect("Valid SEC1 public key");
        return hex::encode(verifying_key.to_encoded_point(true).as_bytes()); // true = compressed
    }
    
    // Fall back to raw encoding if format is unknown
    hex::encode(signer_bytes)
}
```

or

2. By normalizing both formats in the verification logic code.
