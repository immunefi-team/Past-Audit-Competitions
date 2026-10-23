# 59298 sc low function periodattimestamp ignores the input timestamp returning the current period instead

**Submitted on Nov 10th 2025 at 22:11:16 UTC by @Paludo0x for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59298
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The function `periodAtTimestamp(uint48 timestamp)` is intended to return the period index corresponding to the provided timestamp `timestamp`.

However, the implementation computes the elapsed time using the **current** block timestamp rather than the function argument.

This causes incorrect period indices for historical/future timestamps and can even revert on future timestamps when a future `PeriodConfiguration` is selected.

## Vulnerability Details

In `FirelightVault.sol`, the function currently reads:

```solidity
    function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
        PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
        // solhint-disable-next-line max-line-length
        return periodConfiguration.startingPeriod + _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration;
    }
```

But `_sinceEpoch` always uses the current block time:

```solidity
function _sinceEpoch(uint48 epoch) private view returns (uint48) {
    return Time.timestamp() - epoch; 
}
```

The code subtracts `epoch` from `Time.timestamp()` (the current block time), ignoring the `timestamp` argument. As a result, `periodAtTimestamp(t)` returns the **current** period rather than the period for `t`.

If a future configuration is selected by periodConfigurationAtTimestamp(timestamp), the expression `Time.timestamp() - periodConfiguration.epoch` underflows and reverts in Solidity ≥0.8, causing a DoS for callers querying certain future timestamps.

## Impact Details

While there is no direct vector impacting funds right now, this is a bug that can break offchain queries or any integration relying on accurate period indexing.

## Proof of Concept

## Proof of Concept

This is the log I get from the following PoC:

```
  POC: periodAtTimestamp Vulnerability

=== Initial State ===
Deployment time: 1762811863
Initial epoch: 1762811862
Period duration: 86400 seconds
Current period: 0

=== Test 1: Past Timestamp Query ===

After advancing 5 periods:
Current time: 1763243863
Current period: 5

Querying period at deployment time (1762811863):
Expected period: 0
Actual period returned: 5

*** BUG CONFIRMED: Returns 5 instead of 0 ***
The function ignores the timestamp parameter and returns the CURRENT period!

Querying period 2 periods ago (1763071063):
Expected period: 3
Actual period returned: 5
*** BUG: Returns 5 instead of 3 ***
    ✔ BUG DEMONSTRATION: periodAtTimestamp returns incorrect period for PAST timestamps

=== Test 2: Future Timestamp Query ===
Current time: 1763243863
Current period: 5

Querying period 10 periods in the future (1764107863):
Expected period: 15
Actual period returned: 5

*** BUG CONFIRMED: Returns 5 instead of 15 ***
The function ignores the timestamp parameter and returns the CURRENT period!
    ✔ BUG DEMONSTRATION: periodAtTimestamp returns incorrect period for FUTURE timestamps

=== Test 3: Current Timestamp Query (Works Correctly) ===
Current time: 1763243863
Current period: 5
Period at current timestamp: 5

*** This works correctly because the timestamp matches current time ***
This is why the bug was not detected in existing tests!
    ✔ CORRECT BEHAVIOR: periodAtTimestamp works for CURRENT timestamp (masks the bug)


  3 passing (747ms)
```

Proof of Concept which uses project test suite:

