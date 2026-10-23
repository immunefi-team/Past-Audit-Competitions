# #59820 \[SC-Low] periodattimestamp returns current period instead of historical period

**Submitted on Nov 16th 2025 at 05:09:20 UTC by @Pro\_King for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59820
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `periodAtTimestamp(uint48 timestamp)` function is designed to return the period number for any given timestamp but always returns the current period instead. The bug occurs because the helper function `_sinceEpoch()` uses `Time.timestamp()` (current time) rather than the input `timestamp` parameter. This breaks external integrations, frontend historical displays, and analytics systems that rely on historical period queries. Core vault operations are unaffected since they use `currentPeriod()` which works correctly.

## Vulnerability Details

The `periodAtTimestamp()` function contains a logic error:

```solidity
function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
    PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
    return periodConfiguration.startingPeriod + _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration;
}

function _sinceEpoch(uint48 epoch) private view returns (uint48) {
    return Time.timestamp() - epoch;  //  Always uses current time
}
```

`_sinceEpoch()` calculates elapsed time using `Time.timestamp()` (current block time) instead of the input `timestamp` parameter. The function ignores its input and always returns the current period.

**Example:**

* Period 0: Nov 16-23, Period 1: Nov 23-30, Period 2: Nov 30-Dec 7
* Current time: Nov 30 (Period 2)
* Query: `periodAtTimestamp(Nov_19)` (Nov 19 is in Period 0)
* Expected: Period 0
* Actual: Period 2

## Impact Details

* Historical period-based performance reports are incorrect
* Users see all their historical activity mapped to incorrect periods
* All past activity appears to happen in the current period

## References

