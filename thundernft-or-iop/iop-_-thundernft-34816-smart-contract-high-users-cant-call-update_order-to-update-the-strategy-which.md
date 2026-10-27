# IOP \_ ThunderNFT 34816 - \[Smart Contract - High] users cant call update\_order to update the strategy

Submitted on Tue Aug 27 2024 19:13:17 GMT-0400 (Atlantic Standard Time) by @zeroK for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34816

Report type: Smart Contract

Report severity: High

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/thunder\\_exchange>

Impacts:

* Permanent freezing of NFTs

## Description

## Brief/Intro

the `update_order` function in the thunder\_exchange meant to be used to update some important parameter for order makers, makers should be allowed to update the `price and amount` and `strategy and paymentAsset` when it get updated(new token or strategy added to whitelist), however the implementation design of thunder\_exchange and fixed strategy will not allow users to update the strategy address correctly, this happens because when the `thunder_exchange#update_order` get called the \_validate\_maker\_order\_input function will invoked which require the nonce to be above zero and when the `fixed_strategy#update_order` get called the line below will get the user order nonce in the new strategy:

```sway
/// Updates sell MakerOrder if the nonce is in the right range
#[storage(read, write)]
fn _update_sell_order(order: MakerOrder) {
    let nonce = _user_sell_order_nonce(order.maker); //@audit get the nonce for the maker, this will be zero in new strategy contract since user have no storage data in the new deployed contract
    let min_nonce = _user_min_sell_order_nonce(order.maker);

    if ((min_nonce < order.nonce) && (order.nonce <= nonce)) { //@audit order.nonce is 1 and nonce is zero so this won't execute and revert happened
        // Update sell order
        let sell_order = _sell_order(order.maker, order.nonce);
        _validate_updated_order(sell_order, order);
        storage.sell_order.insert((order.maker, order.nonce), Option::Some(order));
    } else {
        revert(115);
    }
}

```

the issue exist because the maker who want to update its order, he's forced to set nonce to 1 and above(since place\_order does not allow creating order with zero nonce) and in the other hand when new strategy get deployed the user storage variable will be zero, this is because the whole codebase is not upgradable and since its not upgradable the storage data from old strategy will not be saved/written in the new strategy contract.

this will prevent updating any parameter like price or nonce or payment asset and strategy which lead to stuck the NFTs.

## Vulnerability Details

when user want to update its order, first thing to do is calling update order in the thunder\_exchange which invokes some sanity checks as shown below:

```sway
    /// Updates the existing MakerOrder
    #[storage(read), payable]
    fn update_order(order_input: MakerOrderInput) {
        _validate_maker_order_input(order_input);

        let strategy = abi(ExecutionStrategy, order_input.strategy.bits());
        let order = MakerOrder::new(order_input);
        match order.side {
            Side::Buy => {
                // Checks if user has enough bid balance
                let pool_balance = _get_pool_balance(order.maker, order.payment_asset);
                require(order.price <= pool_balance, ThunderExchangeErrors::AmountHigherThanPoolBalance); // the price bidder set should be smaller or equal to their balance
            },
            Side::Sell => {}, // if order is selling nft then nothing to check for
        }

        strategy.update_order(order); //@audit call update_order in the new strategy

        log(OrderUpdated {
            order
        });
    }


/// Validates the maker order input
#[storage(read)]
fn _validate_maker_order_input(input: MakerOrderInput) {
    require(input.maker != ZERO_ADDRESS, ThunderExchangeErrors::MakerMustBeNonZeroAddress);
    require(input.maker == get_msg_sender_address_or_panic(), ThunderExchangeErrors::CallerMustBeMaker);

    // !!! Info !!! -> This check will be removed as mentioned
    require(
        (storage.min_expiration.read() <= input.expiration_range) &&
        (input.expiration_range <= storage.max_expiration.read()),
        ThunderExchangeErrors::ExpirationRangeOutOfBound
    );
 
    require(input.nonce > 0, ThunderExchangeErrors::NonceMustBeNonZero); //@audit nonce should be above zero
    require(input.price > 0, ThunderExchangeErrors::PriceMustBeNonZero); 
    require(input.amount > 0, ThunderExchangeErrors::AmountMustBeNonZero); // no matter how much you set this will be 1

    // Checks if the strategy contracId in the order is whitelisted
    let execution_manager_addr = storage.execution_manager.read().unwrap().bits();
    let execution_manager = abi(ExecutionManager, execution_manager_addr);
    require(execution_manager.is_strategy_whitelisted(input.strategy), ThunderExchangeErrors::StrategyNotWhitelisted); // used strategy should be whitelisted

    // Checks if the payment_asset in the order is supported
    let asset_manager_addr = storage.asset_manager.read().unwrap().bits();
    let asset_manager = abi(AssetManager, asset_manager_addr);
    require(asset_manager.is_asset_supported(input.payment_asset), ThunderExchangeErrors::AssetNotSupported);
}

```

