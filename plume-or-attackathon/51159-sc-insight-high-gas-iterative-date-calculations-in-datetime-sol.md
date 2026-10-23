# 51159 sc insight high gas iterative date calculations in datetime sol

**Submitted on Jul 31st 2025 at 16:36:06 UTC by @zbugs for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51159
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/DateTime.sol>
* **Impacts:** Gas optimization (high gas usage)

## Description

### Brief/Intro

The `DateTime.sol` contract's timestamp-to-date conversion functions (`parseTimestamp`, `getYear`, `getMonth`, `getDay`) are implemented with highly inefficient iterative logic. This design leads to significantly increased gas costs for any operation in the Plume network that relies on these date calculations, directly impacting user transaction fees and protocol efficiency.

### Vulnerability Details

The gas inefficiency arises from iterative calculations instead of using optimized mathematical algorithms:

* `getYear`'s Inefficient while loop: `getYear` starts with an approximation then iteratively adjusts `year` and `secondsAccountedFor` using a `while` loop, repeatedly checking leap years and subtracting seconds until the correct year is found.
* `parseTimestamp`'s Cascading Iterations: `parseTimestamp` calls the iterative `getYear`, then uses loops to determine month (up to 12 iterations) and day (up to 31 iterations). Each iteration invokes additional calculations (e.g., `getDaysInMonth`), compounding gas usage.

This chain of iterative loops for year, month, and day results in much higher gas consumption than optimized direct-algorithm implementations (for example, Solady's `DateTimeLib`), which convert timestamps to date components in a single, non-iterative pass.

### Impact Details

Primary impact: high gas consumption (Gas Optimization), affecting economic efficiency and user experience in the Plume protocol. The PoC demonstrates an example optimization of 29,087 gas saved.

### References

* Plume DateTime Contract: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/DateTime.sol?utm\\_source=immunefi>
* Solady DateTimeLib (Optimized Reference): <https://github.com/Vectorized/solady/blob/main/src/utils/DateTimeLib.sol>

## Proof of Concept

{% stepper %}
{% step %}

### Setup

* Measure initial gas (`gasleft()`) for both Plume and Solady approaches.
  {% endstep %}

{% step %}

### Execute conversions

* Execute date conversion using Plume's iterative functions and Solady's optimized function for the same timestamp.
  {% endstep %}

{% step %}

### Measure and compare

* Measure final gas (`gasleft()`) after each execution.
* Calculate gas used for each approach and compute savings: `gasUsedPlume - gasUsedSolady`.
* Assert Solady uses less gas and return the gas savings.
  {% endstep %}
  {% endstepper %}

To observe the gas savings, run the benchmarkGasSoladyAsContract() function in Remix IDE after deploying the DateTimeSoladyBenchmark contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

import {DateTimeLib as SoladyDateTimeLib} from "https://github.com/Vectorized/solady/src/utils/DateTimeLib.sol";
import {DateTime as PlumeDateTime} from "https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/DateTime.sol?utm_source=immunefi";

// This is just to mimic the Plume approach of having a dedicated separate contract for DateTime operations
contract SoladyDateTimeAsContract {
    constructor() {}

    function timestampToDate(uint256 timestamp)
        external
        pure
        returns (uint256 year, uint256 month, uint256 day) {
            (year, month, day) = SoladyDateTimeLib.timestampToDate(timestamp);
        }

}


contract DateTimeSoladyBenchmark { // For Remix built-in testing   
    PlumeDateTime plumeDateTime; // Instance of the DateTime contract
    SoladyDateTimeAsContract soladyDateTimeAsContract;

    // Constructor to deploy the DateTime contract
    constructor() {
        plumeDateTime = new PlumeDateTime();
        soladyDateTimeAsContract = new SoladyDateTimeAsContract();
    }

       function benchmarkGasSoladyAsContract() external view returns (uint256) {
        uint256 testTimestamp = 946684800; // Year 2000 Jan 1

        // ---------- SOLADY -----------

        uint256 gasBeforeSolady = gasleft();

        (uint256 soladyYear, uint256 soladyMonth, uint256 soladyDay) = soladyDateTimeAsContract.timestampToDate(testTimestamp);

        uint256 gasAfterSolady = gasleft();
        uint256 gasUsedSolady = gasBeforeSolady - gasAfterSolady;

        // ---------- PLUME -----------

        uint256 gasBeforePlume = gasleft();
        // NOTE: Although this is an external contract call (to the Plume DateTime.sol contract), this is how Plume configures and uses this contract in production
        // Code usage as in Spin.sol https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Spin.sol#L149
        (uint16 plumeYear, uint8 plumeMonth, uint8 plumeDay) = (
            plumeDateTime.getYear(testTimestamp),
            plumeDateTime.getMonth(testTimestamp),
            plumeDateTime.getDay(testTimestamp)
        );

        uint256 gasAfterPlume = gasleft();
        uint256 gasUsedPlume = gasBeforePlume - gasAfterPlume;

        // ----------------------------

        require(soladyYear == plumeYear, "Year 2000 Jan 1 timestamp mismatch");
        require(soladyMonth == plumeMonth, "Year 2000 Jan 1 timestamp mismatch");
        require(soladyDay == plumeDay, "Year 2000 Jan 1 timestamp mismatch");

        require(gasUsedSolady < gasUsedPlume, "Solady worse than Plume");
        return gasUsedPlume - gasUsedSolady;
    }
}
```

Run in Remix: <https://remix.ethereum.org/#lang=en\\&optimize=false\\&runs=200\\&evmVersion=null\\&version=soljson-v0.8.30+commit.73712a01.js>
