# 59335 sc low periodattimestamp function returns current period instead of queried period leading to temporary freezing of funds

**Submitted on Nov 11th 2025 at 08:59:36 UTC by @legion for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59335
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Temporary freezing of funds

## Description

## Brief/Intro

The `periodAtTimestamp(uint48 timestamp)` view function in `FirelightVault.sol` contains a critical logic error where it always returns the current period number regardless of the timestamp parameter passed to it. This function is designed to allow users and off-chain systems to determine which period corresponds to any given historical or future timestamp. However, due to incorrect use of `Time.timestamp()` instead of the function parameter in the calculation, all queries return the same value—the currently active period. This breaks withdrawal scheduling automation, historical analytics, and user-facing dashboards, potentially causing users' mature withdrawals to remain temporarily frozen beyond their intended claim window because automated systems cannot correctly determine when periods end.

## Vulnerability Details

### Root Cause

The vulnerability exists in the `periodAtTimestamp` function at line 246-250 of `FirelightVault.sol`:

```solidity
function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
    PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
    // solhint-disable-next-line max-line-length
    return periodConfiguration.startingPeriod + _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration;
}
```

The function correctly retrieves the appropriate `PeriodConfiguration` for the given `timestamp` parameter using `periodConfigurationAtTimestamp(timestamp)`. However, it then calculates the elapsed time by calling the private helper function `_sinceEpoch`:

```solidity
function _sinceEpoch(uint48 epoch) private view returns (uint48) {
    return Time.timestamp() - epoch;
}
```

The `_sinceEpoch` function **always** uses `Time.timestamp()` (the current block timestamp) rather than the `timestamp` parameter passed to `periodAtTimestamp`. This means the elapsed time calculation is always based on the current moment, not the queried timestamp.

### Concrete Example

Consider a vault deployed with a 1-week (604800 seconds) period duration:

**Timeline:**

* Deploy time (T₀): timestamp = 1,000,000
* Current time (T\_now): timestamp = 4,024,000 (5 weeks later, period = 5)

**Expected behavior:**

* `periodAtTimestamp(1,000,000)` → should return `0` (period at deploy)
* `periodAtTimestamp(2,209,600)` → should return `2` (period after 2 weeks)
* `periodAtTimestamp(7,048,000)` → should return `10` (future period)

**Actual behavior:**

* `periodAtTimestamp(1,000,000)` → returns `5` (current period)
* `periodAtTimestamp(2,209,600)` → returns `5` (current period)
* `periodAtTimestamp(7,048,000)` → returns `5` (current period)

All three calls return the same value despite querying different timestamps.

## Impact Detail

***Temporary Freezing of Funds*** The FirelightVault implements an asynchronous withdrawal system where users call `withdraw()` or `redeem()` to queue a withdrawal for the **next period** (`currentPeriod() + 1`). Once that period ends, users can call `claimWithdraw(period)` to receive their funds. The `claimWithdraw` function requires that `period < currentPeriod()`, meaning the period must have ended. This can lead to : **Broken off-chain automation**: Dashboards, bots, and integrations rely on `periodAtTimestamp()` to calculate:

* "When will my withdrawal become claimable?"
* "What period will be active at timestamp X?"
* Historical period lookups for analytics and auditing

## References

* **Vulnerable function**: `FirelightVault.sol#L246-250` - `periodAtTimestamp()`
* **Root cause**: `FirelightVault.sol#L794-796` - `_sinceEpoch()`
* **Related functions correctly using `_sinceEpoch` for current time**:
  * `currentPeriodStart()` - Line 274
  * `currentPeriodEnd()` - Line 282

## Proof of Concept

## Proof of Concept

```javascript

const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures.js')
const { expect } = require('chai')
const { ethers } = require('hardhat')

describe("periodAtTimestamp bug verification", function () {
    let token_contract, firelight_vault, admin, users, utils, config;
    const DECIMALS = 6;
    const PERIOD_DURATION = 604800; // 1 week (from fixtures.js default)

    before(async function () {
        ({ token_contract, firelight_vault, limit_updater, users, utils, config, admin } = await loadFixture(
            deployVault.bind(null, { decimals: DECIMALS, initial_deposit_limit: ethers.parseUnits('1000000', DECIMALS) })
        ));
    });

    it("should demonstrate that periodAtTimestamp ignores its timestamp argument", async function () {
        const deployTimestamp = await time.latest();
        
        // Current period should be 0 at deploy
        const currentPeriod = await firelight_vault.currentPeriod();
        console.log("Current period:", currentPeriod.toString());
        
        // Fast forward 5 weeks (5 periods)
        await time.increase(5 * PERIOD_DURATION);
        
        const newCurrentPeriod = await firelight_vault.currentPeriod();
        console.log("Current period after 5 weeks:", newCurrentPeriod.toString());
        
        // Query for period at deploy time (should be 0)
        const periodAtDeploy = await firelight_vault.periodAtTimestamp(deployTimestamp);
        console.log("Period at deploy timestamp (should be 0):", periodAtDeploy.toString());
        
        // Query for period 2 weeks after deploy (should be 2)
        const twoWeeksAfter = deployTimestamp + (2 * PERIOD_DURATION);
        const periodAt2Weeks = await firelight_vault.periodAtTimestamp(twoWeeksAfter);
        console.log("Period at 2 weeks after deploy (should be 2):", periodAt2Weeks.toString());
        
        // Query for a future time (should be in the future, but returns current)
        const futureTime = (await time.latest()) + (10 * PERIOD_DURATION);
        const periodAtFuture = await firelight_vault.periodAtTimestamp(futureTime);
        console.log("Period at future timestamp (should be > current):", periodAtFuture.toString());
        
        // BUG: All three calls should return different values but they all return the current period
        console.log("\nBUG VERIFICATION:");
        console.log("All three queries return current period:", newCurrentPeriod.toString());
        console.log("periodAtDeploy == currentPeriod?", periodAtDeploy === newCurrentPeriod);
        console.log("periodAt2Weeks == currentPeriod?", periodAt2Weeks === newCurrentPeriod);
        console.log("periodAtFuture == currentPeriod?", periodAtFuture === newCurrentPeriod);
        
        // This demonstrates the bug: regardless of the timestamp argument,
        // periodAtTimestamp always returns the current period
        expect(periodAtDeploy).to.equal(newCurrentPeriod);
        expect(periodAt2Weeks).to.equal(newCurrentPeriod);
        expect(periodAtFuture).to.equal(newCurrentPeriod);
    });
});
```
