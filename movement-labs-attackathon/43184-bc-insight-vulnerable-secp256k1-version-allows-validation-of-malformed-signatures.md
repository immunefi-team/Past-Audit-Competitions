# #43184 \[BC-Insight] Vulnerable \`Secp256k1\` version allows validation of malformed signatures

**Submitted on Apr 3rd 2025 at 12:50:35 UTC by @Rhaydden for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43184
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/util/signing/providers/aws-kms>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Brief/Intro

The Movement protocol is using outdated versions of the `secp256k1` library (0.24 and 0.27), which are popularly vulnerable to accepting signatures with overflow in R or S parameters. This vulnerability (CVE-2021-38195) could allow attackers to craft malformed signatures that would be validated as legitimate by the protocol's AWS KMS signing provider, potentially enabling unauthorized transaction validation.

See here for more details: <https://rustsec.org/advisories/RUSTSEC-2021-0076.html>

## Vulnerability Details

AWS KMS provider uses the secp256k1 library version 0.24 which is substantially below the fixed version 0.5.0. This allows signatures with R or S parameters larger than the secp256k1 curve order to be accepted as valid.

In the `cargo.toml` file, the dependency is specified as:

```rust
secp256k1 = "0.24"
```

The issue is in the path is in the signing implementation in `mod.rs` where DER signatures are parsed and converted to secp256k1 signatures:

<https://github.com/immunefi-team/attackathon-movement//blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/util/signing/providers/aws-kms/src/hsm/mod.rs#L110-L115>

```rust
	let secp_signature =
			Secp256k1Signature::from_der(der_signature.as_ref()).map_err(|e: Secp256k1Error| {
				SignerError::Internal(format!("Failed to parse DER signature: {}", e))
			})?;

		let raw_signature = secp_signature.serialize_compact();
```

This code directly uses the vulnerable `from_der` function from the secp256k1 library which doesn't properly check for overflow in R and S parameters. Additionally, the main project also uses a vulnerable version (0.27) as seen in the root Cargo.toml:

```
secp256k1 = { version = "0.27", default-features = false, features = [
```

The bug affects any protoccol using these versions of the library for signature validation, as it may accept invalid signatures that other protocols with proper validation would reject.

## Impact Details

I don't see how this will lead to loss of funds. But it impacts the cryptography of the protocol. But this falls under "A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk" category.

An attacker could craft signatures with malformed R or S values that would be accepted by the protocol but rejected by other chain/ protocols.

## References

* <https://github.com/immunefi-team/attackathon-movement//blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/util/signing/providers/aws-kms/src/hsm/mod.rs#L110-L115>
* RUSTSEC-2021-0076: <https://rustsec.org/advisories/RUSTSEC-2021-0076>
* CVE-2021-38195: <https://nvd.nist.gov/vuln/detail/CVE-2021-38195>
* GitHub Fix PR: <https://github.com/paritytech/libsecp256k1/pull/67>

## Proof of Concept

## Proof of Concept

1. Generate a secp256k1 key pair (attacker\_private\_key, attacker\_public\_key)
2. Create a malformed signature with R or S parameters larger than the secp256k1 curve order n (where n = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141)
3. Craft a signature where R or S is slightly larger than the order n, for example:

```rust
// Pseudocode for demonstration
let curve_order = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141;
let malformed_r = curve_order + 1;
let normal_s = // valid S value
let malformed_signature = create_der_signature(malformed_r, normal_s);
```

4. When this signature is submitted to the protocol's AWS KMS provider for validation:

```rust
// Example attack path
// Assuming message is the hash of a transaction
let message = hash_transaction(tx);

// The vulnerable code in the AWS KMS provider will accept this malformed signature
// from_der doesn't check for overflow
let secp_signature = Secp256k1Signature::from_der(malformed_signature);

// The signature will be accepted despite having an R value larger than the curve order
```

5. The AWS KMS provider would accept this malformed signature as valid.

## Fix

Update secp256k1 library to the version >=0.5.0