```
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures.js')
const { expect } = require('chai')
const { ethers } = require('hardhat')

describe('POC: periodAtTimestamp Vulnerability', function() {
  let firelight_vault, deploymentTime, initialEpoch
  const DECIMALS = 6,
        INITIAL_DEPOSIT_LIMIT = ethers.parseUnits('20000', DECIMALS),
        PERIOD_DURATION = 86400  // 1 day in seconds

  before(async () => {
    ({ firelight_vault } = await loadFixture(
      deployVault.bind(null, {
        decimals: DECIMALS,
        initial_deposit_limit: INITIAL_DEPOSIT_LIMIT,
        period_configuration_duration: PERIOD_DURATION
      })
    ))

    deploymentTime = await time.latest()

    // Get the initial epoch from the period configuration
    const config = await firelight_vault.periodConfigurations(0)
    initialEpoch = Number(config[0])

    console.log('\n=== Initial State ===')
    console.log(`Deployment time: ${deploymentTime}`)
    console.log(`Initial epoch: ${initialEpoch}`)
    console.log(`Period duration: ${PERIOD_DURATION} seconds`)
    console.log(`Current period: ${await firelight_vault.currentPeriod()}`)
  })

  it('BUG DEMONSTRATION: periodAtTimestamp returns incorrect period for PAST timestamps', async () => {
    console.log('\n=== Test 1: Past Timestamp Query ===')

    // Advance time by 5 periods (5 days)
    const periodsToAdvance = 5
    await time.increase(PERIOD_DURATION * periodsToAdvance)

    const currentTime = await time.latest()
    const currentPeriod = await firelight_vault.currentPeriod()

    console.log(`\nAfter advancing ${periodsToAdvance} periods:`)
    console.log(`Current time: ${currentTime}`)
    console.log(`Current period: ${currentPeriod}`)

    // Calculate what the period SHOULD be at deployment time
    // At deployment, we were in period 0
    const expectedPeriodAtDeployment = 0n

    // Query for period at deployment time (should return period 0)
    const actualPeriodAtDeployment = await firelight_vault.periodAtTimestamp(deploymentTime)

    console.log(`\nQuerying period at deployment time (${deploymentTime}):`)
    console.log(`Expected period: ${expectedPeriodAtDeployment}`)
    console.log(`Actual period returned: ${actualPeriodAtDeployment}`)
    console.log(`\n*** BUG CONFIRMED: Returns ${actualPeriodAtDeployment} instead of ${expectedPeriodAtDeployment} ***`)
    console.log(`The function ignores the timestamp parameter and returns the CURRENT period!`)

    // Verify the bug exists
    expect(actualPeriodAtDeployment).to.equal(currentPeriod,
      'BUG: periodAtTimestamp returns current period instead of historical period')
    expect(actualPeriodAtDeployment).to.not.equal(expectedPeriodAtDeployment,
      'BUG: periodAtTimestamp does not return the correct historical period')

    // Test with another historical timestamp (2 periods ago)
    const twoPeriodsAgo = currentTime - (PERIOD_DURATION * 2)
    const expectedPeriodTwoPeriodsAgo = currentPeriod - 2n
    const actualPeriodTwoPeriodsAgo = await firelight_vault.periodAtTimestamp(twoPeriodsAgo)

    console.log(`\nQuerying period 2 periods ago (${twoPeriodsAgo}):`)
    console.log(`Expected period: ${expectedPeriodTwoPeriodsAgo}`)
    console.log(`Actual period returned: ${actualPeriodTwoPeriodsAgo}`)
    console.log(`*** BUG: Returns ${actualPeriodTwoPeriodsAgo} instead of ${expectedPeriodTwoPeriodsAgo} ***`)

    expect(actualPeriodTwoPeriodsAgo).to.equal(currentPeriod)
    expect(actualPeriodTwoPeriodsAgo).to.not.equal(expectedPeriodTwoPeriodsAgo)
  })

  it('BUG DEMONSTRATION: periodAtTimestamp returns incorrect period for FUTURE timestamps', async () => {
    console.log('\n=== Test 2: Future Timestamp Query ===')

    const currentTime = await time.latest()
    const currentPeriod = await firelight_vault.currentPeriod()

    console.log(`Current time: ${currentTime}`)
    console.log(`Current period: ${currentPeriod}`)

    // Query for a timestamp 10 periods in the future
    const periodsInFuture = 10
    const futureTimestamp = currentTime + (PERIOD_DURATION * periodsInFuture)
    const expectedFuturePeriod = currentPeriod + BigInt(periodsInFuture)

    const actualFuturePeriod = await firelight_vault.periodAtTimestamp(futureTimestamp)

    console.log(`\nQuerying period ${periodsInFuture} periods in the future (${futureTimestamp}):`)
    console.log(`Expected period: ${expectedFuturePeriod}`)
    console.log(`Actual period returned: ${actualFuturePeriod}`)
    console.log(`\n*** BUG CONFIRMED: Returns ${actualFuturePeriod} instead of ${expectedFuturePeriod} ***`)
    console.log(`The function ignores the timestamp parameter and returns the CURRENT period!`)
  })

  it('CORRECT BEHAVIOR: periodAtTimestamp works for CURRENT timestamp (masks the bug)', async () => {
    console.log('\n=== Test 3: Current Timestamp Query (Works Correctly) ===')

    const currentTime = await time.latest()
    const currentPeriod = await firelight_vault.currentPeriod()
    const periodAtCurrent = await firelight_vault.periodAtTimestamp(currentTime)

    console.log(`Current time: ${currentTime}`)
    console.log(`Current period: ${currentPeriod}`)
    console.log(`Period at current timestamp: ${periodAtCurrent}`)
    console.log(`\n*** This works correctly because the timestamp matches current time ***`)
    console.log(`This is why the bug was not detected in existing tests!`)

    // This works correctly (but masks the bug)
    expect(periodAtCurrent).to.equal(currentPeriod,
      'periodAtTimestamp works correctly when timestamp equals current time')
  })
})

```
