# 50632 sc insight critical timestamp parsing bug in getyear of datetime contract

**Submitted on Jul 26th 2025 at 22:03:45 UTC by @ubl4nk for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50632
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/DateTime.sol>

Impacts:

* It may break core application logic in time-sensitive smart contracts.

## Description

### Brief/Intro

A critical miscalculation exists in the `getYear(uint256 timestamp)` function of the `DateTime` contract. Under certain conditions—particularly near the end of leap years—the function returns an incorrect year, misclassifying timestamps in late December as belonging to the next calendar year (e.g., returning `2028` instead of `2029` OR returning `2033` instead of the correct `2032`).

### Vulnerability Details

Problematic code:

```solidity
function getYear(
    uint256 timestamp
) public pure returns (uint16) {
    uint256 secondsAccountedFor = 0;
    uint16 year;
    uint256 numLeapYears;

    // Rough approximation
    year = uint16(ORIGIN_YEAR + timestamp / YEAR_IN_SECONDS);
    numLeapYears = leapYearsBefore(year) - leapYearsBefore(ORIGIN_YEAR);

    secondsAccountedFor += LEAP_YEAR_IN_SECONDS * numLeapYears;
    secondsAccountedFor += YEAR_IN_SECONDS * (year - ORIGIN_YEAR - numLeapYears);

    while (secondsAccountedFor > timestamp) {
        if (isLeapYear(uint16(year - 1))) {
            secondsAccountedFor -= LEAP_YEAR_IN_SECONDS;
        } else {
            secondsAccountedFor -= YEAR_IN_SECONDS;
        }
        year -= 1;
    }
    return year;
}
```

Expected: `timestamp = 1861910430` ⇒ Should return year `2029`\
Actual: Returns `2028` ❌

This timestamp (`1861910430`) corresponds to `2029-01-01 00:00:30 UTC`, but the function misclassifies it as 2028.

Additional timestamps that fail:

* `1861910460` → 2029-01-01 00:01:00 → returns 2028 ❌
* `1861914000` → 2029-01-01 01:00:00 → returns 2028 ❌
* `1991212000` → 2032-12-31 22:13:20 → returns 2033 ❌
* `1991212199` → 2032-12-31 22:16:39 → returns 2033 ❌
* `1991212500` → 2032-12-31 22:21:40 → returns 2033 ❌

These demonstrate a consistent off-by-one bug near year boundaries due to leap year misestimation.

Root cause:

* The rough approximation line

```solidity
year = uint16(ORIGIN_YEAR + timestamp / YEAR_IN_SECONDS);
```

does not account for the variable duration introduced by leap years. The later correction loop

```solidity
while (secondsAccountedFor > timestamp) { ... }
```

fails to adjust the estimate correctly in some boundary conditions, producing incorrect year outputs.

### Recommendation

Instead of approximating the year and correcting backwards, use a forward loop that counts exact seconds per year. Example suggested implementation:

```solidity
function getYearFixed(uint256 timestamp) public pure returns (uint16) {
    uint16 year = ORIGIN_YEAR;
    uint256 secondsInYear;

    while (true) {
        secondsInYear = isLeapYear(year) ? LEAP_YEAR_IN_SECONDS : YEAR_IN_SECONDS;
        if (timestamp < secondsInYear) {
            break;
        }
        timestamp -= secondsInYear;
        year++;
    }
    return year;
}
```

## Impact Details

It may break core application logic in time-sensitive smart contracts.

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/DateTime.sol#L111-L134>

## Proof of Concept

<details>

<summary>Proof of Concept (expand)</summary>

The report lists concrete timestamps demonstrating the misclassification:

* `1861910430` (2029-01-01 00:00:30 UTC) → function returns 2028 (incorrect)
* `1861910460` (2029-01-01 00:01:00 UTC) → returns 2028 (incorrect)
* `1861914000` (2029-01-01 01:00:00 UTC) → returns 2028 (incorrect)
* `1991212000` (2032-12-31 22:13:20 UTC) → returns 2033 (incorrect)
* `1991212199` (2032-12-31 22:16:39 UTC) → returns 2033 (incorrect)
* `1991212500` (2032-12-31 22:21:40 UTC) → returns 2033 (incorrect)

These examples can be used to reproduce the issue by invoking `getYear(timestamp)` against the referenced `DateTime.sol` implementation.

</details>
