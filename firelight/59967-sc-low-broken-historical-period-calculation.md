# #59967 \[SC-Low] broken historical period calculation

**Submitted on Nov 17th 2025 at 08:46:37 UTC by @dice for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59967
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The periodAtTimestamp(uint48 timestamp) function is designed to return the period number for any given timestamp parameter. However, it incorrectly uses `block.timestamp` instead of the provided timestamp parameter in its internal calculation via \_sinceEpoch() at line 798-800. This causes the function to always return the current period number regardless of what historical timestamp is passed in, completely breaking historical period lookups and violating the function's documented behaviour.

```solidity
  function _sinceEpoch(uint48 epoch) private view returns (uint48) {
      return Time.timestamp() - epoch;  // Always uses current time
  }


  // Line 246-250: periodAtTimestamp ignores its parameter
  function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
      PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
      return periodConfiguration.startingPeriod + _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration;
      //                                          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
      //                                          Uses block.timestamp, NOT the timestamp parameter!
  }
```

Vulnerability Details

The root cause is in `_sinceEpoch()` which computes Time.timestamp() - epoch instead of `timestamp - epoch`. When periodAtTimestamp() calls this helper function at line 249, it passes the correct epoch but the calculation uses current block.timestamp, making periodAtTimestamp(50 days) and periodAtTimestamp(99 days) return identical results. While periodConfigurationAtTimestamp(timestamp) correctly finds the right configuration for the input timestamp, the subsequent period number calculation is done using current time, rendering the timestamp parameter completely useless.

Example

```
  // Assume: epoch = day 0, duration = 7 days, current time = day 100

  uint256 period1 = vault.periodAtTimestamp(50 days);
  uint256 period2 = vault.periodAtTimestamp(99 days);
  uint256 period3 = vault.currentPeriod();

  // Expected behavior:
  // period1 should be: 0 + (50 - 0) / 7 = 7
  // period2 should be: 0 + (99 - 0) / 7 = 14

  // Actual behavior (BUG):
  // period1 = 0 + (100 - 0) / 7 = 14  
  // period2 = 0 + (100 - 0) / 7 = 14  
  // period3 = 0 + (100 - 0) / 7 = 14 

  // Result: period1 == period2 == period3 (all return current period!)
```

Impact Details

This bug completely `breaks` any historical period queries, causing all external systems, governance mechanisms to determine which period a past event occurred in to receive incorrect data showing the current period instead.

The function `violates` its NatSpec documentation which promises to return "the period number corresponding to the given timestamp", creating a critical trust and integration issue. While not directly exploitable for `fund theft` in the current implementation, it poses systemic risk for any future upgrades or external integrations that rely on historical period calculations for time-weighted logic, snapshot-based voting, or accounting systems. Additionally, it makes the withdrawal queue system's historical state unreconstructable, preventing accurate audits of past withdrawal requests and their corresponding periods.

## Recommended Fix