as shown above, the nonce input should be above zero, this will prevent users to update their orders when new strategy get deployed and whitelisted, this is because the old storage data in the old strategy does not copied to the new strategy. the issue exist in line belows:

```sway

/// Updates sell MakerOrder if the nonce is in the right range
#[storage(read, write)]
fn _update_sell_order(order: MakerOrder) {
    let nonce = _user_sell_order_nonce(order.maker); //@audit get the nonce for the maker, this will be zero in new strategy contract since user have no storage data in the new deployed contract
    let min_nonce = _user_min_sell_order_nonce(order.maker);

    if ((min_nonce < order.nonce) && (order.nonce <= nonce)) { // @audit this check won't pass since order.nonce is 1 and nonce is zero
        // Update sell order
        let sell_order = _sell_order(order.maker, order.nonce);
        _validate_updated_order(sell_order, order);
        storage.sell_order.insert((order.maker, order.nonce), Option::Some(order));
    } else {
        revert(115);
    }
}


#[storage(read)]
fn _user_sell_order_nonce(address: Address) -> u64 {
    let status = storage.user_sell_order_nonce.get(address).try_read(); //@audit this will be zero since there is no data for maker in the new strategy that deployed
    match status {
        Option::Some(nonce) => nonce,
        Option::None => 0, //@audit zero returned 
    }
}

```

as shown, the call to update will revert and other functions won't be able to execute since strategy or paymentAsset not updated.

this will only affect the NFT seller since the owner of the NFT can not cancel or execute its nft sell order, bidder can transfer out their tokens in the pool

## Impact Details

Permanent freezing of NFTs when the strategy get updated.

## References

the most recommended fix is updating the codebase to upgradable codebase with following the requirements to set the codebase to correct upgradable contracts.

another fix is allowing updating strategy with nonce zero to avoid reverting, but we don't recommend this.

## Proof of concept

## Proof of Concept

check the report ID: 34567 notes to execute the POC below:

```sway
contract;

// please take the mentioned note in reportID: 34567  in mind since some lines in the codebase should be commented

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
 
#[test(should_revert)] // change this to #[test()] when the codebase modified
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

    //set functions

    execution_manager.add_strategy(stratgey_fixed);

    thunder_exch.set_asset_manager(asset_manger_contract_id);
    thunder_exch.set_pool(pool_contract);
    thunder_exch.set_execution_manager(execution_manger_contract);
    // thunder_exch.set_protocol_fee_recipient(Identity::Address::from(0xf9b8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07)); // diff from maker
    thunder_exch.set_royalty_manager(royalty_manger_contract);

    //----------------------execute attack --------------------------------//


    // STEP 1: we assume that the order below is created when fixed strategy version 1 deployed, since sway does not allow
    //         to create two contracts using same contract ID as we did above in line 55. 

    let caller = Address::from(0xf9b8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07);

    let extra_parameter = ExtraParams {
        extra_address_param: Address::zero(),
        extra_contract_param: ContractId::zero(),
        extra_u64_param: 0u64
    };

    let maker_order = MakerOrderInput {
     side: Side::Sell,
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

    //STEP 2: this skipped for reason above:

    /*
        let fixed_strategy = abi(ExecutionStrategy, strategy_fixed_price_sale::CONTRACT_ID);
        fixed_strategy.initialize(exchange_contract_id);

        let stratgey_fixed_ver2 = ContractId::from(strategy_fixed_price_sale::CONTRACT_ID); //CONTRACT_ID will cause revert because its used before(if not reverting then we use same fixed strategy which is not our case)
       

    let maker_order_update = MakerOrderInput {
     side: Side::Sell,
     maker:caller,
     collection: ContractId::from(0x7777777777777777777777777777777777777777777777777777777777777777),
     token_id: SubId::from(0x7777777777777777777777777777777777777777777777777777777777777778),
     price: 20u64,
     amount: 1u64,
     nonce: 1u64,
     strategy:stratgey_fixed_ver2,  // this changed but we comment this because of sway current test tool does not allow using same contract id to deploy same contract
     payment_asset: asset_to_add,
     expiration_range: 1000u64,
     extra_params: extra_parameter,
    };
    */

    // maker_order should be set to maker_order_update if STEP 2 not reverted
    thunder_exch.cancel_order(maker_order); //this will revert since the strategy changed and user did not have storage data in the new strategy

}

```
