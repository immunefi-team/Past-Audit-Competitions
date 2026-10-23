# IOP \_ ThunderNFT 34973 - \[Smart Contract - Low] royalty\_managerregister\_royalty\_info might not work

Submitted on Mon Sep 02 2024 03:53:58 GMT-0400 (Atlantic Standard Time) by @jasonxiale for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34973

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/royalty\\_manager>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`royalty_manager.register_royalty_info` is used to set `RoyaltyInfo` by the owner/admin of NFT token, however, in current implementation, there is an issue that when a NFT has both owner and admin, the `RoyaltyInfo` can only be set by the owner.

## Vulnerability Details

According to [royalty\_manager.register\_royalty\_info](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/royalty_manager/src/main.sw#L48-L81), the `RoyaltyInfo` can be set by NFT's owner or admin. But there is a flow in [royalty\_manager#L55-L65](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/royalty_manager/src/main.sw#L55-L65), **if the NFT has an owner, and the `msg_sender` is not the owner, the function will revert even the `msg_sender` is the NFT's amdin**

```solidity
 48     fn register_royalty_info(
 49         collection: ContractId,
 50         receiver: Identity,
 51         fee: u64
 52     ) {
 53         let ownable = abi(Ownable, collection.into());
 54 
 55         if (ownable.owner().is_some()) { <<<--- In this if branch, if the msg_sender is not the NFT's owner, the function will revert.
 56             let caller = msg_sender().unwrap();
 57             let collection_owner = ownable.owner().unwrap();
 58             require(caller == collection_owner, RoyaltyManagerErrors::CallerMustBeOwnerOrAdmin);
 59         } else if (ownable.admin().is_some()) {
 60             let caller = msg_sender().unwrap();
 61             let collection_admin = ownable.admin().unwrap();
 62             require(caller == collection_admin, RoyaltyManagerErrors::CallerMustBeOwnerOrAdmin);
 63         } else {
 64             revert(111)
 65         }
 66 
 67         require(fee <= storage.fee_limit.read(), RoyaltyManagerErrors::FeeHigherThanLimit);
 68 
 69         let info = RoyaltyInfo {
 70             collection: collection,
 71             receiver: receiver,
 72             fee: fee
 73         };
 74 
 75         let option_info: Option<RoyaltyInfo> = Option::Some(info);
 76         storage.royalty_info.insert(collection, option_info);
 77 
 78         log(RoyaltyRegistryEvent {
 79             royalty_info: info
 80         });
 81     }
```

## Impact Details

If a NFT has both owner and admin, the admin can't call `royalty_manager.register_royalty_info`.

## References

Add any relevant links to documentation or code

## Proof of concept

## Proof of Concept

## POC

1. please use the follow code as NFT contract

```rust
contract;

use interfaces::{erc_owner_and_admin_interface::*};

use std::{
    auth::msg_sender,
    contract_id::ContractId,
    logging::log,
    hash::Hash,
    identity::Identity,
    revert::revert,
    storage::storage_map::*
};

storage {
    owner: Option<Identity> = None,
    admin: Option<Identity> = None,
}

impl ERCOwnerAndAdmin for Contract {
    #[storage(read, write)]
    fn initialize(owner_: Identity, admin_: Identity) {
        storage.owner.write(Option::Some(owner_));
        storage.admin.write(Option::Some(admin_));
    }

    #[storage(read)]
    fn owner() -> Option<Identity> {
        storage.owner.read()
    }

    #[storage(read)]
    fn admin() -> Option<Identity> {
        storage.admin.read()
    }
}
```

2. Please generate a Rust test template under `thunder_exchange` folder, and puts the following code in `thunder_exchange/tests/harness.rs` and run `cargo test -- --nocapture`

```bash
cargo test

running 1 test
test sell_taker_can_sell_less_token ... FAILED

failures:

---- sell_taker_can_sell_less_token stdout ----
Error: Transaction(Reverted { reason: "CallerMustBeOwnerOrAdmin", revert_id: 18446744073709486080, receipts: [Call { id: 0000000000000000000000000000000000000000000000000000000000000000, to: deba0f7ed7c8cd7a93d4a269c8d00542c17612f5f9dcc9365b61f7761d29a147, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 9976, param1: 10480, param2: 10509, pc: 13448, is: 13448 }, Call { id: deba0f7ed7c8cd7a93d4a269c8d00542c17612f5f9dcc9365b61f7761d29a147, to: fd530c60247a983d1f5b5d40a66c22337f7cfb5a315773bf10b6df071dcfccbc, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 7518, param1: 67107840, param2: 67106816, pc: 54888, is: 54888 }, ReturnData { id: fd530c60247a983d1f5b5d40a66c22337f7cfb5a315773bf10b6df071dcfccbc, ptr: 67104256, len: 48, digest: 35820f2cd14ad6c5e774325564a00969304f5e6db22a69aefc1be523558ef9b3, pc: 56528, is: 54888, data: Some(00000000000000010000000000...) }, Call { id: deba0f7ed7c8cd7a93d4a269c8d00542c17612f5f9dcc9365b61f7761d29a147, to: fd530c60247a983d1f5b5d40a66c22337f7cfb5a315773bf10b6df071dcfccbc, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 2926, param1: 67103232, param2: 67102208, pc: 54888, is: 54888 }, ReturnData { id: fd530c60247a983d1f5b5d40a66c22337f7cfb5a315773bf10b6df071dcfccbc, ptr: 67099648, len: 48, digest: 35820f2cd14ad6c5e774325564a00969304f5e6db22a69aefc1be523558ef9b3, pc: 56528, is: 54888, data: Some(00000000000000010000000000...) }, LogData { id: deba0f7ed7c8cd7a93d4a269c8d00542c17612f5f9dcc9365b61f7761d29a147, ra: 0, rb: 10868993773200300074, ptr: 67098624, len: 8, digest: cd04a4754498e06db5a13c5f371f1f04ff6d2470f24aa9bd886540e5dce77f70, pc: 26808, is: 13448, data: Some(0000000000000002) }, Revert { id: deba0f7ed7c8cd7a93d4a269c8d00542c17612f5f9dcc9365b61f7761d29a147, ra: 18446744073709486080, pc: 26816, is: 13448 }, ScriptResult { result: Revert, gas_used: 9772 }] })


failures:
    sell_taker_can_sell_less_token

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 1.62s

error: test failed, to rerun pass `--test integration_tests`
```

As shown above, tx will revert when the `royalty_manager.register_royalty_info` is called by `wallet_3`, which is set as admin for ERCOwnerAndAdmin. And if the `royalty_manager.register_royalty_info` is called by `wallet_1` (which is set as owner of ERCOwnerAndAdmin), the function will not revert.

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
        name = "ERCOwnerAndAdmin",
        abi = "out/debug/erc_owner_and_admin-abi.json"
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

    let erc_owner_and_admin_id = Contract::load_from(
        "./out/debug/erc_owner_and_admin.bin",
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

    let erc_owner_and_admin_instance = ERCOwnerAndAdmin::new(erc_owner_and_admin_id.clone(), wallet.clone());
    let erc_owner_and_admin_methods = erc_owner_and_admin_instance.clone().methods();
    erc_owner_and_admin_methods.initialize(wallet_1.address().into(), wallet_3.address().into()).with_tx_policies(TxPolicies::default()).call().await?;

    let royalty_manager_instance = RoyaltyManager::new(royalty_manager_id.clone(), wallet.clone());
    let royalty_manager_methods = royalty_manager_instance.clone().methods();
    royalty_manager_methods.initialize().with_tx_policies(TxPolicies::default()).call().await?;

    thunder_exchange_methods.set_pool(pool_id.clone()).with_tx_policies(TxPolicies::default()).call().await?;
    thunder_exchange_methods.set_execution_manager(execution_manager_id.clone()).with_tx_policies(TxPolicies::default()).call().await?;
    thunder_exchange_methods.set_royalty_manager(royalty_manager_id.clone()).with_tx_policies(TxPolicies::default()).call().await?;
    thunder_exchange_methods.set_asset_manager(asset_manager_id.clone()).with_tx_policies(TxPolicies::default()).call().await?;

    let nft_instance = NFT::new(nft_id.clone(), wallet_1.clone());

    let nft_contract_id: ContractId = nft_id.into();
    let identity   = Identity::Address(Address::from(wallet.address()));

    nft_instance.methods().constructor(identity).call().await?;

    royalty_manager_instance
        .clone()
        .methods()
        .set_royalty_fee_limit(100u64)
        .call()
        .await?;

    //royalty_manager_instance
    //    .clone()
    //    .with_account(wallet_1.clone())
    //    .methods()
    //    .register_royalty_info(erc_owner_and_admin_id.clone(), wallet_2.address().into(), 1)
    //    .with_contracts(&[&strategy_fixed_price_sale_instance, &execution_manager_instance, &thunder_exchange_instance, &nft_instance, &asset_manager_instance, &pool_instance, &royalty_manager_instance, &erc_owner_and_admin_instance])
    //    .call()
    //    .await?;

    royalty_manager_instance
        .clone()
        .with_account(wallet_3.clone())
        .methods()
        .register_royalty_info(erc_owner_and_admin_id.clone(), wallet_2.address().into(), 10)
        .with_contracts(&[&strategy_fixed_price_sale_instance, &execution_manager_instance, &thunder_exchange_instance, &nft_instance, &asset_manager_instance, &pool_instance, &royalty_manager_instance, &erc_owner_and_admin_instance])
        .call()
        .await?;

    Ok(())
}
```
