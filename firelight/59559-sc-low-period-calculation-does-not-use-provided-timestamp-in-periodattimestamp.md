# #59559 \[SC-Low] period calculation does not use provided timestamp in periodattimestamp

**Submitted on Nov 13th 2025 at 14:42:17 UTC by @sahuang for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59559
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary

The `periodAtTimestamp()` function contains a logic error where it uses the current block timestamp instead of the provided `timestamp` parameter when calculating period numbers. This causes two potential failure modes:

1. **Underflow Reverts**: When querying future timestamps before the epoch is reached, the function reverts with an arithmetic underflow
2. **Silent Incorrect Returns**: When querying future timestamps after the epoch is reached, the function returns an incorrect period number based on current time instead of the requested timestamp

Both failures break the contract's ability to provide accurate period information, and this falls in the "Contract fails to deliver promised returns, but doesn't lose value" low-severity impact.

## Finding Description

The vulnerability exists in the interaction between two functions:

1. `periodAtTimestamp(uint48 timestamp)`:

```solidity
function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
    PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
    return periodConfiguration.startingPeriod + _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration;
}
```

2. `_sinceEpoch(uint48 epoch)`:

```solidity
function _sinceEpoch(uint48 epoch) private view returns (uint48) {
    return Time.timestamp() - epoch;  // @audit: Uses current time, not the timestamp parameter
}
```

The `_sinceEpoch()` function **ignores the caller-supplied `timestamp` parameter** and always uses `Time.timestamp()` (current block time). This creates an incorrect calculation path.

## Impact

As discussed above, the function returns incorrect value and satisfies the low severity criteria.

## Recommendation

Modify `_sinceEpoch()` to accept and use the actual timestamp parameter:

```solidity
function _sinceEpoch(uint48 epoch, uint48 timestamp) private pure returns (uint48) {
    return timestamp - epoch;
}
```

Then update respective callers.

## Proof of Concept

## Proof of Concept

We update `period_update.js` and add another test under `describe('Period update test', function()` section:

```js
it("periodAtTimestamp should map to the same period", async () => {
    // choose a timestamp ahead of the current block time
    const t0 = await time.latest();
    const ts = t0 + 2 * PERIOD_CONFIGURATION_DURATION;

    const duration = await current_period_duration();
    expect(duration).to.equal(PERIOD_CONFIGURATION_DURATION);

    // snapshot the computed period before time movement
    const initial = await firelight_vault.periodAtTimestamp(ts);
    // move forward exactly one period
    await time.increase(PERIOD_CONFIGURATION_DURATION);
    // query again after the time jump
    const shifted = await firelight_vault.periodAtTimestamp(ts);

    expect(shifted).to.equal(initial);
});
```

Running the test would fail. This demonstrates a different return value is from expected value.
