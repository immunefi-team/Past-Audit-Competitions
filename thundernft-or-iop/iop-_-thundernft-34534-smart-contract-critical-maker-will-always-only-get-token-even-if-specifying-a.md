# IOP \_ ThunderNFT 34534 - \[Smart Contract - Critical] Maker will always only get token even if specif

Submitted on Wed Aug 14 2024 22:15:43 GMT-0400 (Atlantic Standard Time) by @NinetyNineCrits for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34534

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/libraries>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The project allows makers to specify an amount of an assetId they are requesting. However, they will only ever get 1 unit of that assetId when an order is filled, but will pay the whole price of the order.

## Vulnerability Details

A maker can specify the amount of tokens he would like to get:

```rust
pub struct MakerOrder {
    ...
    pub amount: u64,
```

When a taker fills a makers order, the function `_execute_sell_taker_order` verifies, that the taker has send a certain amount of tokens as part of the call:

```rust
fn _execute_sell_taker_order(order: TakerOrder) {
    ...
    require(msg_amount() == execution_result.amount, ThunderExchangeErrors::AmountMismatched);
```

However, it turns out that `execution_result.amount` is currently hardcoded as 1:

```rust
pub fn s1(maker_order: MakerOrder, taker_order: TakerOrder) -> ExecutionResult {
    ExecutionResult {
        ...
        amount: 1,
        ...
    }
```

## Impact Details

In case of ERC1155 tokens, the maker will always only get 1 unit of a certain tokenId, despite having asked for more, which is causing them an unexpected loss.

## References

not applicable

## Proof of concept

## Proof of Concept

<https://drive.google.com/file/d/1nA6ZQQa\\_yGosqMaR\\_Ug6OL-MQdrmuaef/view?usp=drive\\_link>

The given google drive link contains a fully functional test suite containing all the projects contracts. It was build using the fuel rust SDK, using the official docs as starting point

* <https://docs.fuel.network/docs/sway/testing/testing-with-rust/>
* <https://docs.fuel.network/docs/fuels-rs/getting-started/>

The POC is contained in `tests/harness.rs` and can be run simply with `cargo test` as long as the [fuel toolchain](https://docs.fuel.network/guides/installation/) is installed.

It does the following:

1. deploy and initialize all the projects contracts, including setting up all the references they need to each other
2. deploy a minimalistic ERC1155 contract that allows arbitrary mints
3. have the taker (seller) mint 10 tokens for himself
4. have the maker (buyer) place an order for 10 units of same AssetId that the taker minted in step 3
5. have the taker fill the order from step 4, but only transfer in 1 unit (in fact, the call would fail if he sent more)
6. assert that the maker only got 1 unit of the ERC1155 instead of 10
