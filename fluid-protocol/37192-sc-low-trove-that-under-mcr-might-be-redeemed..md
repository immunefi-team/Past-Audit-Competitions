# #37192 \[SC-Low] Trove that under MCR might be redeemed.

**Submitted on Nov 28th 2024 at 08:32:15 UTC by @jasonxiale for** [**IOP | Fluid Protocol**](https://immunefi.com/audit-competition/iop-fluid-protocol)

* **Report ID:** #37192
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Hydrogen-Labs/fluid-protocol/tree/main/contracts/protocol-manager-contract/src/main.sw>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

`protocol-manager-contract.redeem_collateral` is supposed to work on troves with ICR >= MCR, but in current implementation, troves(whose ICR < MCR) might be redeemed.

## Vulnerability Details

Quting from [TroveManager.sol#L966-L969](https://github.com/liquity/dev/blob/e38edf3dd67e5ca7e38b83bcf32d515f896a7d2f/packages/contracts/contracts/TroveManager.sol#L966-L969) and the comment [Loop through the Troves starting from the one with lowest collateral ratio until \_amount of LUSD is exchanged for collateral](https://github.com/liquity/dev/blob/e38edf3dd67e5ca7e38b83bcf32d515f896a7d2f/packages/contracts/contracts/TroveManager.sol#L972). We can see that in `liquity's implementation`, the redeem starts first trove with ICR >= MCR, which is also the lowest collateral ratio. This applies to the our project's code in \[<https://github.com/Hydrogen-Labs/fluid-protocol/blob/78ab7bdd243b414b424fca6e1eb144218f36a18a/contracts/protocol-manager-contract/src/main.sw#L326-L329>]:

```rust
294 // Get information about all assets in the system
295 #[storage(read)]
296 fn get_all_assets_info() -> AssetInfo {
    ...
313     while (i < length) {
314         let oracle = abi(Oracle, asset_contracts.get(i).unwrap().oracle.into());
315         let trove_manager = abi(TroveManager, asset_contracts.get(i).unwrap().trove_manager.into());
316         let asset = assets.get(i).unwrap();
317         let price = oracle.get_price();
318         let mut current_borrower = sorted_troves.get_last(asset);
319         let mut current_cr = u64::max();
320         if (current_borrower != null_identity_address()) {
321             current_cr = trove_manager.get_current_icr(current_borrower, price);
322         }
323         prices.push(price);
324         system_debt.push(trove_manager.get_entire_system_debt());
325         redemption_totals.push(RedemptionTotals::default());
--->>> this code is used to find the first trove with ICR >= MCR
326         while (current_borrower != null_identity_address() && current_cr < MCR) {
327             current_borrower = sorted_troves.get_prev(current_borrower, asset);
328             current_cr = trove_manager.get_current_icr(current_borrower, price);
329         }
330         current_borrowers.push(current_borrower);
331         current_crs.push(current_cr);
332         i += 1;
333     }
    ...
343 }
```

However, in above code, **there is a case that after the `while loop` in** [**protocol-manager-contractw#L326-L329**](https://github.com/Hydrogen-Labs/fluid-protocol/blob/78ab7bdd243b414b424fca6e1eb144218f36a18a/contracts/protocol-manager-contract/src/main.sw#L326-L329)**, the last trove's icr might be less than MCR**,in such case, the trove(icr \<MCR) still will be added to `current_borrowers`, and later might be redeemed.

## Impact Details

Please consider the following case:

1. a new asset(TokenA) is added to the protocol by calling `protocol-manager-contract.register_asset`
2. A few users start to borrow USDF using TokenA as collateral with icr a little higher than MCR
3. TokenA's price has fluctuated, all the troves become liquidatable(icr < MCR).
4. in such case, when `redeem_collateral` is called, the liquidatable troves will be redeemed.

## References

<https://github.com/liquity/dev/blob/e38edf3dd67e5ca7e38b83bcf32d515f896a7d2f/packages/contracts/contracts/TroveManager.sol#L966-L969> <https://github.com/liquity/dev/blob/e38edf3dd67e5ca7e38b83bcf32d515f896a7d2f/packages/contracts/contracts/TroveManager.sol#L972>

## Proof of Concept

## Proof of Concept

Please use the following patch and run

```bash
cargo test proper_redemption_with_a_trove_closed_fully_under_MCR -- --nocapture
...
     Running tests/success_redemptions.rs (/opt/work/fluid-protocol/target/debug/deps/success_redemptions-a2d6c1cfcebe8b03)

running 1 test
Deploying core contracts...
Initializing core contracts...
icr : 1368159203
debt: 2010000000000
coll: 2750000000000
icr : 1231343283
debt: 2010000000000
coll: 2750000000000
icr : 1559405940
debt: 1010000000000
coll: 1750000000000
cnt: 1
test success_redemptions::proper_redemption_with_a_trove_closed_fully_under_MCR ... ok
```

As the above output shows, when the user open a trove, the collateral's price is 1e9, and the icr is 1368159203. And then the price is changed to 0.9e9, the icr is 1231343283, but the trove can still be redeem.

1. we define a new price function to support 0.9e9 as price

```diff
diff --git a/test-utils/src/interfaces/pyth_oracle.rs b/test-utils/src/interfaces/pyth_oracle.rs
index 5f6eb70..bd1625a 100644
--- a/test-utils/src/interfaces/pyth_oracle.rs
+++ b/test-utils/src/interfaces/pyth_oracle.rs
@@ -24,6 +24,18 @@ pub fn pyth_price_feed(price: u64) -> Vec<(Bits256, Price)> {
     )]
 }
 
+pub fn pyth_price_feed_d(price: u64) -> Vec<(Bits256, Price)> {
+    vec![(
+        Bits256::zeroed(),
+        Price {
+            confidence: 0,
+            exponent: 9,
+            price: price,
+            publish_time: PYTH_TIMESTAMP,
+        },
+    )]
+}
+
```

2. we add a new function to show icr and owners count

```diff
diff --git a/test-utils/src/interfaces/trove_manager.rs b/test-utils/src/interfaces/trove_manager.rs
index 728359d..397bb05 100644
--- a/test-utils/src/interfaces/trove_manager.rs
+++ b/test-utils/src/interfaces/trove_manager.rs
@@ -292,6 +292,25 @@ pub mod trove_manager_abi {
             .unwrap()
     }

+    pub async fn get_trove_owners_count<T: Account>(
+        trove_manager: &ContractInstance<TroveManagerContract<T>>,
+    ) -> CallResponse<u64> {
+        let tx_params = TxPolicies::default().with_tip(1);
+
+        trove_manager
+            .contract
+            .methods()
+            .get_trove_owners_count()
+            .with_contract_ids(&[
+                trove_manager.contract.contract_id().into(),
+                trove_manager.implementation_id.into(),
+            ])
+            .with_tx_policies(tx_params)
+            .call()
+            .await
+            .unwrap()
+    }
+
     pub async fn get_entire_debt_and_coll<T: Account>(
         trove_manager: &ContractInstance<TroveManagerContract<T>>,
         id: Identity,
@@ -312,6 +331,23 @@ pub mod trove_manager_abi {
             .unwrap()
     }

+    pub async fn get_current_icr<T: Account>(
+        trove_manager: &ContractInstance<TroveManagerContract<T>>,
+        id: Identity, price: u64
+    ) -> CallResponse<u64> {
+        trove_manager
+            .contract
+            .methods()
+            .get_current_icr(id, price)
+            .with_contract_ids(&[
+                trove_manager.contract.contract_id().into(),
+                trove_manager.implementation_id.into(),
+            ])
+            .call()
+            .await
+            .unwrap()
+    }
+
```

3. the following code is used to test

```diff
diff --git a/contracts/protocol-manager-contract/tests/success_redemptions.rs b/contracts/protocol-manager-contract/tests/success_redemptions.rs
index 027834b..12a3a73 100644
--- a/contracts/protocol-manager-contract/tests/success_redemptions.rs
+++ b/contracts/protocol-manager-contract/tests/success_redemptions.rs
@@ -10,7 +10,7 @@ use test_utils::{
         borrow_operations::{borrow_operations_abi, BorrowOperations},
         coll_surplus_pool::coll_surplus_pool_abi,
         protocol_manager::protocol_manager_abi,
-        pyth_oracle::{pyth_oracle_abi, pyth_price_feed},
+        pyth_oracle::{pyth_oracle_abi, pyth_price_feed, pyth_price_feed_d},
         token::token_abi,
         trove_manager::{trove_manager_abi, trove_manager_utils, Status},
     },
@@ -518,3 +518,155 @@ async fn proper_redemption_with_a_trove_closed_fully() {

     assert_eq!(coll_surplus, coll3 - with_min_borrow_fee(debt3));
 }
+
+#[tokio::test]
+async fn proper_redemption_with_a_trove_closed_fully_under_MCR() {
+    let (mut contracts, admin, mut wallets) = setup_protocol(5, true, false).await;
+
+    let healthy_wallet1 = wallets.pop().unwrap();
+
+    let balance: u64 = 12_000 * PRECISION;
+
+    token_abi::mint_to_id(
+        &contracts.asset_contracts[0].asset,
+        balance,
+        Identity::Address(healthy_wallet1.address().into()),
+    )
+    .await;
+
+    let borrow_operations_healthy_wallet1 = ContractInstance::new(
+        BorrowOperations::new(
+            contracts.borrow_operations.contract.contract_id().clone(),
+            healthy_wallet1.clone(),
+        ),
+        contracts.borrow_operations.implementation_id.clone(),
+    );
+
+    let coll1 = 2_750 * PRECISION;
+    let debt1 = 2_000 * PRECISION;
+
+    oracle_abi::set_debug_timestamp(&contracts.asset_contracts[0].oracle, PYTH_TIMESTAMP).await;
+    pyth_oracle_abi::update_price_feeds(
+        &contracts.asset_contracts[0].mock_pyth_oracle,
+        pyth_price_feed_d(1_000_000_000),
+    )
+    .await;
+
+    borrow_operations_abi::open_trove(
+        &borrow_operations_healthy_wallet1,
+        &contracts.asset_contracts[0].oracle,
+        &contracts.asset_contracts[0].mock_pyth_oracle,
+        &contracts.asset_contracts[0].mock_redstone_oracle,
+        &contracts.asset_contracts[0].asset,
+        &contracts.usdf,
+        &contracts.fpt_staking,
+        &contracts.sorted_troves,
+        &contracts.asset_contracts[0].trove_manager,
+        &contracts.active_pool,
+        coll1,
+        debt1,
+        Identity::Address(Address::zeroed()),
+        Identity::Address(Address::zeroed()),
+    )
+    .await
+    .unwrap();
+
+
+    let redemption_amount: u64 = 1_000 * PRECISION;
+
+    let protocol_manager_health1 = ContractInstance::new(
+        ProtocolManager::new(
+            contracts.protocol_manager.contract.contract_id().clone(),
+            healthy_wallet1.clone(),
+        ),
+        contracts.protocol_manager.implementation_id,
+    );
+
+    let icr  = trove_manager_abi::get_current_icr(
+        &contracts.asset_contracts[0].trove_manager,
+        Identity::Address(healthy_wallet1.address().into()),
+        1_000_000_000, // 1.0 * 1e9
+    ).await.value;
+    println!("icr : {}", icr);
+
+    let debt = trove_manager_abi::get_trove_debt(
+        &contracts.asset_contracts[0].trove_manager,
+        Identity::Address(healthy_wallet1.address().into()),
+    ).await.value;
+    println!("debt: {}", debt);
+
+    let coll = trove_manager_abi::get_trove_coll(
+        &contracts.asset_contracts[0].trove_manager,
+        Identity::Address(healthy_wallet1.address().into()),
+    ).await.value;
+    println!("coll: {}", coll);
+
+    oracle_abi::set_debug_timestamp(&contracts.asset_contracts[1].oracle, PYTH_TIMESTAMP).await;
+    pyth_oracle_abi::update_price_feeds(
+        &contracts.asset_contracts[1].mock_pyth_oracle,
+        pyth_price_feed_d(900_000_000),
+    )
+    .await;
+
+    let icr  = trove_manager_abi::get_current_icr(
+        &contracts.asset_contracts[0].trove_manager,
+        Identity::Address(healthy_wallet1.address().into()),
+        900_000_000, // 1.1 * 1e9
+    ).await.value;
+    println!("icr : {}", icr);
+
+    let debt = trove_manager_abi::get_trove_debt(
+        &contracts.asset_contracts[0].trove_manager,
+        Identity::Address(healthy_wallet1.address().into()),
+    ).await.value;
+    println!("debt: {}", debt);
+
+    let coll = trove_manager_abi::get_trove_coll(
+        &contracts.asset_contracts[0].trove_manager,
+        Identity::Address(healthy_wallet1.address().into()),
+    ).await.value;
+    println!("coll: {}", coll);
+
+
+    protocol_manager_abi::redeem_collateral(
+        &protocol_manager_health1,
+        redemption_amount,
+        10,
+        0,
+        None,
+        None,
+        &contracts.usdf,
+        &contracts.fpt_staking,
+        &contracts.coll_surplus_pool,
+        &contracts.default_pool,
+        &contracts.active_pool,
+        &contracts.sorted_troves,
+        &contracts.asset_contracts,
+    )
+    .await;
+
+
+    let icr  = trove_manager_abi::get_current_icr(
+        &contracts.asset_contracts[0].trove_manager,
+        Identity::Address(healthy_wallet1.address().into()),
+        900_000_000, // 0.9 * 1e9
+    ).await.value;
+    println!("icr : {}", icr);
+
+    let debt = trove_manager_abi::get_trove_debt(
+        &contracts.asset_contracts[0].trove_manager,
+        Identity::Address(healthy_wallet1.address().into()),
+    ).await.value;
+    println!("debt: {}", debt);
+
+    let coll = trove_manager_abi::get_trove_coll(
+        &contracts.asset_contracts[0].trove_manager,
+        Identity::Address(healthy_wallet1.address().into()),
+    ).await.value;
+    println!("coll: {}", coll);
+
+    let cnt = trove_manager_abi::get_trove_owners_count(
+        &contracts.asset_contracts[0].trove_manager,
+    ).await.value;
+    println!("cnt: {}", cnt);
+}
```
