# #59422 \[SC-Low] periodattimestamp ignores the supplied timestamp

**Submitted on Nov 12th 2025 at 08:55:32 UTC by @peller for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59422
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**

## Description

## Brief/Intro

periodAtTimestamp(uint48 timestamp) should return the period index for an arbitrary timestamp, yet the implementation ignores the provided value and always recomputes using the current block time; when the timestamp lies inside a future period configuration the function even underflows and reverts, making historical or predictive queries unreliable.

## Vulnerability Details

1. After locating the relevant PeriodConfiguration, periodAtTimestamp calls \_sinceEpoch ( contracts/FirelightVault.sol:246-249).
2. \_sinceEpoch ignores the caller-supplied timestamp and always computes Time.timestamp() - epoch (contracts/FirelightVault.sol:795-796), so every call effectively returns the period for “now” rather than the requested moment.
3. When admins schedule a future configuration (addPeriodConfiguration accepts newEpoch >= nextPeriodEnd), querying a timestamp that falls inside that future window causes Time.timestamp() < epoch, making the subtraction underflow and revert.
4. Result: historical queries are inaccurate, future queries revert, and any off-chain logic that depends on correct period indices breaks.

## Impact Details

Impact category: Low (contract fails to operate as specified—time-lock and accounting logic relying on accurate period numbers receive wrong data or reverts).

## References

* contracts/FirelightVault.sol:246-249, 795-796 – periodAtTimestamp calls \_sinceEpoch, which always subtracts the current timestamp instead of the supplied argument.
  * contracts/FirelightVault.sol:803-815 – future period configurations are allowed, triggering the underflow case.

## Proof of Concept

## Proof of Concept

add test/period\_at\_timestamp.js

```

const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { expect } = require('chai')
const { deployVault } = require('./setup/fixtures')

describe('periodAtTimestamp behavior', function () {
  before(async () => {
    ({ firelight_vault, period_configuration_updater, config } = await loadFixture(deployVault.bind(null, {})))
  })

  it('returns the current period even for historical timestamps', async () => {
    const initialTimestamp = await time.latest()
    const initialPeriod = await firelight_vault.currentPeriod()
    expect(initialPeriod).to.equal(0n)

    await time.increase(BigInt(config.period_configuration_duration) * 3n)

    const currentPeriod = await firelight_vault.currentPeriod()
    expect(currentPeriod).to.be.gt(initialPeriod)

    const lookup = await firelight_vault.periodAtTimestamp(initialTimestamp)

    expect(lookup).to.equal(currentPeriod)
    expect(lookup).to.not.equal(initialPeriod)
  })

  it('reverts for timestamps scheduled in future period configurations', async () => {
    const currentEnd = await firelight_vault.currentPeriodEnd()
    const duration = config.period_configuration_duration
    const newEpoch = Number(currentEnd) + Number(duration)

    await firelight_vault.connect(period_configuration_updater).addPeriodConfiguration(newEpoch, duration)

    await expect(firelight_vault.periodAtTimestamp(newEpoch)).to.be.reverted
  })
})

```

output

```
npx hardhat test test/period_at_timestamp.js


  periodAtTimestamp behavior
    ✔ returns the current period even for historical timestamps
    ✔ reverts for timestamps scheduled in future period configurations (280ms)


  2 passing (5s)
```