* periodAtTimestamp function: [periodAtTimestamp](https://github.com/firelight-protocol/firelight-core/blob/db36312f1fb24efc88c3fde15a760defbc3e6370/contracts/FirelightVault.sol#L246)
* \_sineEpoch: [\_sineEpoch](https://github.com/firelight-protocol/firelight-core/blob/db36312f1fb24efc88c3fde15a760defbc3e6370/contracts/FirelightVault.sol#L795)

## Proof of Concept

## Proof of Concept

Firstly make a file by name of `period_at_timestamp_bug_poc.js` at test folder then for running the test use this command `npx hardhat test test/period_at_timestamp_bug_poc.js`

```solidity
const {
  loadFixture,
  time,
} = require("@nomicfoundation/hardhat-network-helpers");
const { deployVault } = require("./setup/fixtures.js");
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("periodAtTimestamp() Bug - Returns Wrong Period for Historical Timestamps", function () {
  const DECIMALS = 6;
  const INITIAL_DEPOSIT_LIMIT = ethers.parseUnits("20000", DECIMALS);
  const PERIOD_DURATION = 604800; // 7 days in seconds

  async function deployVaultFixture() {
    const fixture = await deployVault({
      decimals: DECIMALS,
      initial_deposit_limit: INITIAL_DEPOSIT_LIMIT,
      period_configuration_duration: PERIOD_DURATION,
    });
    return fixture;
  }

  it("periodAtTimestamp() returns wrong period for historical queries", async function () {
    const { firelight_vault } = await loadFixture(deployVaultFixture);

    // Geting the initial epoch (when vault was deployed)
    const initialConfig = await firelight_vault.periodConfigurations(0);
    const epoch = initialConfig.epoch;
    const duration = initialConfig.duration;
    const startingPeriod = initialConfig.startingPeriod;

    console.log(`Epoch (Period 0 start): ${epoch}`);
    console.log(
      `Duration: ${duration} seconds (${Number(duration) / 86400} days)`
    );
    console.log(`Starting Period: ${startingPeriod}`);

    // Calculating period boundaries manually
    const period0Start = Number(epoch);
    const period0End = period0Start + Number(duration);
    const period1Start = period0End;
    const period1End = period1Start + Number(duration);
    const period2Start = period1End;
    const period2End = period2Start + Number(duration);

    console.log(
      `\nPeriod 0: ${new Date(period0Start * 1000).toISOString()} to ${new Date(
        period0End * 1000
      ).toISOString()}`
    );
    console.log(
      `Period 1: ${new Date(period1Start * 1000).toISOString()} to ${new Date(
        period1End * 1000
      ).toISOString()}`
    );
    console.log(
      `Period 2: ${new Date(period2Start * 1000).toISOString()} to ${new Date(
        period2End * 1000
      ).toISOString()}`
    );

    // Get current time (should be in Period 0 initially)
    const currentTime = await time.latest();
    const currentPeriod = await firelight_vault.currentPeriod();
    console.log(
      `\nCurrent time: ${new Date(currentTime * 1000).toISOString()}`
    );
    console.log(`Current period: ${currentPeriod}`);

    // Picking a timestamp in Period 0 (early in the period)
    const historicalTimePeriod0 =
      Number(period0Start) + Math.floor(Number(duration) / 2); // Middle of Period 0
    console.log(`\n Query Period 0 timestamp`);
    console.log(
      `Query timestamp: ${new Date(historicalTimePeriod0 * 1000).toISOString()}`
    );
    console.log(`Expected period: 0 (because timestamp is in Period 0)`);

    // Query the historical timestamp
    const resultPeriod0 = await firelight_vault.periodAtTimestamp(
      BigInt(historicalTimePeriod0)
    );
    console.log(`Actual period returned: ${resultPeriod0}`);
    console.log(`Current period: ${currentPeriod}`);

    // This should pass if we're still in Period 0,
    expect(resultPeriod0).to.equal(
      0,
      "Should return Period 0 for timestamp in Period 0"
    );

    // Advance time to Period 2
    console.log(`\n Advancing time to Period 2`);
    const timeToAdvance = Number(period2Start) - Number(currentTime) + 1; // Go to start of Period 2
    await time.increase(BigInt(timeToAdvance));

    const newCurrentTime = await time.latest();
    const newCurrentPeriod = await firelight_vault.currentPeriod();
    console.log(
      `New current time: ${new Date(newCurrentTime * 1000).toISOString()}`
    );
    console.log(`New current period: ${newCurrentPeriod}`);
    expect(newCurrentPeriod).to.equal(2, "Should be in Period 2 now");

    // Now query the SAME historical timestamp (Period 0)
    console.log(
      `\n Query Period 0 timestamp again (now we are in Period 2) `
    );
    console.log(
      `Query timestamp: ${new Date(historicalTimePeriod0 * 1000).toISOString()}`
    );
    console.log(`Expected period: 0 (timestamp is still in Period 0)`);
    console.log(`Current period: ${newCurrentPeriod}`);

    const resultPeriod0Again = await firelight_vault.periodAtTimestamp(
      BigInt(historicalTimePeriod0)
    );
    console.log(`Actual period returned: ${resultPeriod0Again}`);

    // THE BUG: It returns Period 2 (current period) instead of Period 0!
    console.log(
      `\nResult: Returned ${resultPeriod0Again}, expected 0`
    );


    // function returns current period instead of historical period
    expect(resultPeriod0Again).to.equal(
      newCurrentPeriod,
      `periodAtTimestamp() returned ${resultPeriod0Again} which matches current period ${newCurrentPeriod}, proving it ignores the input timestamp`
    );
    expect(resultPeriod0Again).to.not.equal(
      0,
      `periodAtTimestamp() should have returned 0 for historical Period 0 timestamp, but returned ${resultPeriod0Again}`
    );

   
    console.log(
      `\n Query Period 1 timestamp (we are in Period 2)`
    );
    const historicalTimePeriod1 =
      Number(period1Start) + Math.floor(Number(duration) / 2); // Middle of Period 1
    console.log(
      `Query timestamp: ${new Date(historicalTimePeriod1 * 1000).toISOString()}`
    );
    console.log(`Expected period: 1 (because timestamp is in Period 1)`);

    const resultPeriod1 = await firelight_vault.periodAtTimestamp(
      BigInt(historicalTimePeriod1)
    );
    console.log(`Actual period returned: ${resultPeriod1}`);

    // returns current period instead of Period 1
    expect(resultPeriod1).to.equal(
      newCurrentPeriod,
      ` periodAtTimestamp() returned ${resultPeriod1} which matches current period ${newCurrentPeriod}, proving it ignores the input timestamp`
    );
    expect(resultPeriod1).to.not.equal(
      1,
      `periodAtTimestamp() should have returned 1 for historical Period 1 timestamp, but returned ${resultPeriod1}`
    );

  
  });

});

```

**Test output:**

```solidity
  periodAtTimestamp() Bug - Returns Wrong Period for Historical Timestamps
Epoch (Period 0 start): 1763268881
Duration: 604800 seconds (7 days)
Starting Period: 0

Period 0: 2025-11-16T04:54:41.000Z to 2025-11-23T04:54:41.000Z
Period 1: 2025-11-23T04:54:41.000Z to 2025-11-30T04:54:41.000Z
Period 2: 2025-11-30T04:54:41.000Z to 2025-12-07T04:54:41.000Z

Current time: 2025-11-16T04:54:42.000Z
Current period: 0

 Query Period 0 timestamp
Query timestamp: 2025-11-19T16:54:41.000Z
Expected period: 0 (because timestamp is in Period 0)
Actual period returned: 0
Current period: 0

 Advancing time to Period 2
New current time: 2025-11-30T04:54:42.000Z
New current period: 2

 Query Period 0 timestamp again (now we are in Period 2) 
Query timestamp: 2025-11-19T16:54:41.000Z
Expected period: 0 (timestamp is still in Period 0)
Current period: 2
Actual period returned: 2

Result: Returned 2, expected 0

 Query Period 1 timestamp (we are in Period 2)
Query timestamp: 2025-11-26T16:54:41.000Z
Expected period: 1 (because timestamp is in Period 1)
Actual period returned: 2
    ✔ periodAtTimestamp() returns wrong period for historical queries (9536ms)


  1 passing (10s)
```
