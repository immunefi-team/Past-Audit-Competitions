# 59115 sc low periodattimestamp function is incorrectly implemented and always returns period at current timestamp&#x20;

**Submitted on Nov 8th 2025 at 21:28:28 UTC by @Tadev for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59115
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

As per the documentation, the `periodAtTimestamp` function is supposed to return the period number for the timestamp given. This function is defined as follows:

```
    function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
        PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
        // solhint-disable-next-line max-line-length
        return periodConfiguration.startingPeriod + _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration;
    }
```

The problem is that the current implementation is incorrect, always returning the period at the current timestamp.

## Vulnerability Details

The `periodAtTimestamp` function fetches the `periodConfiguration` of the provided timestamp. After that, it computes:

```
periodConfiguration.startingPeriod + _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration
```

The problem lies in `_sinceEpoch` internal function, which doesn't use the provided timestamp but `block.timestamp` instead:

```
    function _sinceEpoch(uint48 epoch) private view returns (uint48) {
        return Time.timestamp() - epoch;
    }
```

This means the `periodAtTimestamp` function will return current period instead of period at given timestamp.

## Impact Details

The impact of this issue is low as it consists of a logical error in `periodAtTimestamp` function. Anyone querying this function to know the period of a given timestamp will get a wrong result.

The impact is limited in the protocol itself because `periodAtTimestamp` function is used internally only in `currentPeriod` function, and the timestamp provided is the current block timestamp.

## Proof of Concept

## Proof of Concept

Please create a *poc.js* file in the *test* folder and copy paste the following code:

```
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures.js')
const { expect } = require('chai')
const { ethers } = require('hardhat')

describe('POC LOW', function() {
  const DECIMALS = 18,
        INITIAL_DEPOSIT_LIMIT = ethers.parseUnits('100000', DECIMALS)

  before(async () => {
    ({ firelight_vault, config } = await loadFixture(
      deployVault.bind(null, { decimals: DECIMALS, initial_deposit_limit: INITIAL_DEPOSIT_LIMIT })
    ))
  })

  it('demonstrates periodAtTimestamp ignores the timestamp parameter', async () => {
    const currentTime = await time.latest()
    const periodDuration = config.period_configuration_duration
    
    console.log(`\n  Current block timestamp: ${currentTime}`)
    console.log(`  Period duration: ${periodDuration} seconds`)
    
    // Get current period
    const currentPeriod = await firelight_vault.currentPeriod()
    console.log(`  Current period: ${currentPeriod}`)
    
    // Try to query period at current timestamp
    const periodAtNow = await firelight_vault.periodAtTimestamp(currentTime)
    console.log(`\n  periodAtTimestamp(${currentTime}) = ${periodAtNow}`)
    console.log(`  ✓ Correct: matches current period`)
    
    // Try to query period 3 periods in the future
    const futureTimestamp = currentTime + (periodDuration * 3)
    const periodAtFuture = await firelight_vault.periodAtTimestamp(futureTimestamp)
    const expectedPeriod = currentPeriod + 3n
    
    console.log(`\n  Query period at future timestamp:`)
    console.log(`  Future timestamp: ${futureTimestamp} (3 periods ahead)`)
    console.log(`  Expected period: ${expectedPeriod}`)
    console.log(`  periodAtTimestamp(${futureTimestamp}) = ${periodAtFuture}`)
    console.log(`  ✗ BUG: Returns ${periodAtFuture} instead of ${expectedPeriod}`)
  })
})
```

The output of this test is:

```
POC LOW

  Current block timestamp: 1766690171
  Period duration: 172800 seconds
  Current period: 0

  periodAtTimestamp(1766690171) = 0
  ✓ Correct: matches current period

  Query period at future timestamp:
  Future timestamp: 1767208571 (3 periods ahead)
  Expected period: 3
  periodAtTimestamp(1767208571) = 0
  ✗ BUG: Returns 0 instead of 3
```

This tests highlights that when querying the period at a given timestamp in the future, the function doesn't return the correct result. This issue is the same when querying with a timestamp that is in the past.
