# #59385 \[SC-Low] timestamp ignored current block time used

**Submitted on Nov 11th 2025 at 18:22:24 UTC by @jesse03 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59385
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  * Temporary freezing of funds
  * Temporary freezing of NFTs

## Description

periodAtTimestamp() calls periodConfigurationAtTimestamp(timestamp) (good), but then ignores the caller‑supplied timestamp and computes with the *current* block time via \_sinceEpoch(periodConfiguration.epoch), which internally does Time.timestamp() - epoch.

This creates two wrong/unsafe behaviors that are directly tied to what periodConfigurationAtTimestamp may return:

1. Wrong result for past/future lookups.\
   For any timestamp != block.timestamp, the returned period number is computed using *now* instead of the provided timestamp. So the function returns the current period, not the period at the given time.
2. Underflow & revert for future timestamps (DoS on view).\
   periodConfigurationAtTimestamp can legitimately return a configuration whose epoch is in the *future* (e.g., you query a time after a scheduled future configuration). When epoch > block.timestamp, \_sinceEpoch(epoch) does block.timestamp - epoch and underflows, causing a revert.

***

Minimal repro:

* Assume current time t=105, first config: epoch=100, duration=10.
* An admin schedules a new config at epoch=110 (the next period boundary) with duration=20.
* Call periodAtTimestamp(115) now (i.e., while block.timestamp is still 105).

Flow:

* periodConfigurationAtTimestamp(115) ⇒ returns the future config {epoch:110, duration:20}.
* \_sinceEpoch(110) ⇒ 105 - 110 ⇒ underflow → revert.

Even without the underflow, for any past timestamp (e.g., timestamp=101) the function wrongly calculates using block.timestamp rather than 101, so the answer is incorrect.

***

Root cause:

```solidity
function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
    PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
    // BUG: uses current time instead of the input timestamp
    return periodConfiguration.startingPeriod
        + _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration;
}

function _sinceEpoch(uint48 epoch) private view returns (uint48) {
    return Time.timestamp() - epoch; // underflows if epoch > now
}
```

Because periodConfigurationAtTimestamp can return a config with epoch > now when you ask about the future, the subsequent subtraction against now is unsafe and logically wrong.

***

Correct fix:

Use the provided timestamp in the arithmetic; do not derive from current time.

```solidity
function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
    PeriodConfiguration memory pc = periodConfigurationAtTimestamp(timestamp);
    // safe: pc.epoch <= timestamp by construction of periodConfigurationAtTimestamp
    return pc.startingPeriod + (timestamp - pc.epoch) / pc.duration;
}
```

You can then delete \_sinceEpoch, or (if you prefer keeping a helper) make it take the reference timestamp explicitly:

```solidity
function _sinceEpoch(uint48 ref, uint48 epoch) private pure returns (uint48) {
    // ref is the timestamp you're evaluating at
    return ref - epoch;
}
```

and call:

```solidity
return pc.startingPeriod + _sinceEpoch(timestamp, pc.epoch) / pc.duration;
```

## Proof of Concept

```
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures.js')
const { expect } = require('chai')

describe('periodAtTimestamp POC (bug reproduction)', function () {
  let firelight_vault, period_configuration_updater

  const DECIMALS = 6,
        INITIAL_DEPOSIT_LIMIT =  '20000000000', // 20k tokens (with 6 decimals)
        PERIOD_CONFIGURATION_DURATION = 172800   // 2 days

  before(async () => {
    ({ firelight_vault, period_configuration_updater } = await loadFixture(
      deployVault.bind(null, {
        decimals: DECIMALS,
        initial_deposit_limit: INITIAL_DEPOSIT_LIMIT,
        period_configuration_duration: PERIOD_CONFIGURATION_DURATION
      })
    ))
  })

  it('BUG: returns current period for a past timestamp (ignores input timestamp)', async () => {
    // Move forward one full period to ensure there is a "past" period to query
    const currentPeriodDuration = Number(
      (await firelight_vault.currentPeriodEnd()) - (await firelight_vault.currentPeriodStart())
    )
    await time.increase(currentPeriodDuration)

    const currentPeriod = await firelight_vault.currentPeriod()
    const currentStart = Number(await firelight_vault.currentPeriodStart())

    // Choose a timestamp strictly in the past, inside the previous period
    const pastTs = currentStart - 1

    // BUG: The function under test uses block.timestamp internally, so this returns the current period
    const periodAtPast = await firelight_vault.periodAtTimestamp(pastTs)
    expect(periodAtPast).to.equal(currentPeriod) // demonstrates the bug
  })

  it('BUG: reverts for future timestamp when future epoch > now (underflow)', async () => {
    // Schedule a new period configuration in the future (at the end of the next period)
    const currentEnd = Number(await firelight_vault.currentPeriodEnd())
    const currentPeriodDuration = Number(
      (await firelight_vault.currentPeriodEnd()) - (await firelight_vault.currentPeriodStart())
    )

    const newEpoch = currentEnd + currentPeriodDuration
    const newDuration = 3 * 24 * 60 * 60 // 3 days, divisible by 1 day

    await firelight_vault.connect(period_configuration_updater).addPeriodConfiguration(newEpoch, newDuration)

    // Query a timestamp after the scheduled future configuration starts, while block.timestamp is still before it
    const queryTs = newEpoch + 1

    // BUG: Internally computes Time.timestamp() - epoch, which underflows and reverts
    await expect(firelight_vault.periodAtTimestamp(queryTs)).to.be.reverted
  })
})
```
