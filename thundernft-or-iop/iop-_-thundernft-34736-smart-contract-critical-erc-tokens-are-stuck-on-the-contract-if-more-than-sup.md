# IOP \_ ThunderNFT 34736 - \[Smart Contract - Critical] ERC tokens are stuck on the contract if more th

Submitted on Thu Aug 22 2024 21:39:01 GMT-0400 (Atlantic Standard Time) by @jecikpo for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34736

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/libraries>

Impacts:

* Permanent freezing of NFTs

## Description

## Brief/Intro

When a *Sell* order is executed the amount of assets sold to the taker is always 1. Once an order is executed it is deleted. If there were more than 1 asset amount on that order they are locked in the exchange permanently

## Vulnerability Details

The Thunder exchange supports also ERC1155 tokens which could be provided during placement of the *Sell* order by calling the `place_order()` method.

The contract validate if the order input data matches with the assets supplied to the contract by the seller using the following code:

```
require(msg_asset_id() == AssetId::new(order.collection, order.token_id), ThunderExchangeErrors::AssetIdNotMatched);
require(msg_amount() == order_input.amount, ThunderExchangeErrors::AmountNotMatched);
```

When a *Sell* order is executed by calling the `execute_order()` method the `amount` to be sold is hardcoded to `1` in the `ExecutionResult` function `s1()` as per the following snippet:

```
    pub fn s1(maker_order: MakerOrder, taker_order: TakerOrder) -> ExecutionResult {
        ExecutionResult {
            is_executable: (
                [ . . . ]
            ),
            collection: taker_order.collection,
            token_id: taker_order.token_id,
            amount: 1,
            payment_asset: maker_order.payment_asset,
        }
    }
```

This means that only single asset is sold. Later on during the order processing, the order gets deleted in the `_execute_order()` function. Once an order is deleted:

1. it cannot be further executed, hence no more of the remaining assets can be sold.
2. it cannot be cancelled by the maker, hence the assets cannot be withdrawn back to the original owner.

## Impact Details

As the ERC1155 should be supported, the impact of this bug is Critical because it leads to the seller's assets being permanently stuck on the contract in case they want to sell more than one (which is usually the case with ERC1155).

## Solution Proposal

The `amount` to be sold needs to be dynamic and taken from the `TakerOrder` input data (which needs to be modified). Also the amount inside the order must be adjusted. Solution to this issue needs to be added into multiple contracts, hence needs to be carefully though of.

## References

The problematic line in the libraries: <https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/libraries/src/execution\\_result.sw#L31>

## Proof of concept

## Proof of Concept

Can be found here: <https://gist.github.com/c8a90b54808923309280d9624ae01401.git>
