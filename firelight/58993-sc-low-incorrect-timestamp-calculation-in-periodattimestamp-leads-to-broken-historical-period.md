# 58993 sc low incorrect timestamp calculation in periodattimestamp leads to broken historical period lookups

**Submitted on Nov 7th 2025 at 15:18:35 UTC by @dobrevaleri for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #58993
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief

The `FirelightVault::periodAtTimestamp()` function incorrectly uses the current block timestamp instead of the provided `timestamp` parameter when calculating the period number. This causes the function to always return the current period regardless of which historical timestamp is queried, breaking any functionality that relies on historical period-to-timestamp mappings.

## Vulnerability Details

The `FirelightVault::periodAtTimestamp()` function is designed to return the period number corresponding to any given timestamp. The function correctly identifies which period configuration applies to the given timestamp via `periodConfigurationAtTimestamp()`, but then fails to use the `timestamp` parameter in the actual period calculation.

```solidity
function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
    PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
    return periodConfiguration.startingPeriod + _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration;
}
```

The issue stems from the `_sinceEpoch()` helper function:

```solidity
function _sinceEpoch(uint48 epoch) private view returns (uint48) {
    return Time.timestamp() - epoch;
}
```

The helper function always calculates the time difference from `epoch` to `Time.timestamp()` (the current block timestamp), completely ignoring the `timestamp` parameter passed to `periodAtTimestamp()`. This means that instead of calculating how many period durations have elapsed from the epoch to the **given timestamp**, it calculates how many have elapsed to the **current time**.

For example, with a daily period duration:

* Contract deployed at timestamp T₀ (period 0)
* Query `periodAtTimestamp(T₀)` at timestamp T₃ (3 days later, period 3)
* Expected result: 0
* Actual result: 3

The calculation becomes:

```
startingPeriod + (Time.timestamp() - epoch) / duration
= 0 + (T₃ - T₀) / 86400
= 3  // Wrong! Should be (T₀ - T₀) / 86400 = 0
```

The `periodConfigurationAtTimestamp()` function correctly finds the period configuration for the given timestamp by iterating through configurations and comparing against the timestamp parameter. However, this correct behavior is undermined when `_sinceEpoch()` then uses the current time instead.

## Impact Details

Historical period queries always return the current period number with the period configuration at the `timestamp` instead of the correct historical period.

## References

<https://github.com/firelight-protocol/firelight-core/blob/db36312f1fb24efc88c3fde15a760defbc3e6370/contracts/FirelightVault.sol#L249>

<https://github.com/firelight-protocol/firelight-core/blob/db36312f1fb24efc88c3fde15a760defbc3e6370/contracts/FirelightVault.sol#L796>

## Proof of Concept

## Proof of Concept

```javascript
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures.js')
const { expect } = require('chai')
const { ethers } = require('hardhat')

describe('PoC: periodAtTimestamp Bug', function () {
    const DECIMALS = 6
    const INITIAL_DEPOSIT_LIMIT = ethers.parseUnits('20000', DECIMALS)
    const PERIOD_CONFIGURATION_DURATION = 86400 // 1 day

    let token_contract, firelight_vault, users, utils
    let deploymentTime

    before(async () => {
        ({ token_contract, firelight_vault, users, utils } = await loadFixture(
            deployVault.bind(null, {
                decimals: DECIMALS,
                initial_deposit_limit: INITIAL_DEPOSIT_LIMIT,
                period_configuration_duration: PERIOD_CONFIGURATION_DURATION
            })
        ))

        deploymentTime = await time.latest()
    })

    it('demonstrates periodAtTimestamp always returns current period instead of period at given timestamp', async () => {
        const initialTime = await time.latest()
        const period0Timestamp = initialTime

        // Advance time to period 1 (1 day later)
        await time.increase(PERIOD_CONFIGURATION_DURATION)
        const period1Time = await time.latest()
        const period1Timestamp = period1Time

        // Advance time to period 2 (2 days from start)
        await time.increase(PERIOD_CONFIGURATION_DURATION)
        const period2Time = await time.latest()

        // Advance time to period 3 (3 days from start)
        await time.increase(PERIOD_CONFIGURATION_DURATION)
        const period3 = await firelight_vault.currentPeriod()

        // BUG: All these calls should return different periods, but they all return the CURRENT period
        const periodAtTime0 = await firelight_vault.periodAtTimestamp(period0Timestamp)
        const periodAtTime1 = await firelight_vault.periodAtTimestamp(period1Timestamp)
        const periodAtTime2 = await firelight_vault.periodAtTimestamp(period2Time)

        // Assertions to prove the bug
        expect(periodAtTime0).to.equal(period3, 'BUG CONFIRMED: periodAtTimestamp(period0Time) returns current period')
        expect(periodAtTime1).to.equal(period3, 'BUG CONFIRMED: periodAtTimestamp(period1Time) returns current period')
        expect(periodAtTime2).to.equal(period3, 'BUG CONFIRMED: periodAtTimestamp(period2Time) returns current period')

        // These assertions show what SHOULD happen (but currently fail)
        try {
            expect(periodAtTime0).to.equal(0)
        } catch (e) {
            // Expected to fail due to bug
        }

        try {
            expect(periodAtTime1).to.equal(1)
        } catch (e) {
            // Expected to fail due to bug
        }

        try {
            expect(periodAtTime2).to.equal(2)
        } catch (e) {
            // Expected to fail due to bug
        }
    })

    it('demonstrates impact: historical period queries are broken', async () => {
        const currentTime = await time.latest()
        const currentPeriod = await firelight_vault.currentPeriod()

        // Try to query a time 2 days in the past (should be period currentPeriod-2)
        const pastTime = currentTime - (2 * PERIOD_CONFIGURATION_DURATION)
        const periodAtPast = await firelight_vault.periodAtTimestamp(pastTime)

        expect(periodAtPast).to.equal(currentPeriod, 'BUG CONFIRMED: Returns current period for historical timestamp')
    })
})

```
