# 59369 sc low the function periodattimestamp uses the current timestamp instead of provided timestamp causing incorrect period calculation

**Submitted on Nov 11th 2025 at 16:20:02 UTC by @Y4nhu1 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59369
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The function `periodAtTimestamp` uses the current timestamp for period number calculation, which leads to incorrect results when querying a timestamp other than the current one.

## Vulnerability Details

The function `periodAtTimestamp` should return the period number for the timestamp given. The statement `_sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration` calculates the elapsed period based on the time elapsed since the start time of the current period configuration, i.e. `periodConfiguration.epoch`.

```solidity
function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
    // [...]
    return periodConfiguration.startingPeriod + _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration;
}
```

According to the function definition, the time elapsed from the `periodConfiguration.epoch` to the parameter `timestamp` should be used. However, the function `_sinceEpoch` calculates the time elapsed from the `periodConfiguration.epoch` to the current timestamp.

```solidity
function _sinceEpoch(uint48 epoch) private view returns (uint48) {
    return Time.timestamp() - epoch;
}
```

## Impact Details

This leads to users or other contracts obtaining incorrect results when querying a timestamp other than the current one.

## References

* <https://github.com/firelight-protocol/firelight-core/blob/db36312f1fb24efc88c3fde15a760defbc3e6370/contracts/FirelightVault.sol#L249>
* <https://github.com/firelight-protocol/firelight-core/blob/db36312f1fb24efc88c3fde15a760defbc3e6370/contracts/FirelightVault.sol#L795-L797>

## Proof of Concept

## Proof of Concept

Run the test using the command `npx hardhat test --grep "periodAtTimestamp returns diffe rent values as time progresses"`.

Without adding a new period configuration, the period number for a future timestamp should not change. However, the test results show that over time, `periodAtTimestamp()` returns different results for the same timestamp, indicating a calculation error.

```diff
diff --git a/test/period_update.js b/test/period_update.js
index 97c4dee..1808848 100644
--- a/test/period_update.js
+++ b/test/period_update.js
@@ -33,6 +33,18 @@ describe('Period update test', function() {
     withdraw_period_one = (await withdraw_request.wait()).logs[1].args[3]
   })
 
+  it('periodAtTimestamp returns different values as time progresses', async () => {
+    let timestamp = (await time.latest()) + PERIOD_CONFIGURATION_DURATION * 5;  // a future timestamp
+    
+    expect(await current_period_duration()).to.equal(PERIOD_CONFIGURATION_DURATION);
+
+    const period_at_timestamp_before = await firelight_vault.periodAtTimestamp(timestamp);
+
+    await time.increase(PERIOD_CONFIGURATION_DURATION);
+
+    expect(await firelight_vault.periodAtTimestamp(timestamp)).to.equal(period_at_timestamp_before + 1n);
+  })
+
   it('should validate initial period duration', async () => {
     expect(await current_period_duration()).to.equal(PERIOD_CONFIGURATION_DURATION)
```
