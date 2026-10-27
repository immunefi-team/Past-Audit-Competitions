# 59740 \[SC-Low] periodattimestamp provides period of current timestamp even for different timestamps

**Submitted on Nov 15th 2025 at 12:03:27 UTC by @redbeans for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59740
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

function periodAtTimestamp is expected to

> @notice Returns the period number for the timestamp given.

but it returns the period of current timestamp in different timestamps.

## Vulnerability Details

This function uses the \_sinceEpoch and its expected to get how much time has passed since epoch from supplied timestamp.

```solidity
    function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
        PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
        // solhint-disable-next-line max-line-length
        return periodConfiguration.startingPeriod + _sinceEpoch(periodConfiguration.epoch) / periodConfiguration.duration;
    }
```

but \_sinceEpoch uses current timestamp so it will return the current period instead of the supplied timestamp which is wrong.

```solidity
    function _sinceEpoch(uint48 epoch) private view returns (uint48) {
        return Time.timestamp() - epoch;
    }
```

Moreover, this periodAtTimestamp is used only in currentPeriod function which uses the Time.timestamp() so this is works correctly as the issue in periodAtTimestamp always return the period of the current timestamp.

```
    function currentPeriod() public view returns (uint256) {
        return periodAtTimestamp(Time.timestamp());
    }
```

## Impact Details

The only function that use it is currentPeriod() which is fine. If the periodAtTimestamp() used by external party then it will return incorrect periods.

## References

<https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol#L246>

## Link to Proof of Concept

<https://gist.github.com/redbeans9/305c6ca7db24665f2d6993fcf641d636>

## Proof of Concept

## Proof of Concept

```solidity
 function testPeriodAtTimestamp_BugSimple() public {
        uint48 duration = 7 days;
        uint48 jan1 = initTime; // Epoch = Jan 1
        uint48 jan15 = jan1 + (14 days); // Query: Jan 15 → offset 14 days
        uint48 jan29 = jan1 + (28 days); // Current: Jan 29 → offset 28 days

        // Warp to Jan 29 (current time)
        vm.warp(jan29);

        // Call with Jan 15 timestamp
        uint256 returnedPeriod = vault.periodAtTimestamp(jan15);

        console.log("=== Period Bug PoC ===");
        console.log("Init Epoch (Jan 1 TS):", jan1);
        console.log("Query TS (Jan 15):", jan15);
        console.log("Current TS (Jan 29):", jan29);
        console.log("Duration:", duration);
        console.log("Returned period:", returnedPeriod);
        console.log("Expected (14d / 7d): 2 Period");

        // Assert the bug: Returns 4 (wrong) instead of 2
        assertEq(returnedPeriod, 4);
        assertTrue(returnedPeriod != 2); // Explicit mismatch
    }
```
