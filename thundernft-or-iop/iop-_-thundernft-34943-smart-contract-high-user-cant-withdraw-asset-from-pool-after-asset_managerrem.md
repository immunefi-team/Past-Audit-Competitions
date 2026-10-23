# IOP \_ ThunderNFT 34943 - \[Smart Contract - High] User cant withdraw asset from pool after asset\_mana

Submitted on Sun Sep 01 2024 16:12:44 GMT-0400 (Atlantic Standard Time) by @jasonxiale for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34943

Report type: Smart Contract

Report severity: High

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/pool>

Impacts:

* Permanent freezing of funds

## Description

## Brief/Intro

`asset_manager.add_asset` and `asset_manager.remove_asset` are used to control which asset are allowed in the pool. And when `pool.deposit` and `pool.withdraw` are called, the functions checks if the assetId is supported by `asset_manager.is_asset_supported`.

However there is an issue that after `asset_manager.remove_asset` is called, the corresponding asset in the pool can't be withdrawn.

## Vulnerability Details

As shown in [pool.withdraw](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/pool/src/main.sw#L105-L124), when a user calls the function to withdraw asset, the function will check if the asset is supported in [pool#L112](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/pool/src/main.sw#L112), and if not, the function will revert.

```rust
105     fn withdraw(asset: AssetId, amount: u64) {
106         let sender = msg_sender().unwrap();
107         let current_balance = _balance_of(sender, asset);
108         require(current_balance >= amount, PoolErrors::AmountHigherThanBalance);
109 
110         let asset_manager_addr = storage.asset_manager.read().unwrap().bits();
111         let asset_manager = abi(AssetManager, asset_manager_addr);
112         require(asset_manager.is_asset_supported(asset), PoolErrors::AssetNotSupported); <<<<<----- here checks if the assetId is supported by asset_manager
113 
114         let new_balance = current_balance - amount;
115         storage.balance_of.insert((sender, asset), new_balance);
116 
117         transfer(sender, asset, amount);
118 
119         log(Withdrawal {
120             address: sender,
121             asset,
122             amount,
123         });
124     }
```

## Impact Details

Please consider a case that:

1. the asset\_manager's owner calls `asset_manager.add_asset` to add assetId\_x
2. Alice deposit some `assetId_x` by calling `pool.deposit`
3. after a while, the asset\_manager's owner calls `asset_manager.remove_asset` to remove the assetId\_x
4. When Alice calls `pool.withdraw` to withdraw her assetId\_x, the function will revert in [pool#L112](https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/pool/src/main.sw#L112)

## References

Add any relevant links to documentation or code

## Proof of concept

## Proof of Concept

Please generate a Rust test template under `thunder_exchange` folder, and puts the following code in `thunder_exchange/tests/harness.rs` and run `cargo test -- --nocapture`

```bash
cargo test -- --nocapture
running 1 test
test sell_taker_can_sell_less_token ... balance: 200
balance: 199
Error: Transaction(Reverted { reason: "AssetNotSupported", revert_id: 18446744073709486080, receipts: [Call { id: 0000000000000000000000000000000000000000000000000000000000000000, to: 2e644050002d6495bace13664683a34e6cb94f8b2fdb19e423a67df6434fa1b6, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 13948, param1: 10480, param2: 10496, pc: 13232, is: 13232 }, Call { id: 2e644050002d6495bace13664683a34e6cb94f8b2fdb19e423a67df6434fa1b6, to: d7f461207387619982cbb2d36ae2921ceaa1953d6828faa3ff384e85505d657e, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 4624, param1: 67106489, param2: 67105465, pc: 66304, is: 66304 }, ReturnData { id: d7f461207387619982cbb2d36ae2921ceaa1953d6828faa3ff384e85505d657e, ptr: 67102905, len: 1, digest: 6e340b9cffb37a989ca544e6bb780a2c78901d3fb33738768511a30617afa01d, pc: 70976, is: 66304, data: Some(00) }, LogData { id: 2e644050002d6495bace13664683a34e6cb94f8b2fdb19e423a67df6434fa1b6, ra: 0, rb: 15586886228264746057, ptr: 67101881, len: 8, digest: cd04a4754498e06db5a13c5f371f1f04ff6d2470f24aa9bd886540e5dce77f70, pc: 30996, is: 13232, data: Some(0000000000000002) }, Revert { id: 2e644050002d6495bace13664683a34e6cb94f8b2fdb19e423a67df6434fa1b6, ra: 18446744073709486080, pc: 31004, is: 13232 }, ScriptResult { result: Revert, gas_used: 13611 }] })
FAILED

failures:

failures:
    sell_taker_can_sell_less_token

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 1.84s
```

As the code shows, the `pool.withdraw` will be revert with `AssetNotSupported`

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

    asset_manager_methods.remove_asset(0).with_tx_policies(TxPolicies::default()).call().await?;

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
