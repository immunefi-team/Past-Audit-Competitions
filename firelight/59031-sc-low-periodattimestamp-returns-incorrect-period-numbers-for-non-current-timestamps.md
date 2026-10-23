# #59031 \[SC-Low] periodattimestamp returns incorrect period numbers for non current timestamps

**Submitted on Nov 7th 2025 at 21:37:11 UTC by @Falendar for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59031
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `periodAtTimestamp` function doesn't return the correct period for the provided `timestamp`.

## Vulnerability Details

The vulnerability exists in the interaction between two functions:

```solidity
function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
    PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
    return periodConfiguration.startingPeriod + _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration;
}
```

```solidity
function _sinceEpoch(uint48 epoch) private view returns (uint48) {
    return Time.timestamp() - epoch;
}
```

The bug occurs because:

1. `periodAtTimestamp(timestamp)` accepts a `timestamp` parameter indicating which point in time to query
2. It correctly uses this parameter to fetch the appropriate `periodConfiguration` via `periodConfigurationAtTimestamp(timestamp)`
3. However, when calculating the period number, it calls `_sinceEpoch(periodConfiguration.epoch)` which ignores the input parameter and uses `Time.timestamp()` (current block time) instead
4. This means the elapsed time calculation uses the wrong reference point: current time instead of the queried timestamp

## Impact Details

Any external protocol, contract, user or future implementations that calls `periodAtTimestamp` for historical analysis will receive incorrect data.

## References

<https://github.com/firelight-protocol/firelight-core/blob/db36312f1fb24efc88c3fde15a760defbc3e6370/contracts/FirelightVault.sol#L246-L250>

## Proof of Concept

## Proof of Concept

### Example

**periodConfigurations\[0] = {epoch: 1000, duration: 100, startingPeriod: 0}**

Period timeline:

Period 0: 1000-1100

Period 1: 1100-1200

**Period 2: 1200-1300** (this is the period that needs to be retrieved)

...

Period 10: 2000-2100

**Current blockchain timestamp: Time.timestamp() = 2000**

**Retrieving period at timestamp 1250: vault.periodAtTimestamp(1250);**

elapsed = Time.timestamp() - 1000 = 2000 - 1000 = 1000

periodsElapsed = 1000 / 100 = 10

periodNumber = 0 + 10 = 10 (current value, WRONG)

**Result: Returns Period 10 instead of Period 2 (always returns current period)**
