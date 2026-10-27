# IOP \_ ThunderNFT 34578 - \[Smart Contract - Insight] unds Not Locked During Order Placement

Submitted on Fri Aug 16 2024 09:32:39 GMT-0400 (Atlantic Standard Time) by @bugtester for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34578

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/thunder\\_exchange>

Impacts:

* Permanent freezing of funds

## Description

## Brief/Intro

in the place\_order function of the ThunderExchange contract where user funds are not properly locked during the order placement process. This allows users to place multiple orders using the same funds

The current implementation of the place\_order function only checks if the user has enough balance to cover the order but does not lock these funds immediately. This oversight allows users to place multiple buy orders simultaneously with the same pool balance, .

## Impact

Place multiple buy orders with insufficient funds. leads funds loss

## Proof of concept

## Proof of Concept

<https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder\\_exchange/src/main.sw#L84C3-L132C6>

User A has a balance of 100 units in payment\_asset. User A places a buy order for 100 units of payment\_asset. Before the system processes and updates the balance, User A quickly places another buy order for 100 units of payment\_asset. Both orders are accepted because the system checks the balance but does not lock the funds during the process. ( or minus balance after place order )

/// Places MakerOrder by calling the strategy contract /// Checks if the order is valid #\[storage(read), payable] fn place\_order(order\_input: MakerOrderInput) { \_validate\_maker\_order\_input(order\_input);

```
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
        require(msg_amount() == order_input.amount, ThunderExchangeErrors::AmountNotMatched);
    },
}

strategy.place_order(order);

log(OrderPlaced {
    order
});
```

}

### FIX

implement a mechanism to lock user funds during the order placement process. This can be achieved by:

Locking the required funds immediately after validating the order and before any further processing.

\#\[storage(read, write), payable] fn place\_order(order\_input: MakerOrderInput) { \_validate\_maker\_order\_input(order\_input);

```
let strategy = abi(ExecutionStrategy, order_input.strategy.bits());
let order = MakerOrder::new(order_input);
match order.side {
    Side::Buy => {
        // Lock user funds before placing the order
        let pool_balance = _get_pool_balance(order.maker, order.payment_asset);
        require(order.price <= pool_balance, ThunderExchangeErrors::AmountHigherThanPoolBalance);

        // Lock the funds
        _lock_funds(order.maker, order.payment_asset, order.price);
    },
    Side::Sell => {
        require(msg_asset_id() == AssetId::new(order.collection, order.token_id), ThunderExchangeErrors::AssetIdNotMatched);
        require(msg_amount() == order_input.amount, ThunderExchangeErrors::AmountNotMatched);
    },
}

strategy.place_order(order);

log(OrderPlaced {
    order
});
```

}

fn \_lock\_funds(account: Identity, asset: AssetId, amount: u64) { // Logic to lock the specified amount of funds for the account // This can involve updating a separate storage map to track locked funds }
