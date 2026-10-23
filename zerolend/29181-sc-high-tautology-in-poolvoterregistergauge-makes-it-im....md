# 29181 - \[SC - High] Tautology in PoolVoterregisterGauge makes it im...

Submitted on Mar 9th 2024 at 20:29:08 UTC by @nethoxa for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29181

Report type: Smart Contract

Report severity: High

Target: <https://github.com/zerolend/governance>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

Tautology in `PoolVoter::registerGauge` makes it impossible to add pools to the `_pools` array, as it checks for `isPool[_asset]` to be `true`, which is set in the same logical branch so it can't be triggered. That makes most of the `distribute*` functions useless as they rely on either `_pools.length` (which would be `0` so they become NOPs) or they access elements inside such array, triggering an OOB error and always reverting such a transaction.

## Vulnerability Details

It is pretty visual, the only place where `isPool` is set to `true` for a given asset is in `PoolVoter::registerGauge`:

```solidity
    function registerGauge(
        address _asset,
        address _gauge
    ) external onlyOwner returns (address) {
        if (isPool[_asset]) {
            _pools.push(_asset);
            isPool[_asset] = true;
        }

        ...
    }
```

however, as mappings defaults to `0` or `false`, it won't be triggered and the `_pools` array will remain empty.

## Impact Details

Calls to `distribute` or `distributeEx` will either revert or return without doing anything, as they rely on `_pools.length` to iterate the array or they will access it, even if it is empty. That is, those functions remain useless, so it can be seen a self-griefing from the contract to any user who calls those functions.

## Proof of Concept

I initialized a Foundry repo to test it, but I believe this is simple enough that it does not matter which framework you use to test it. I can provide the repo if you want:

```solidity
pragma solidity 0.8.20;

import {Test} from "forge-std/Test.sol";
import {PoolVoter} from "src/voter/PoolVoter.sol";

contract POC is Test {

    function test_POC() external {
        PoolVoter voter = new PoolVoter();
        voter.init(address(0xabcd), address(0xabcd));
        
        voter.registerGauge(address(0xabcd), address(0xabcd));
        require(voter.pools().length == 0, "POC");
    }
}
```
