# 59635 sc low timestamp agnostic periodattimestamp misreports historical periods breaking time locked logic

**Submitted on Nov 14th 2025 at 10:18:46 UTC by @vivekd for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59635
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

periodAtTimestamp(uint48) is supposed to tell integrators which period corresponds to any arbitrary timestamp. The function instead always returns the current period because it ignores its timestamp argument, so anything relying on historical or future lookups receives incorrect data. This can misconfigure time-locked withdrawals, scheduled upgrades, or analytics built atop the vault.

## Vulnerability Details

* periodAtTimestamp fetches the correct PeriodConfiguration for the supplied timestamp (contracts/FirelightVault.sol:246-248), but then calls \_sinceEpoch(periodConfiguration.epoch).
* \_sinceEpoch unconditionally returns Time.timestamp() - epoch, i.e., "now minus epoch," with no way to pass the caller's timestamp (contracts/FirelightVault.sol:795-797).
* Therefore periodAtTimestamp(x) collapses to startingPeriod + (now - epoch) / duration, which is simply currentPeriod() regardless of x.
* Functions such as currentPeriod() and analytics that depend on this view are fine, but anyone trying to reason about historical or future periods using the public getter gets incorrect data.

## Impact Details

Off-chain systems (withdrawal schedulers, upgrade planners, risk dashboards) cannot safely reason about periods that are not "right now," which can lead to incorrect unlock times or reporting.

## Proof of Concept

## Proof of Concept

```javascript
// test/period_timestamp_poc.js
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { expect } = require('chai')
const { deployVault } = require('./setup/fixtures')

describe('periodAtTimestamp PoC', function () {
  let firelight_vault, config

  before(async () => {
    ({ firelight_vault, config } = await loadFixture(deployVault))
  })

  it('returns the current period even when queried for the original epoch', async () => {
    const firstConfig = await firelight_vault.periodConfigurations(0)
    const genesisEpoch = firstConfig.epoch

    await time.increase(Number(config.period_configuration_duration) * 3)

    const current = await firelight_vault.currentPeriod()
    const historical = await firelight_vault.periodAtTimestamp(genesisEpoch)

    expect(Number(current)).to.be.gt(0)
    expect(historical).to.equal(current)
  })
})
```

### What the code does

Deploys the vault, records the epoch of the first period (time zero), advances three periods, then compares currentPeriod() with periodAtTimestamp(genesisEpoch)—a query that should have returned 0. The test asserts they're equal, showing the getter ignores its argument.

### How to run

1. Install deps if needed: npm install
2. Ensure .env is populated (any dummy 32-byte keys).
3. Run only this test: npx hardhat test test/period\_timestamp\_poc.js

### Logs

```
newuser@LAPTOP-MLPJMQD2:~/firelight-core$ npx hardhat test test/period_timestamp_poc.js


  periodAtTimestamp PoC
    ✔ returns the current period even when queried for the original epoch


  1 passing (411ms)
```

The PoC reproduces the bug: querying the genesis timestamp yields the current period instead of 0.
