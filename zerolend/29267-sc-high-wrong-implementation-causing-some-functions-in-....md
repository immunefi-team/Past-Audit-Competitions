# 29267 - \[SC - High] Wrong implementation causing some functions in ...

Submitted on Mar 12th 2024 at 18:54:46 UTC by @EricTee for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29267

Report type: Smart Contract

Report severity: High

Target: <https://github.com/zerolend/governance>

Impacts:

* Unusable functions in governance/contracts/voter/PoolVoter.sol

## Description

## Brief/Intro

Some functions in `governance/contracts/voter/PoolVoter.sol` are unusable. Specifically, there is no setter for `isPool` variable to set certain `address asset` to `true` or `false`. As a result, the length of `_pools` will always be 0. `PoolVoter::distribute` and `PoolVoter::distributeEx` functions which rely on length of pool will become unusable.

## Vulnerability Details

In `PoolVoter::registerGauge`:

```
  function registerGauge(
        address _asset,
        address _gauge
    ) external onlyOwner returns (address) {
        if (isPool[_asset]) {
            _pools.push(_asset);
            isPool[_asset] = true;
        }

        bribes[_gauge] = address(0);
        gauges[_asset] = _gauge;
        poolForGauge[_gauge] = _asset;
        _updateFor(_gauge);

        return _gauge;
    }
```

The contract check for `if (isPool[_asset])`. However, there is no setter to let owner set certain `_asset` to `true` in `isPool` mapping variable. As a result, the length of pool will always be 0 and `PoolVoter::distribute` and `PoolVoter::distributeEx` functions which rely on length of pool will become unusable.

## Impact Details

The impact of this issue is that `PoolVoter::distribute` and `PoolVoter::distributeEx` functions which rely on length of pool are unusable.

Consider making the following changes in `PoolVoter::registerGauge`:

```diff
    function registerGauge(
        address _asset,
        address _gauge
    ) external onlyOwner returns (address) {
--        if (isPool[_asset]) {
++        if (!isPool[_asset]) {
            _pools.push(_asset);
            isPool[_asset] = true;
        }

        bribes[_gauge] = address(0);
        gauges[_asset] = _gauge;
        poolForGauge[_gauge] = _asset;
        _updateFor(_gauge);

        return _gauge;
    }
```

## References

<https://github.com/zerolend/governance/blob/main/contracts/voter/PoolVoter.sol#L136>

## Proof of Concept

Manual Analysis
