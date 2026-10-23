# IOP \_ ThunderNFT 34567 - \[Smart Contract - Medium] users with current bid order can not update their

Submitted on Thu Aug 15 2024 23:17:52 GMT-0400 (Atlantic Standard Time) by @zeroK for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34567

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/thunder\\_exchange>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value
* Block stuffing

## Description

## Brief/Intro

the function `update_order` meant to allow users with current active bid to update their order important input, however there is critical check that prevent users to update their bid and they all forced to cancel their bid, this will lead to lose of gas only for buy order users. this happens because the update\_order calls the update\_order in the fixed strategy which in return it calls the \_update\_buy\_order which checks if the payment asset in unchanged, the payment asset is same asset that set in whitelist when assetManger call add asset or remove it by calling remove asset.

same thing can be applied to sell order users which is more critical situation compared to buy order users, we can take the steps below that can happen when the paymentAsset removed and new asset added:

* Alice create sell order(listing NFT) by calling place\_order with paymentAsset == USDT.
* 10 users create buy order to bid on alice NFT, and their payment asset == USDT.
* for some reason, USDT removed from whitelisted address by calling the `assetManger.sw#remove_asset` and ETH added as paymentAsset by calling `add_asset` function.
* because of the check`(order.unwrap().payment_asset == updated_order.payment_asset)` which check the payment asset of old order and updated one in `_validate_updated_order` which invoked by \_update\_buy and \_update\_sell alice and all other 10 users with bid order can not update their bid order payment asset to eth, same true for alice.
* this way alice and other users are forced to remove their orders(cancle it) and re create same order again and add new bid on the alice NFT which lead to lose of gas too.

## Vulnerability Details

the function update\_order implemented as below:

```sway
    #[storage(read), payable]
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
            Side::Sell => {}, // if order is selling nft then nothing to check for
        }

        strategy.update_order(order);

        log(OrderUpdated {
            order
        });
    }


```

which invoke the update\_order function for both buy and sell order:

```sway
    /// Updates the existing MakerOrder of the user
    /// Only callable by Thunder Exchange contract
    #[storage(read, write)]
    fn update_order(order: MakerOrder) {
        // only_exchange();

        match order.side {
            Side::Buy => {
                _update_buy_order(order)
            },
            Side::Sell => {
                _update_sell_order(order)
            }
        }
    }

```

both \_update\_buy\_order and \_update\_sell\_order invoke the \_validate\_updated\_order function as shown below

```sway
/// Updates buy MakerOrder if the nonce is in the right range
#[storage(read, write)]
fn _update_buy_order(order: MakerOrder) {
    let nonce = _user_buy_order_nonce(order.maker); // nonce should not be changed for update purpose same to maker address

    if ((order.nonce <= nonce)) {
        // Update buy order
        let buy_order = _buy_order(order.maker, order.nonce); // get the old order
        _validate_updated_order(buy_order, order);
        storage.buy_order.insert((order.maker, order.nonce), Option::Some(order));
    } else {
        revert(114);
    }
}

/// Updates sell MakerOrder if the nonce is in the right range
#[storage(read, write)]
fn _update_sell_order(order: MakerOrder) {
    let nonce = _user_sell_order_nonce(order.maker);
    let min_nonce = _user_min_sell_order_nonce(order.maker);

    if ((min_nonce < order.nonce) && (order.nonce <= nonce)) {
        // Update sell order
        let sell_order = _sell_order(order.maker, order.nonce);
        _validate_updated_order(sell_order, order);
        storage.sell_order.insert((order.maker, order.nonce), Option::Some(order));
    } else {
        revert(115);
    }
}

#[storage(read)]
fn _validate_updated_order(order: Option<MakerOrder>, updated_order: MakerOrder) {
    require(
        (order.unwrap().maker == updated_order.maker) &&
        (order.unwrap().collection == updated_order.collection) &&
        (order.unwrap().token_id == updated_order.token_id) &&
        (order.unwrap().payment_asset == updated_order.payment_asset) && //@audit payment asset should be allowed to be changed
        _is_valid_order(order),
        StrategyFixedPriceErrors::OrderMismatchedToUpdate
    );
}

```

