# 30584 - \[SC - Insight] Invalid check to make sure Minter is already in...

Submitted on May 1st 2024 at 13:08:37 UTC by @kankodu for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30584

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Minter.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

* A meaningless check

## Vulnerability Details

* In `Minter.initialize` function, there is this check `require(msg.sender != address(0), "already initialized");` that is supposed to make sure it throws an error Minter is already initialized. This check is meaningless.
* msg.sender won't ever be equal to address(0). There is no circumstance where this error will be thrown.
* If the error is moved before `require(initializer == msg.sender, "not initializer");` and updated with `require(initializer != address(0), "already initialized");` in that case it makes sense. It will now throw an error that Minter is already initialized if someone (including the previous initializer) tries to initialise it again.

```solidity
  function initialize() external {
        require(initializer != address(0), "already initialized");
        require(initializer == msg.sender, "not initializer");
        initializer = address(0);
    }
```

## Impact Details

* Contract fails to deliver promised returns, but doesn't lose value
* A useful error won't ever be thrown if it is left as it is.

## References

* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Minter.sol?utm\\_source=immunefi#L98>

## Proof of Concept

* Take a look at [this](https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/test/Minter.t.sol#L343) test.
  * It has to impersonate the address(0) for the error to be thrown. In real environment this won't be possible.
