# IOP \_ ThunderNFT 34522 - \[Smart Contract - Low] Self-transfer would inflate the balance

Submitted on Wed Aug 14 2024 13:27:48 GMT-0400 (Atlantic Standard Time) by @NinetyNineCrits for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34522

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/pool>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The pools `_transfer` function does not handle the transfer case correctly, when `from` and `to` are the same address. If thats the case, the addresses balance will be doubled. Currently this can not be exploited, as there is no path where both inputs point to the same address.

## Vulnerability Details

The `_transfer` function caches both the `from` and `to` values before modifying them in storage:

```rust
    let from_balance = _balance_of(from, asset);
    let to_balance = _balance_of(to, asset);
    require(from_balance >= amount, PoolErrors::AmountHigherThanBalance);

    storage.balance_of.insert((from, asset), from_balance - amount);
    storage.balance_of.insert((to, asset), to_balance + amount);
```

If they are the same, the `from` update wont affect the `to_balance` and the `to` update will work on an outdated value, causing an increase of overall balance.

## Impact Details

The function as a unit works incorrectly for a certain edge case, hence "fails to deliver promise". While this can not be exploited currently, it poses a substantial risk on further changes, due to balance inflation.

## References

Not Applicable

## Proof of concept

## Proof of Concept

<https://gist.github.com/99crits/c6bbf48b8801aefe9ab6db6000162e47>

The given gist uses a sway unit test to demonstrate the issue and can be run with `forc test`. This POC contains a simplified version of the pool only containing the necessary parts to showcase the bug.

The initialize function gives the sender an initial balance of 1000. After performing a self-transfer, the senders balance is 2000.