this way its impossible to update the current bids and listing NFT in this situation and lead to cancel all orders and re execute it again.

## Impact Details

users can't update their orders when paymentAsset removed and another one added.

## Recommend

there is no reason to check for updated order payment if its same as old one since in the flow we check the updated order if its payment asset is whitelisted or not.

## Proof of concept

## Proof of Concept

run this poc in contract-v1/tests/src/main.sw with following these details below:

## details should follow to execute the test in sway

* for anyone want to create POC using sway itself, i created a simple template do so, all thanks to @theschnilch who helped a lot to make this test file valid.

to create POC or doing some tests in sway do the follow:

* create forc.toml in contracts-v1 and add the below in the forc.toml:

```
[workspace]
members = ["asset_manager", "erc721", "execution_manager", "execution_strategies/strategy_fixed_price_sale" ,"libraries", "interfaces" , "pool", "royalty_manager", "thunder_exchange", "tests"]

```

* and then create new folder called tests and add your file, the path will be something like this : `contracts-v1/tests/src/main.sw`

then in your forc.toml in tests folder add the below:

```
[project]
authors = ["0k"]
entry = "main.sw"
license = "Apache-2.0"
name = "test_contract"

[dependencies]
interfaces = { path = "../interfaces" }
libraries = { path = "../libraries" }


[contract-dependencies]

asset_manager = { path = "../asset_manager" }

thunder_exchange = { path = "../thunder_exchange" }

pool = { path = "../pool" }

execution_manager = { path = "../execution_manager" }

royalty_manager = { path = "../royalty_manager" }

strategy_fixed_price_sale = {path = "../execution_strategies/strategy_fixed_price_sale"}
```

run forc test in `smart-contracts/contracts-v1`

* and after that you can add the code below in your main.sw:

