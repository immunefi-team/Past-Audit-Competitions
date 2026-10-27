# IOP \_ ThunderNFT 34906 - \[Smart Contract - Low] Existing Sell order can be executed despite payment

Submitted on Sat Aug 31 2024 15:42:28 GMT-0400 (Atlantic Standard Time) by @jecikpo for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34906

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/thunder\\_exchange>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

If a *Sell* order is placed within Thunder exchange using certain payment asset, it can still be executed, even if the payment asset was remove (delisted) from the `AssetManager` contract.

## Vulnerability Details

User's can place (or update) *Sell* order and indicate certain `payment_asset` within the placed order. This `payment_asset` must be whitelisted by the protocol owner within the `AssetManager` contract. This verification happens inside the internal function `_validate_maker_order_input()` in `ThunderExchange` contract.

When a buyer want's to execute a given *Sell* order and hence exchange the `payment_asset` (indicated in the *Sell* order placed) for the NFT/ERC1155 he calls the `execute_order()` at `ThunderExchange`. Here however the `payment_asset` is not validated within the internal `_validate_taker_order()` function, hence such a sale can proceed.

## Impact Details

User's in certain condition (owners of already placed orders) can violate the protocol rule of not selling their NFTs/ERC1155s using delisted asset. The contract doesn't lose value here, but users may abuse the rule set by the protocol owners, hence the severity is Low.

## Solution Proposal

The asset validation would need to be added to the `_validate_taker_order()` internal function. This would have certain extra gas cost on the `execute_order()` method.

## References

The problematic validation: <https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder\\_exchange/src/main.sw#L349>

## Proof of concept

## Proof of Concept

PoC can be found here: <https://gist.github.com/jecikpo/ad4bba11cc2ba9de9202f83ad7225511>
