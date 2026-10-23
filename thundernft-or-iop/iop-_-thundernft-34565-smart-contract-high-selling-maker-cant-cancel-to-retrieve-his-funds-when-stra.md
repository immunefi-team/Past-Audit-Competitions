# IOP \_ ThunderNFT 34565 - \[Smart Contract - High] Selling maker cant cancel to retrieve his funds whe

Submitted on Thu Aug 15 2024 22:24:33 GMT-0400 (Atlantic Standard Time) by @NinetyNineCrits for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34565

Report type: Smart Contract

Report severity: High

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/thunder\\_exchange>

Impacts:

* Temporary freezing of NFTs for at least 1 hour

## Description

## Brief/Intro

Similar to submission 34496 ("Users cant withdraw their funds for removed assets"), users can not cancel maker-sell-orders to get their NFTs back, once the strategy, on which an order has been placed, has gotten removed (e.g. in case of version upgrades).

However, unlike submission 34496, there are some important ramifications on handling this issue, as the check serves an important role. These ramifications are discussed below.

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

Those assets remain in the ThunderExchange contract until the order is filled or canceled. The cancellation only works when an order is removed from a strategy that is still whitelisted:

```rs
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
        require(execution_manager.is_strategy_whitelisted(strategy), ThunderExchangeErrors::StrategyNotWhitelisted);
```

## Impact Details

NFTs (or ERC1155) of selling makers can not retrieve their funds, when a strategy for which they placed an order (and sent the funds in) got removed.

## Thoughts on ramifications

Unlike submission 34496 the validation for the strategy can not be simply removed, as it serves an important role here. So thinking about mitigations I would just like to bring up some ideas

1. Allow the owner to do force cancellations (which would be done before adding a new contract)
2. Migrate the existing orders to the storage of the new strategy
3. Add a new field like `was_formerly_whitelisted` in the execution manager and fill it when a strategy gets removed. This field would additionally be checked for cancellations (`is_whitelisted || was_whitelisted`), while executions keep only checking for currently whitelisted strategies.

## References

Not applicable

## Proof of concept

## Proof of Concept

<https://drive.google.com/file/d/1q4pSXHRu7JcNg3Wi1sU188oOzu-P67tn/view?usp=sharing>

The given google drive link contains a fully functional test suite containing all the projects contracts. It was build using the fuel rust SDK, using the official docs as starting point

* <https://docs.fuel.network/docs/sway/testing/testing-with-rust/>
* <https://docs.fuel.network/docs/fuels-rs/getting-started/>

The POC is contained in `tests/harness.rs` and can be run simply with `cargo test "cant_cancel_once_support_for_strategy_ceased"` as long as the [fuel toolchain](https://docs.fuel.network/guides/installation/) is installed.

It does the following:

1. deploy and initialize all the projects contracts, including setting up all the references they need to each other
2. deploy a minimalistic ERC1155 contract that allows arbitrary mints
3. have the maker (seller) mint 1 token for himself
4. have the maker place an order with that 1 token, which will be transferred into the exchange contract
5. have the owner remove the strategy
6. have the maker try to cancel the order to get his assets back, which fails