```sway
contract;

 use interfaces::{
    thunder_exchange_interface::{ThunderExchange},
    royalty_manager_interface::*,
    asset_manager_interface::*,
    execution_manager_interface::ExecutionManager,
    execution_strategy_interface::*,
    pool_interface::Pool,
};

use libraries::{
    msg_sender_address::*,
    constants::*,
    order_types::*,
    ownable::*,
};

use std::{
    block::timestamp,
    auth::*,
    call_frames::*,
    context::*,
    contract_id::ContractId,
    logging::log,
    revert::require,
    storage::storage_map::*,
    asset::*,
};
 
#[test(should_revert)]
fn test_attack() {
    initialize_functions();


}

fn initialize_functions() {

    //initialize all contracts 

    let thunder_exch = abi(ThunderExchange, thunder_exchange::CONTRACT_ID);
    thunder_exch.initialize();

    let asset_mngr = abi(AssetManager, asset_manager::CONTRACT_ID);
    asset_mngr.initialize();

    // required for initialize below contracts
    let exchange_contract_id = ContractId::from(thunder_exchange::CONTRACT_ID);

    let asset_manger_contract_id = ContractId::from(asset_manager::CONTRACT_ID);

    let fixed_strategy = abi(ExecutionStrategy, strategy_fixed_price_sale::CONTRACT_ID);
    fixed_strategy.initialize(exchange_contract_id);

    let execution_manager = abi(ExecutionManager, execution_manager::CONTRACT_ID);
    execution_manager.initialize();

    let royalty_manager = abi(RoyaltyManager, royalty_manager::CONTRACT_ID);
    royalty_manager.initialize();

    //get contract IDs 

    let execution_manger_contract = ContractId::from(execution_manager::CONTRACT_ID);
    let royalty_manger_contract = ContractId::from(royalty_manager::CONTRACT_ID);
    let pool_contract = ContractId::from(pool::CONTRACT_ID);
    let stratgey_fixed = ContractId::from(strategy_fixed_price_sale::CONTRACT_ID);


    let asset_to_add = AssetId::base();
    
    asset_mngr.add_asset(asset_to_add);


    let asset_id = AssetId::bits(asset_to_add);


    let pool = abi(Pool, pool::CONTRACT_ID);
    pool.initialize(exchange_contract_id, asset_manger_contract_id);

    let caller = Address::from(0xf8f8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07);

    pool.update_balance(asset_to_add, 100u64);

    //set functions

    execution_manager.add_strategy(stratgey_fixed);

    thunder_exch.set_asset_manager(asset_manger_contract_id);
    thunder_exch.set_pool(pool_contract);
    thunder_exch.set_execution_manager(execution_manger_contract);
    // thunder_exch.set_protocol_fee_recipient(Identity::Address::from(0xf9b8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07)); // diff from maker
    thunder_exch.set_royalty_manager(royalty_manger_contract);

    //----------------------execute attack --------------------------------//

    let caller = Address::from(0xf9b8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07);

    let extra_parameter = ExtraParams {
        extra_address_param: Address::zero(),
        extra_contract_param: ContractId::zero(),
        extra_u64_param: 0u64
    };

    let maker_order = MakerOrderInput {
     side: Side::Buy,
     maker:caller,
     collection: ContractId::from(0x7777777777777777777777777777777777777777777777777777777777777777),
     token_id: SubId::from(0x7777777777777777777777777777777777777777777777777777777777777778),
     price: 10u64,
     amount: 1u64,
     nonce: 1u64,
     strategy:stratgey_fixed,
     payment_asset: asset_to_add,
     expiration_range: 1000u64,
     extra_params: extra_parameter,
    };


    thunder_exch.place_order(maker_order);

    //now remove the whitelist token and add another one 

    asset_mngr.remove_asset(0);

    let asset_to_add1 = AssetId::from(0x7777_7777_7777_7777_7777_7777_7777_7777_7777_7777_7777_7777_7777_7777_7777_7777);
    
    asset_mngr.add_asset(asset_to_add1);

    //call update with new paymentAsset

    pool.update_balance(asset_to_add1, 100u64);

    let maker_order1 = MakerOrderInput {
     side: Side::Buy,
     maker:caller,
     collection: ContractId::from(0x7777777777777777777777777777777777777777777777777777777777777777),
     token_id: SubId::from(0x7777777777777777777777777777777777777777777777777777777777777778),
     price: 10u64,
     amount: 1u64,
     nonce: 1u64,
     strategy:stratgey_fixed,
     payment_asset: asset_to_add1,
     expiration_range: 1000u64,
     extra_params: extra_parameter,
    };

    thunder_exch.update_order(maker_order1);

}

```

## NOTES TO TAKE IN MIND BEFORE RUNNING THE POC:

thunderNFT does not support sway test so we create one, and while sway is under development, there is some change we made to the in scope contracts to execute the POC:

* first we set the EXTRAPARAMS to public by adding `pub` keyword to it: this because sway test files does not allow using internal function or struct.
* in \_validate\_maker\_order\_input we commented the line below, because sway itself does not allow using msg.sender or control it so its possible to execute call to place\_order with maker equal to msg.sender.

`// require(input.maker == get_msg_sender_address_or_panic(), ThunderExchangeErrors::CallerMustBeMaker); // maker of order should be the caller`

* we added a simple function to update the user balance in the pool:

this is because sway test files does not allow calling payable function with asset id and coins at all, so we need to find way to update the user pool balance to simulate user depositing tokens:

```sway
    #[storage(read, write)]
    fn update_balance(asset: AssetId, amount: u64) {
        let identity = Identity::Address( Address::from(0xf9b8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07));
        storage.balance_of.insert((identity, asset), amount); 
    }
```

## POC:

