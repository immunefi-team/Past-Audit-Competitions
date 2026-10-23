# IOP \_ ThunderNFT 34587 - \[Smart Contract - High] Users might temporarily get their funds locked in P

Submitted on Fri Aug 16 2024 16:44:40 GMT-0400 (Atlantic Standard Time) by @jecikpo for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34587

Report type: Smart Contract

Report severity: High

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/pool>

Impacts:

* Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

The `Pool` contract allows deposits and withdrawals of assets. Assets must be whitelisted within `AssetManager` contract. The `withdraw()` method is verifying if an asset is whitelisted, hence once it gets removed from the `AssetManager` users cannot withdraw them.

## Vulnerability Details

The `withdraw()` method is unnecessarily checking whether an asset is whitelisted with `AssetManager` contract by calling `is_asset_supported()` as per the code snippet below:

```
fn withdraw(asset: AssetId, amount: u64) {
        [...]
        require(asset_manager.is_asset_supported(asset), PoolErrors::AssetNotSupported);
        [...]
    }
```

Hence if a protocol decided to delist an asset which is already deposited by a user, the user cannot withdraw their funds. In order to withdraw the funds the contract owner would need to again whitelist an asset by calling `add_asset()` in the `AssetManager` contract.

## Impact Details

User funds get stuck when the situation above happens in the contract and the only way to get them out is that the contract owner reacts and puts the asset back on the whitelist. Depending on the protocol teams reaction this can easily go above 1 hour hence the impact is High.

While it is possible to recover from the situation by protocol team's reaction, the "emergency" whitelisting previously de-listed asset could cause some other users to deposit the unwanted asset, and hence the problem might continue.

## Solution proposal

Allow withdrawal of assets regardless if they are whitelisted or not.

## References

The line causing the issue: <https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/pool/src/main.sw#L112>

## Proof of concept

## Proof of Concept

The PoC is in a secret Gist: <https://gist.github.com/jecikpo/2f31696bfcb9d59c4c1129b72efd2537>
