# IOP \_ ThunderNFT 34560 - \[Smart Contract - Critical] Updating sell-maker-orders does not provide ref

Submitted on Thu Aug 15 2024 18:38:05 GMT-0400 (Atlantic Standard Time) by @NinetyNineCrits for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34560

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/thunder\\_exchange>

Impacts:

* Permanent freezing of funds

## Description

## Brief/Intro

A selling maker gets no refund on any ERC1155-like tokens, when he places an order with amount > 1 and then updates the order with a lower amount. Those tokens will be stuck in the contract.

Likewise a selling maker can increase the amount when updating an order but its not checked whether they have sent in the difference. This allows anyone to take the stuck tokens from the above case.

## Vulnerability Details

When a selling maker places an order, it is checked in `place_order` that they have sent the right amount of the right asset:

```rs
fn place_order(order_input: MakerOrderInput) {
    ...
    match order.side {
        ...
        Side::Sell => {
            // Sell MakerOrder (e.g. listing)
            // Checks if assetId and amount mathces with the order
            require(msg_asset_id() == AssetId::new(order.collection, order.token_id), ThunderExchangeErrors::AssetIdNotMatched);
            require(msg_amount() == order_input.amount, ThunderExchangeErrors::AmountNotMatched);
        },
    }
```

The `update_order` function doesnt do any specific checks for updates of sell-maker-orders:

```rs
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
        Side::Sell => {},
    }

    strategy.update_order(order);
```

That means any changes in the `amount` field will cause an inconsistency in the amount thats in the contract.

## Impact Details

Selling makers can have their tokens stuck in the contract, which can then be taken by other selling makers

## References

Not Applicable

## Proof of concept

## Proof of Concept

<https://drive.google.com/file/d/11T\\_ut1jiEdtYB7zwBbXjpPeKGpLISpM6/view?usp=sharing>

The given google drive link contains a fully functional test suite containing all the projects contracts. It was build using the fuel rust SDK, using the official docs as starting point

* <https://docs.fuel.network/docs/sway/testing/testing-with-rust/>
* <https://docs.fuel.network/docs/fuels-rs/getting-started/>

The POC is contained in `tests/harness.rs` and can be run simply with `cargo test "selling_maker_gets_no_refund_on_order_update"` as long as the [fuel toolchain](https://docs.fuel.network/guides/installation/) is installed.

It does the following:

1. deploy and initialize all the projects contracts, including setting up all the references they need to each other
2. deploy a minimalistic ERC1155 contract that allows arbitrary mints
3. have the maker (seller) mint 10 tokens for himself
4. have the maker place an order with those 10 tokens, which will be transferred into the exchange contract
5. have the maker update his order with lowering the amount of tokens to 1
6. have another user take the order
7. assert that maker will not have any tokens, taker will have 1 and the contract holds the remaining 9.
