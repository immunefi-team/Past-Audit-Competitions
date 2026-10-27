# IOP \_ ThunderNFT 34714 - \[Smart Contract - Medium] owner of NFT who have sell orderlisting NFT can n

Submitted on Wed Aug 21 2024 22:08:06 GMT-0400 (Atlantic Standard Time) by @zeroK for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34714

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/thunder\\_exchange>

Impacts:

* Smart contract unable to operate due to lack of token funds
* Block stuffing

## Description

## Brief/Intro

the function `execute_order` meant to allow users to buy the NFT directly by sending the listed price for the nft and it allows seller of the NFT to accept specific bid offers. however its impossible for the seller/owner of the NFT to accept any bids because of the check that exist in the `_execute_sell_taker_order` function which checks if the msg\_asset() is valid when the user calls the execute\_order function, its impossible for the seller to have the asset\_id(the NFT) since the `place_order` function ask the seller to transfer the NFT to the thunder exchange contract, this way its impossible for seller to accept any bids.

* we believe this can be high severity report instead of medium but we will go with medium severity and let the team/immunefi decide the valid severity for this report.

## Vulnerability Details

when NFT owner want want to list its NFT the function `place_order` should be called with Side == sell:

```sway
    /// Places MakerOrder by calling the strategy contract
    /// Checks if the order is valid
    #[storage(read), payable] // @audit when user set sell order(listing) the function force the user to sent its NFT in checks below
    fn place_order(order_input: MakerOrderInput) {
        _validate_maker_order_input(order_input); // sanity checks 

        let strategy = abi(ExecutionStrategy, order_input.strategy.bits());
        let order = MakerOrder::new(order_input);

        match order.side {
            Side::Buy => { //users make offer for specific nft(bid)
                // Buy MakerOrder (e.g. make offer)
                // Checks if user has enough bid balance
                let pool_balance = _get_pool_balance(order.maker, order.payment_asset); // get the maker balance of the payment asset
                require(order.price <= pool_balance, ThunderExchangeErrors::AmountHigherThanPoolBalance); // example: price is 5 eth and user have 6 eth
            },
            Side::Sell => { 
                // Sell MakerOrder (e.g. listing)
                // Checks if assetId and amount mathces with the order
                //@audit forced to send the NFT
                require(msg_asset_id() == AssetId::new(order.collection, order.token_id), ThunderExchangeErrors::AssetIdNotMatched);
                require(msg_amount() == order_input.amount, ThunderExchangeErrors::AmountNotMatched);
            }, //transfer the NFT to this contract
        }

        strategy.place_order(order); // call fixed strategy 

        log(OrderPlaced {
            order
        });
    }

```

as shown the function place\_order ask for sending the NFT when users want to list their NFT, and after that users can bid on the NFT by calling place\_order with Side == Buy. when the seller decide to accept a bid, the function execute order should be called with Side == sell:

```sway
    /// Executes order by either
    /// filling the sell MakerOrder (e.g. purchasing NFT)
    /// or the buy MakerOrder (e.g. accepting an offer)
    #[storage(read), payable] 
    fn execute_order(order: TakerOrder) {
        _validate_taker_order(order);

        match order.side {
            Side::Buy => _execute_buy_taker_order(order), // buy the NFT directly by buyer
            Side::Sell => _execute_sell_taker_order(order), // accept bid by seller
        }

        log(OrderExecuted {
            order
        });
    }



#[storage(read), payable] //@audit seller is forced to send the NFT which doesn't exist for the user instead its in the thunder exchange itself when place_order called
fn _execute_sell_taker_order(order: TakerOrder) {
    let strategy = abi(ExecutionStrategy, order.strategy.bits());
    let execution_result = strategy.execute_order(order);
    require(execution_result.is_executable, ThunderExchangeErrors::ExecutionInvalid);
    require(
        msg_asset_id() == AssetId::new(execution_result.collection, execution_result.token_id), //@audit
        ThunderExchangeErrors::PaymentAssetMismatched
    ); // but the seller does not have the NFT ?
    require(msg_amount() == execution_result.amount, ThunderExchangeErrors::AmountMismatched);

    // Transfer the NFT
    transfer(
        Identity::Address(order.maker),
        AssetId::new(execution_result.collection, execution_result.token_id),
        execution_result.amount
    );

    // Deduct the fees from bid balance
    _transfer_fees_and_funds_with_pool(
        order.strategy,
        execution_result.collection,
        order.maker,
        order.taker,
        order.price,
        execution_result.payment_asset,
    );
}


```

call to execute\_order will revert, this is because the user is forced to call transfer function in sway which is built-in function with important checks that executes by fuelVM itself, one of these checks is checking if the caller have the determined asset\_id plus the amount. <https://docs.fuel.network/docs/specs/fuel-vm/instruction-set/#tr-transfer-coins-to-contract>

this way its impossible for seller to accept any bids and should wait for someone who buy the NFT from him directly or cancel the order.

## Impact Details

sellers can never accept bid offers because of incorrect logic set in the `_execute_sell_taker_order` function.

## References

its recommended to not ask for seller if they transfered their NFT or not in the \_execute\_sell\_taker\_order function, instead storage map can be added to save/check if the caller indeed transferred the NFT or not.

## Proof of concept

## Proof of Concept

NOTE: check the report id 34567 to get better experience when running this test

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


    let bidder = Address::from(0xf9d8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07);



    
    let maker_order_bid = MakerOrderInput {
     side: Side::Buy,
     maker:bidder,
     collection: ContractId::from(0x7777777777777777777777777777777777777777777777777777777777777777),
     token_id: SubId::from(0x7777777777777777777777777777777777777777777777777777777777777778),
     price: 5u64, // offer 5 tokens (bid)
     amount: 1u64,
     nonce: 1u64,
     strategy:stratgey_fixed,
     payment_asset: asset_to_add,
     expiration_range: 1000u64,
     extra_params: extra_parameter,
    };


    thunder_exch.place_order(maker_order); // this could be called as .call {gas: 5000, asset_id : NFT_id, coins: 1 } but since this not possible in sway test then you need to comment the require check in the place_order--> order.sell
    thunder_exch.place_order(maker_order_bid);

/*
    script;

use interfaces::{
    pool_interface::Pool,
    thunder_exchange_interface::ThunderExchange,
};

use libraries::{
    order_types::*,
};

#[payable]
fn main(
    thunder_exchange: ContractId,
    pool: ContractId,
    order: MakerOrderInput,
    amount: u64, // 1 which is the nft 
    asset: AssetId, // NFT id 
    order: TakerOrder,
) {
    let exchange = abi(ThunderExchange, thunder_exchange.into());
    exchange.execute_order{gas: 5000, asset_id: asset, coins: amount }(order); // order is the maker_order_bid which the seller want to accept but the call revert since seller does not have the NFT anymore
}  
    
    */

}

```
