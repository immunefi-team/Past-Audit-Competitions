# IOP \_ ThunderNFT 34542 - \[Smart Contract - Insight] Not Handling Balance Entries Properly in the Wit

Submitted on Thu Aug 15 2024 11:27:41 GMT-0400 (Atlantic Standard Time) by @bugtester for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34542

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/pool>

Impacts:

* Unbounded gas consumption

## Description

## Brief/Intro

The current implementation of the withdraw function in our smart contract includes a check and update mechanism for user balances. However, the storage entries for zero balances are retained, leading to unnecessary storage usage and potentially high costs, especially with a large number of users having zero balances

using storage.balance\_of.remove(&(sender, asset)), you ensure that entries with a zero balance are not kept in the storage, which optimizes storage usage.

## Impact Details

The primary issue with retaining zero balance entries is the unnecessary usage of storage space, which leads to higher costs and potential inefficiencies. This issue becomes more significant when there is a large number of users with zero balances.

## References

Add any relevant links to documentation or code

## Proof of concept

## Proof of Concept

<https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/pool/src/main.sw#L105C5-L124C6>

```
fn withdraw(asset: AssetId, amount: u64) {
    let sender = msg_sender().unwrap();
    let current_balance = _balance_of(sender, asset);
    require(current_balance >= amount, PoolErrors::AmountHigherThanBalance);


    let asset_manager_addr = storage.asset_manager.read().unwrap().bits();
    let asset_manager = abi(AssetManager, asset_manager_addr);
    require(asset_manager.is_asset_supported(asset), PoolErrors::AssetNotSupported);


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

### fix

updating the withdraw function to remove the storage entry if the new balance after withdrawal is zero. This will optimize storage usage and reduce costs.

\#\[storage(read, write)] fn withdraw(asset: AssetId, amount: u64) { let sender = msg\_sender().unwrap(); let current\_balance = \_balance\_of(sender, asset); require(current\_balance >= amount, PoolErrors::AmountHigherThanBalance);

```
let asset_manager_addr = storage.asset_manager.read().unwrap().bits();
let asset_manager = abi(AssetManager, asset_manager_addr);
require(asset_manager.is_asset_supported(asset), PoolErrors::AssetNotSupported);

let new_balance = current_balance - amount;

if new_balance == 0 {
    storage.balance_of.remove(&(sender, asset)); // Remove the entry if the new balance is zero
} else {
    storage.balance_of.insert((sender, asset), new_balance);
}

transfer(sender, asset, amount);

log(Withdrawal {
    address: sender,
    asset,
    amount,
});
```

}

/// Returns the balance of the user by the assetId #\[storage(read)] fn balance\_of(account: Identity, asset: AssetId) -> u64 { \_balance\_of(account, asset) }

/// Internal function to get the balance of an account for a specific asset #\[storage(read)] fn \_balance\_of(account: Identity, asset: AssetId) -> u64 { let status = storage.balance\_of.get(&(account, asset)).try\_read(); match status { Option::Some(balance) => \*balance, Option::None => 0, } }
