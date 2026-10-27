# #42710 \[SC-Medium] Modulo opation introduces bias during the winning yeet calculation

**Submitted on Mar 25th 2025 at 13:03:42 UTC by @RNemes for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42710
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Yeetback.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Theft of unclaimed yield

## Description

## Brief/Intro

Randomness of game results decreases due to use of the Modulo operator in selecting the winning yeets.

## Vulnerability Details

The `draftWinners` function uses the modulo operation to select the winning yeet. The Pyth documentation mentions:

[Pyth Docs](https://docs.pyth.network/entropy/best-practices#generating-random-values-within-a-specific-range)

```
Notice that using the modulo operator can distort the distribution of random numbers if it's not a power of 2. This is negligible for small and medium ranges, but it can be noticeable for large ranges. For example, if you want to generate a random number between 1 and 52, the probability of having value 5 is approximately 10^-77 higher than the probability of having value 50 which is infinitesimal.
```

This is not an issue for a small number of yeets ie 52 but as the number of yeets per game gets larger the effects are greater and will cause a bias in the results.

## Impact Details

As the number of players increase the randomness of the results decreases, which causes unfairness in the game.

## References

[Pyth Docs](https://docs.pyth.network/entropy/best-practices#generating-random-values-within-a-specific-range)

## Proof of Concept

## Proof of Concept

The following test shows how the randomness changes as the no of yeets increases

```solidity
function test_ModuloBias() public {
        // Test both a power of 2 and non-power of 2
        checkDistribution(1023);  // non-power of 2
        checkDistribution(1024);  // power of 2
        checkDistribution(52);  // non-power of 2
        checkDistribution(64);  // non-power of 2
    }

    function checkDistribution(uint256 nrOfYeets) private {
        uint256[] memory counts = new uint256[](nrOfYeets);
        uint256 iterations = 1_00_000;  // Use more iterations for better statistical significance
        
        for (uint256 i = 0; i < iterations; i++) {
            uint256 randomDataNumber = uint256(keccak256(abi.encodePacked(i)));
            uint256 index = randomDataNumber % nrOfYeets;
            counts[index]++;
        }

        uint256 expectedCount = iterations / nrOfYeets;
        uint256 sumDeviations = 0;
        uint256 maxDeviation = 0;
        uint256 minCount = type(uint256).max;
        uint256 maxCount = 0;

        for (uint256 i = 0; i < nrOfYeets; i++) {
            uint256 deviation;
            if (counts[i] > expectedCount) {
                deviation = counts[i] - expectedCount;
            } else {
                deviation = expectedCount - counts[i];
            }
            
            sumDeviations += deviation;
            maxDeviation = deviation > maxDeviation ? deviation : maxDeviation;
            minCount = counts[i] < minCount ? counts[i] : minCount;
            maxCount = counts[i] > maxCount ? counts[i] : maxCount;
        }

        uint256 avgDeviation = sumDeviations / nrOfYeets;
        
        emit log_named_uint("Number of possible values", nrOfYeets);
        emit log_named_uint("Expected count per value", expectedCount);
        emit log_named_uint("Minimum count observed", minCount);
        emit log_named_uint("Maximum count observed", maxCount);
        emit log_named_uint("Average deviation", avgDeviation);
        emit log_named_uint("Maximum deviation", maxDeviation);
        emit log_named_uint("Max deviation percentage", (maxDeviation * 100) / expectedCount);
        emit log_named_uint("Spread (max-min)", maxCount - minCount);
    }
```
