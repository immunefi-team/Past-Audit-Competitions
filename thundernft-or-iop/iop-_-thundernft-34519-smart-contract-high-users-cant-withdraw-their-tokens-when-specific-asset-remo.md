# IOP \_ ThunderNFT 34519 - \[Smart Contract - High] users cant withdraw their tokens when specific asse

Submitted on Wed Aug 14 2024 12:53:37 GMT-0400 (Atlantic Standard Time) by @zeroK for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34519

Report type: Smart Contract

Report severity: High

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/pool>

Impacts:

* Permanent freezing of funds

## Description

## Brief/Intro

the pool.sw contract allow users to deposit and withdraw whitelisted tokens into the pool contract, this is possible by calling the deposit function and withdraw function, however there is an issue when users want to withdraw their tokens, as the withdraw token check if the token is whitelisted or not, and this can cause issue and lock user funds in case if the `assetManger` contract decide to remove specific token from whitelist, this way users token will be stuck forever.

## Vulnerability Details

the deposit function allow users to deposit specific amount for allowed tokens only as shown below:

```sway

    /// Deposits the supported asset into this contract
    /// and assign the deposited amount to the depositer as bid balance
    #[storage(read, write), payable]
    fn deposit() {
        let asset_manager_addr = storage.asset_manager.read().unwrap().bits();
        let asset_manager = abi(AssetManager, asset_manager_addr);
        require(asset_manager.is_asset_supported(msg_asset_id()), PoolErrors::AssetNotSupported); //@audit only supported

        let address = msg_sender().unwrap();
        let amount = msg_amount();
        let asset = msg_asset_id();

        let current_balance = _balance_of(address, asset);
        let new_balance = current_balance + amount;
        storage.balance_of.insert((address, asset), new_balance);

        log(Deposit {
            address,
            asset,
            amount
        });
    }

```

however same check exist in the withdraw function too:

```sway
    /// Withdraws the amount of assetId from the contract
    /// and sends to sender if sender has enough balance
    #[storage(read, write)]
    fn withdraw(asset: AssetId, amount: u64) {
        let sender = msg_sender().unwrap();
        let current_balance = _balance_of(sender, asset);
        require(current_balance >= amount, PoolErrors::AmountHigherThanBalance);

        let asset_manager_addr = storage.asset_manager.read().unwrap().bits();
        let asset_manager = abi(AssetManager, asset_manager_addr);
        require(asset_manager.is_asset_supported(asset), PoolErrors::AssetNotSupported); //@audit prevent if asset removed

        let new_balance = current_balance - amount;
        storage.balance_of.insert((sender, asset), new_balance);

        transfer(sender, asset, amount);

        log(Withdrawal {
            address: sender,
            asset,
            amount,
        });
    }


```

this can lead to stuck of funds since asset can be added and removed from whitelist using the add/remove asset function below:

```sway
    /// Adds asset into supported assets vec
    #[storage(read, write)]
    fn add_asset(asset: AssetId) {
        storage.owner.only_owner();
        require(
            !_is_asset_supported(asset),
            AssetManagerErrors::AssetAlreadySupported
        );

        storage.is_supported.insert(asset, true);
        storage.assets.push(asset);
    }

    /// Removes asset from supported assets vec
    #[storage(read, write)]
    fn remove_asset(index: u64) {
        storage.owner.only_owner();
        let asset = storage.assets.remove(index);
        storage.is_supported.insert(asset, false);
    }
```

if owner called the remove function while the tokens exist in the pool then all users with this token can not withdraw back their tokens amount.

we set the severity to high for reason below:

* the owner can add back the token again by calling add\_asset function but its not recommended behavior.

## Impact Details

users amount can be temporary freez in pool contract.

## References

there is no need to check for token whitelisted tokens when user withdraw since deposit allow to deposit whitelist tokens only

## Proof of concept

## Proof of Concept

the POC show that the token can be added and removed any time:

```sway
//run this test below asse_manger/src/main.sw

#[test]

fn test_attack_asset() {
    let assetManger = abi(AssetManager, CONTRACT_ID);
    assetManger.initialize();
    let asset_id = AssetId::default();
    assetManger.add_asset(asset_id);

    //users deposit between this gap

    assetManger.remove_asset(0);

}
```
