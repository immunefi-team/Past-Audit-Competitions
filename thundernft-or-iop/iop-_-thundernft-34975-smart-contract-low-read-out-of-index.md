# IOP \_ ThunderNFT 34975 - \[Smart Contract - Low] Read out of index

Submitted on Mon Sep 02 2024 03:56:37 GMT-0400 (Atlantic Standard Time) by @jasonxiale for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34975

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/asset\\_manager>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

In `asset_manager.get_supported_asset` and `execution_manager.get_whitelisted_strategy`, there is a out-of-index read issue.

## Vulnerability Details

In [asset\_manager.get\_supported\_asset](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/asset_manager/src/main.sw#L75-L81), the function checks `index <= storage.assets.len`, which is incorrect, because the index starts from 0.

```rust
 74     #[storage(read)]
 75     fn get_supported_asset(index: u64) -> Option<AssetId> {
 76         let len = storage.assets.len();
 77         require(len != 0, AssetManagerErrors::ZeroLengthVec);
 78         require(index <= len, AssetManagerErrors::IndexOutOfBound); <<<--- Here is not correct
 79 
 80         storage.assets.get(index).unwrap().try_read()
 81     }
```

Same issue happens in [execution\_manager.get\_whitelisted\_strategy](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/execution_manager/src/main.sw#L75-L82):

```rust
 75     #[storage(read)]
 76     fn get_whitelisted_strategy(index: u64) -> Option<ContractId> {
 77         let len = storage.strategies.len();
 78         require(len != 0, ExecutionManagerErrors::ZeroLengthVec);
 79         require(index <= len, ExecutionManagerErrors::IndexOutOfBound); <<<--- Here is not correct
 80 
 81         storage.strategies.get(index).unwrap().try_read()
 82     }
```

## Impact Details

function read out-of-index, and tx will revert with unexpected message

## References

Add any relevant links to documentation or code

## Proof of concept

## Proof of Concept

Please generate a Rust test template under `thunder_exchange` folder, and puts the following code in `thunder_exchange/tests/harness.rs` and run `cargo test -- --nocapture`

```bash
cargo test

running 1 test
test sell_taker_can_sell_less_token ... FAILED

failures:

---- sell_taker_can_sell_less_token stdout ----
balance: 200
balance: 199
get_supported_asset: Some(0000000000000000000000000000000000000000000000000000000000000000)
Error: Transaction(Reverted { reason: "Revert(0)", revert_id: 0, receipts: [Call { id: 0000000000000000000000000000000000000000000000000000000000000000, to: d7f461207387619982cbb2d36ae2921ceaa1953d6828faa3ff384e85505d657e, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 2190, param1: 10480, param2: 10507, pc: 11696, is: 11696 }, Revert { id: d7f461207387619982cbb2d36ae2921ceaa1953d6828faa3ff384e85505d657e, ra: 0, pc: 34884, is: 11696 }, ScriptResult { result: Revert, gas_used: 2362 }] })


failures:
    sell_taker_can_sell_less_token

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 1.78s
```

As the code shows above, the tx is reverted

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

    pool_instance
        .clone()
        .with_account(wallet_3.clone())
        .methods()
        .withdraw(AssetId::zeroed(), 1)
        .with_variable_output_policy(VariableOutputPolicy::Exactly(1))
        .with_contracts(&[&strategy_fixed_price_sale_instance, &execution_manager_instance, &thunder_exchange_instance, &nft_instance, &asset_manager_instance, &pool_instance, &royalty_manager_instance])
        .call()
        .await?;
    println!("balance: {:?}", pool_instance.clone().methods().balance_of(Identity::Address(wallet_3.address().into()), AssetId::zeroed()).call().await?.value);

    //asset_manager_methods.remove_asset(0).with_tx_policies(TxPolicies::default()).call().await?;
    println!("get_supported_asset: {:?}", asset_manager_methods.get_supported_asset(0).with_tx_policies(TxPolicies::default()).call().await?.value);
    println!("get_supported_asset: {:?}", asset_manager_methods.get_supported_asset(1).with_tx_policies(TxPolicies::default()).call().await?.value);

    pool_instance
        .clone()
        .with_account(wallet_3.clone())
        .methods()
        .withdraw(AssetId::zeroed(), 1)
        .with_variable_output_policy(VariableOutputPolicy::Exactly(1))
        .with_contracts(&[&strategy_fixed_price_sale_instance, &execution_manager_instance, &thunder_exchange_instance, &nft_instance, &asset_manager_instance, &pool_instance, &royalty_manager_instance])
        .call()
        .await?;
    println!("balance: {:?}", pool_instance.clone().methods().balance_of(Identity::Address(wallet_3.address().into()), AssetId::zeroed()).call().await?.value);

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
