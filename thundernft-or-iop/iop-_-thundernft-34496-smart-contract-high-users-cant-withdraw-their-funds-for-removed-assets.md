# IOP \_ ThunderNFT 34496 - \[Smart Contract - High] Users cant withdraw their funds for removed assets

Submitted on Tue Aug 13 2024 23:14:04 GMT-0400 (Atlantic Standard Time) by @NinetyNineCrits for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34496

Report type: Smart Contract

Report severity: High

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/pool>

Impacts:

* Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

Users can not withdraw their funds when support for an asset is ceased. Only deposits should be restricted in that case and withdrawals of previously deposited assets still be possible (as they still belong to users), even if the project decides not to support said asset anymore.

## Vulnerability Details

The pools `withdraw` function requires an asset to still be whitelisted in the AssetManager:

```rust
#[storage(read, write)]
fn withdraw(asset: AssetId, amount: u64) {
    let sender = msg_sender().unwrap();
    let current_balance = _balance_of(sender, asset);
    require(current_balance >= amount, PoolErrors::AmountHigherThanBalance);
    
    let asset_manager_addr = storage.asset_manager.read().unwrap().bits();
    let asset_manager = abi(AssetManager, asset_manager_addr);
    require(asset_manager.is_asset_supported(asset), PoolErrors::AssetNotSupported);
```

## Impact Details

Users can not withdraw their funds for assets that are not supported anymore. Its possible for the owner to reenable it to allow withdrawals again, hence only a temporary freeze. However, deposits for said asset would be then possible again too, possibly leading to the asset never being fully emptied from the pool.

If stopping withdrawals is an intended use case (e.g. for emergencies), it should be done via a separate pause mechanism.

## References

Not applicable

## Proof of concept

## Proof of Concept

<https://drive.google.com/file/d/1KyirxwbeR2ObOCben\\_xusFhuBHL4\\_1a9/view?usp=sharing>

The given google drive link contains a minimum viable test suite that was build using the fuel rust SDK, using the official docs as starting point

* <https://docs.fuel.network/docs/sway/testing/testing-with-rust/>
* <https://docs.fuel.network/docs/fuels-rs/getting-started/>

The POC is contained in `tests/harness.rs` and can be run simply with `cargo test` as long as the [fuel toolchain](https://docs.fuel.network/guides/installation/) is installed.

It does the following:

1. deploy and initialize both AssetManager and Pool
2. demonstrate that normal user deposit and withdraw works
3. demonstrate that withdraw fails to work after support for the asset has been removed