```sway
contract;

 use interfaces::{
    thunder_exchange_interface::{ThunderExchange},
    royalty_manager_interface::*,
    asset_manager_interface::*,
    execution_manager_interface::ExecutionManager,
    execution_strategy_interface::*,
    pool_interface::Pool,
};

use libraries::{
    msg_sender_address::*,
    constants::*,
    order_types::*,
    ownable::*,
};

use std::{
    block::timestamp,
    auth::*,
    call_frames::*,
    context::*,
    contract_id::ContractId,
    logging::log,
    revert::require,
    storage::storage_map::*,
    asset::*,
};
 
#[test(should_revert)]
fn test_attack() {
    initialize_functions();


}

fn initialize_functions() {

    //initialize all contracts 

    let thunder_exch = abi(ThunderExchange, thunder_exchange::CONTRACT_ID);
    thunder_exch.initialize();

    let asset_mngr = abi(AssetManager, asset_manager::CONTRACT_ID);
    asset_mngr.initialize();

    // required for initialize below contracts
    let exchange_contract_id = ContractId::from(thunder_exchange::CONTRACT_ID);

    let asset_manger_contract_id = ContractId::from(asset_manager::CONTRACT_ID);

    let fixed_strategy = abi(ExecutionStrategy, strategy_fixed_price_sale::CONTRACT_ID);
    fixed_strategy.initialize(exchange_contract_id);

    let execution_manager = abi(ExecutionManager, execution_manager::CONTRACT_ID);
    execution_manager.initialize();

    let royalty_manager = abi(RoyaltyManager, royalty_manager::CONTRACT_ID);
    royalty_manager.initialize();

    //get contract IDs 

    let execution_manger_contract = ContractId::from(execution_manager::CONTRACT_ID);
    let royalty_manger_contract = ContractId::from(royalty_manager::CONTRACT_ID);
    let pool_contract = ContractId::from(pool::CONTRACT_ID);
    let stratgey_fixed = ContractId::from(strategy_fixed_price_sale::CONTRACT_ID);


    let asset_to_add = AssetId::base();
    
    asset_mngr.add_asset(asset_to_add);


    let asset_id = AssetId::bits(asset_to_add);


    let pool = abi(Pool, pool::CONTRACT_ID);
    pool.initialize(exchange_contract_id, asset_manger_contract_id);

    let caller = Address::from(0xf8f8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07);

    pool.update_balance(asset_to_add, 100u64);

    //set functions

    execution_manager.add_strategy(stratgey_fixed);

    thunder_exch.set_asset_manager(asset_manger_contract_id);
    thunder_exch.set_pool(pool_contract);
    thunder_exch.set_execution_manager(execution_manger_contract);
    // thunder_exch.set_protocol_fee_recipient(Identity::Address::from(0xf9b8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07)); // diff from maker
    thunder_exch.set_royalty_manager(royalty_manger_contract);

    //----------------------execute attack --------------------------------//

    let caller = Address::from(0xf9b8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07);

    let extra_parameter = ExtraParams {
        extra_address_param: Address::zero(),
        extra_contract_param: ContractId::zero(),
        extra_u64_param: 0u64
    };

    let maker_order = MakerOrderInput {
     side: Side::Buy,
     maker:caller,
     collection: ContractId::from(0x7777777777777777777777777777777777777777777777777777777777777777),
     token_id: SubId::from(0x7777777777777777777777777777777777777777777777777777777777777778),
     price: 10u64,
     amount: 1u64,
     nonce: 1u64,
     strategy:stratgey_fixed,
     payment_asset: asset_to_add,
     expiration_range: 1000u64,
     extra_params: extra_parameter,
    };


    thunder_exch.place_order(maker_order);

    //now remove the whitelist token and add another one 

    asset_mngr.remove_asset(0);

    let asset_to_add1 = AssetId::from(0x7777_7777_7777_7777_7777_7777_7777_7777_7777_7777_7777_7777_7777_7777_7777_7777);
    
    asset_mngr.add_asset(asset_to_add1);

    //call update with new paymentAsset

    pool.update_balance(asset_to_add1, 100u64);

    let maker_order1 = MakerOrderInput {
     side: Side::Buy,
     maker:caller,
     collection: ContractId::from(0x7777777777777777777777777777777777777777777777777777777777777777),
     token_id: SubId::from(0x7777777777777777777777777777777777777777777777777777777777777778),
     price: 10u64,
     amount: 1u64,
     nonce: 1u64,
     strategy:stratgey_fixed,
     payment_asset: asset_to_add1,
     expiration_range: 1000u64,
     extra_params: extra_parameter,
    };

    thunder_exch.update_order(maker_order1);

}
```
