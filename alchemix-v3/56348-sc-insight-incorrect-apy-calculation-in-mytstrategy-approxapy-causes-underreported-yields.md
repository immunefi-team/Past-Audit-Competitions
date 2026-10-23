# 56348 sc insight incorrect apy calculation in mytstrategy approxapy causes underreported yields

**Submitted on Oct 14th 2025 at 20:05:31 UTC by @Gakarot for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56348
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* **Impacts:**

## Description

## Brief/Intro

`MYTStrategy::_approxAPY()` incorrectly divides the squared APR term by 2 \* SECONDS\_PER\_YEAR, instead of just 2, effectively nullifying compounding growth in APY approximation. This leads to misreported yield rates across strategies.

## Impact Details

APY and APR become near identical, misleading users and allocators relying on the `snapshotYield()` results for decision making or display.

## References

Link to Code ---> <https://github.com/alchemix-finance/v3-poc/blob/b2e2aba046c36ff5e1db6f40f399e93cd2bdaad0/src/MYTStrategy.sol#L230-L234>

## Mitigation

To correct the APY miscalculation, the squared APR term should not be divided by SECONDS\_PER\_YEAR. The denominator should only be 2, restoring the correct compounding approximation formula. Specifically, in `MYTStrategy::_approxAPY()`, replace:

```solidity
-return apr + aprSq / (2 * SECONDS_PER_YEAR);
+return apr + aprSq / 2;
```

## Proof of Concept

## Proof of Concept

* Paste this test in file location: v3-poc/src/test/MYTStrategy.t.sol
* Runnable Command: MAINNET\_RPC\_URL="<https://eth-mainnet.g.alchemy.com/v2/YOUR\\_KEY>" forge test --mt test\_POC\_IncorrectAPYCalculation -vvvvv
* Note: This PoC runs on a local Foundry mainnet fork using a read only Alchemy RPC key.

```solidity
function test_POC_IncorrectAPYCalculation() public view {
        // Setup a base per-second rate that equals roughly 5% APR
        uint256 ratePerSecWad = 1.585e9; // 0.05 / 31_557_600 * 1e18

        // Simulate what snapshotYield() would call internally
        uint256 secondsPerYear = strategy.SECONDS_PER_YEAR();
        uint256 apr = ratePerSecWad * secondsPerYear;
        uint256 aprSq = (apr * apr) / 1e18;

        // Buggy formula used in strategy
        uint256 buggyApy = apr + (aprSq / (2 * secondsPerYear));

        // Correct compounding approximation
        uint256 correctApy = apr + (aprSq / 2);

        console.log("Proof of Incorrect APY Calculation");
        console.log("Base APR (should be ~5%):", apr);
        console.log("Buggy APY (contract formula):", buggyApy);
        console.log("Correct APY (mathematical):", correctApy);

    }
```
