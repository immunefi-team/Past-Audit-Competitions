# 59355 sc low periodattimestamp ignores the input timestamp and returns the wrong period for non current timestamps

**Submitted on Nov 11th 2025 at 13:03:14 UTC by @Diavol0 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59355
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

* Impact (Immunefi Smart Contract impacts):
  * Contract fails to deliver promised returns, but doesn’t lose value
  * Griefing (misleading result can cause failed calls and wasted gas)
* Affected assets:
  * contracts/FirelightVault.sol:246
  * contracts/FirelightVault.sol:795

Summary

* Logic error: the NatSpec states “Returns the period number for the timestamp given,” but the function derives the period from the current block time instead of the provided `timestamp`. It only returns a correct value when the input happens to be the current timestamp.
* Internal usage remains correct because `currentPeriod()` calls `periodAtTimestamp(Time.timestamp())`. Given that `_sinceEpoch(epoch)` equals `Time.timestamp() - epoch`, the buggy implementation coincidentally matches the expected value for that specific path. The impact is external-facing (integrations/UX) for past/future timestamps.

Root Cause

* The computation ignores the input argument and uses the current time:
  * Current code: `periodConfiguration.startingPeriod + _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration`
  * `_sinceEpoch` is `Time.timestamp() - epoch`, discarding the `timestamp` passed to `periodAtTimestamp`.

Impact details

* External callers computing periods for non-current timestamps receive wrong values, which can lead to:
  * Passing incorrect period numbers to other functions and triggering reverts (wasted gas).
  * Incorrect off-chain accounting/scheduling for historical/future periods.
* Protocol funds and internal accounting are unaffected because `currentPeriod()` remains correct.

References

* Code: contracts/FirelightVault.sol:246, contracts/FirelightVault.sol:795
* Test evidence: test/audit\_findings.test.js:20 (“periodAtTimestamp uses current time instead of input”).

Remediation

* Use the input timestamp in the calculation:
  * `periodConfiguration.startingPeriod + (timestamp - periodConfiguration.epoch) / periodConfiguration.duration`
* The existing note about far-future reliability (due to future configuration changes) can stay; this fix restores correctness for any given timestamp under the current configuration.

## Proof of Concept

Code (Hardhat test)

```js
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { expect } = require('chai')
const { deployVault } = require('./setup/fixtures.js')

describe('periodAtTimestamp uses current time instead of input', function () {
  before(async () => {
    ({ firelight_vault } = await loadFixture(deployVault.bind()))
  })

  it('should compute period based on the input timestamp (expected), but returns current period (actual)', async () => {
    // Gather current period configuration parameters
    const [epoch, duration, startingPeriod] = await firelight_vault.currentPeriodConfiguration()

    const currentPeriod = await firelight_vault.currentPeriod()

    // Choose a future timestamp several periods ahead without advancing chain time
    const futureTs = Number(await firelight_vault.currentPeriodEnd()) + Number(duration) * 3

    // Expected correct computation based on the function docs
    const expected = startingPeriod + BigInt(Math.floor((futureTs - Number(epoch)) / Number(duration)))

    // Actual contract output (buggy: ignores the input timestamp and uses block timestamp)
    const actual = await firelight_vault.periodAtTimestamp(futureTs)

    // Sanity: expected future period should differ from currentPeriod
    expect(expected).to.not.equal(currentPeriod)

    // This assertion FAILS with the current implementation, demonstrating the bug
    expect(actual).to.equal(expected)
  })
})
```

How to run

* Create a new file `test/periodAtTimestamp.poc.js` in your project and paste the snippet above.
* Run: `npx hardhat test test/periodAtTimestamp.poc.js --no-compile`

Expected result

* The final assertion fails. Example output observed: `AssertionError: expected 0 to equal 4` (actual equals the current period, not the computed future period), proving the function ignores the provided timestamp.