function \_sinceEpoch(uint48 epoch, uint48 timestamp) private pure returns (uint48) { return timestamp - epoch; // Use the timestamp parameter }

## References

<https://github.com/firelight-protocol/firelight-core/blob/db36312f1fb24efc88c3fde15a760defbc3e6370/contracts/FirelightVault.sol#L249>

<https://github.com/firelight-protocol/firelight-core/blob/db36312f1fb24efc88c3fde15a760defbc3e6370/contracts/FirelightVault.sol#L795>

## Proof of Concept

## Proof of Concept

```
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures.js')
const { expect } = require('chai')
const { ethers } = require('hardhat')

/**
 * POC: periodAtTimestamp() Bug
 *
 * This test demonstrates a bug where periodAtTimestamp(timestamp)
 * ignores the timestamp parameter and always returns the current period number.
 *
 * ROOT CAUSE: _sinceEpoch() uses Time.timestamp() instead of the provided timestamp
 */
describe('POC: periodAtTimestamp() Always Returns Current Period', function() {
  let firelight_vault, deployment_time

  const DECIMALS = 6,
        INITIAL_DEPOSIT_LIMIT = ethers.parseUnits('50000', DECIMALS),
        PERIOD_DURATION = 86400 // 1 day = 86400 seconds

  before(async () => {
    ({ firelight_vault } = await loadFixture(
      deployVault.bind(null, {
        decimals: DECIMALS,
        initial_deposit_limit: INITIAL_DEPOSIT_LIMIT,
        period_configuration_duration: PERIOD_DURATION
      })
    ))

    deployment_time = await time.latest()
  })

  it('BUG: periodAtTimestamp() ignores timestamp parameter and returns current period', async () => {
    console.log('\n=== POC: Demonstrating periodAtTimestamp() Bug ===\n')

    // Record initial state
    const initial_period = await firelight_vault.currentPeriod()
    const initial_time = await time.latest()

    console.log(`Initial State:`)
    console.log(`  Current Time: ${initial_time}`)
    console.log(`  Current Period: ${initial_period}\n`)

    // Move forward 5 days (5 periods)
    await time.increase(PERIOD_DURATION * 5)

    const current_time = await time.latest()
    const current_period = await firelight_vault.currentPeriod()

    console.log(`After 5 Days:`)
    console.log(`  Current Time: ${current_time}`)
    console.log(`  Current Period: ${current_period}\n`)

    // Test 1: Query period at deployment time (period 0)
    const period_at_deployment = await firelight_vault.periodAtTimestamp(deployment_time)
    console.log(`Test 1: Query period at deployment time (${deployment_time})`)
    console.log(`  Expected: Period 0`)
    console.log(`  Actual: Period ${period_at_deployment}`)
    console.log(`   BUG: Should be 0 but got ${period_at_deployment}\n`)

    // Test 2: Query period at 2 days after deployment (period 2)
    const timestamp_day_2 = deployment_time + PERIOD_DURATION * 2
    const period_at_day_2 = await firelight_vault.periodAtTimestamp(timestamp_day_2)
    console.log(`Test 2: Query period at day 2 (${timestamp_day_2})`)
    console.log(`  Expected: Period 2`)
    console.log(`  Actual: Period ${period_at_day_2}`)
    console.log(`   BUG: Should be 2 but got ${period_at_day_2}\n`)

    // Test 3: Query period at 3 days after deployment (period 3)
    const timestamp_day_3 = deployment_time + PERIOD_DURATION * 3
    const period_at_day_3 = await firelight_vault.periodAtTimestamp(timestamp_day_3)
    console.log(`Test 3: Query period at day 3 (${timestamp_day_3})`)
    console.log(`  Expected: Period 3`)
    console.log(`  Actual: Period ${period_at_day_3}`)
    console.log(`   BUG: Should be 3 but got ${period_at_day_3}\n`)

    // Test 4: All historical queries return the same value (current period)
    console.log(`Test 4: All different timestamps return the same period`)
    console.log(`  periodAtTimestamp(deployment_time) = ${period_at_deployment}`)
    console.log(`  periodAtTimestamp(day 2) = ${period_at_day_2}`)
    console.log(`  periodAtTimestamp(day 3) = ${period_at_day_3}`)
    console.log(`  currentPeriod() = ${current_period}`)
    console.log(`   BUG: All return ${current_period} (current period)!\n`)

    // Assertions proving the bug
    expect(period_at_deployment).to.equal(current_period, 'BUG: Historical query returned current period')
    expect(period_at_day_2).to.equal(current_period, 'BUG: Historical query returned current period')
    expect(period_at_day_3).to.equal(current_period, 'BUG: Historical query returned current period')

    // These are what SHOULD happen (but don't):
    // expect(period_at_deployment).to.equal(0) // ✓ Should be period 0
    // expect(period_at_day_2).to.equal(2)      // ✓ Should be period 2
    // expect(period_at_day_3).to.equal(3)      // ✓ Should be period 3

    console.log(`\n=== Explanation ===`)
    console.log(`The _sinceEpoch() function uses Time.timestamp() (current time)`)
    console.log(`instead of the timestamp parameter passed to periodAtTimestamp().`)
    console.log(`This makes all historical period queries return currentPeriod().\n`)
  })

  it('IMPACT: Historical verification becomes impossible', async () => {
    console.log('\n=== Impact Demonstration ===\n')

    // Simulate a user making a withdrawal request at a specific time
    const withdrawal_request_time = await time.latest()
    const expected_withdrawal_period = await firelight_vault.currentPeriod()

    console.log(`User makes withdrawal request:`)
    console.log(`  Request Time: ${withdrawal_request_time}`)
    console.log(`  Expected Period: ${expected_withdrawal_period}\n`)

    // Move forward 3 periods
    await time.increase(PERIOD_DURATION * 3)

    const current_time = await time.latest()
    const current_period = await firelight_vault.currentPeriod()

    console.log(`3 Days Later:`)
    console.log(`  Current Time: ${current_time}`)
    console.log(`  Current Period: ${current_period}\n`)

    // Try to verify the withdrawal period historically
    const verified_period = await firelight_vault.periodAtTimestamp(withdrawal_request_time)

    console.log(`Attempting to verify historical withdrawal period:`)
    console.log(`  Querying periodAtTimestamp(${withdrawal_request_time})`)
    console.log(`  Expected Result: Period ${expected_withdrawal_period}`)
    console.log(`  Actual Result: Period ${verified_period}`)
    console.log(`  Current Period: ${current_period}\n`)

    // The verification is broken
    expect(verified_period).to.equal(current_period, 'Bug: Returns current period instead of historical')
    expect(verified_period).to.not.equal(expected_withdrawal_period, 'Bug: Cannot verify historical period')

    console.log(` IMPACT: External contracts cannot verify historical withdrawal periods`)
    console.log(` IMPACT: Auditing tools cannot reconstruct past state`)
    console.log(` IMPACT: Governance systems using historical data will fail\n`)
  })

  it('IMPACT: Function documentation is violated', async () => {
    console.log('\n=== Documentation Violation ===\n')

    const past_timestamp = deployment_time + PERIOD_DURATION
    const result = await firelight_vault.periodAtTimestamp(past_timestamp)
    const current = await firelight_vault.currentPeriod()

    console.log(`Function NatSpec says:`)
    console.log(`  "Returns the period number for the timestamp given"`)
    console.log(`  "@param timestamp The timestamp to find the period number for"`)
    console.log(`  "@return The period number corresponding to the given timestamp"\n`)

    console.log(`Reality:`)
    console.log(`  Input: timestamp = ${past_timestamp}`)
    console.log(`  Output: period = ${result}`)
    console.log(`  Current Period: ${current}\n`)

    expect(result).to.equal(current, 'Function returns current period, not period for given timestamp')

    console.log(` The function ALWAYS returns currentPeriod()`)
    console.log(` The timestamp parameter is effectively ignored`)
    console.log(` This violates the documented behavior\n`)
  })

  it('COMPARISON: currentPeriod() vs periodAtTimestamp(past)', async () => {
    console.log('\n=== Side-by-Side Comparison ===\n')

    const timestamps = []
    const periods_via_function = []

    // Collect data over 5 periods
    for (let i = 0; i < 5; i++) {
      timestamps.push(await time.latest())
      await time.increase(PERIOD_DURATION)
    }

    const current_period = await firelight_vault.currentPeriod()
    const current_time = await time.latest()

    console.log(`Current State:`)
    console.log(`  Time: ${current_time}`)
    console.log(`  Period: ${current_period}\n`)

    console.log(`Querying Historical Periods:\n`)
    console.log(`Timestamp\t\tExpected\tActual\t\tMatch?`)
    console.log(`─────────────────────────────────────────────────────────`)

    for (let i = 0; i < timestamps.length; i++) {
      const period = await firelight_vault.periodAtTimestamp(timestamps[i])
      const expected = i
      const match = period == BigInt(expected) ? '✓' : ''
      console.log(`${timestamps[i]}\t${expected}\t\t${period}\t\t${match}`)
      periods_via_function.push(period)
    }

    console.log(`\n`)

    // All should return different values, but they all return current_period
    const all_same = periods_via_function.every(p => p == current_period)
    expect(all_same).to.be.true

    console.log(`Result: All queries returned period ${current_period} (the current period)`)
    console.log(`Expected: Should have returned periods 0, 1, 2, 3, 4\n`)
  })
})

```
