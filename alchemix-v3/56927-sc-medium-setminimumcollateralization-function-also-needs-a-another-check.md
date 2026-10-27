# 56927 sc medium setminimumcollateralization function also needs a another check&#x20;

**Submitted on Oct 21st 2025 at 20:11:59 UTC by @griffin for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56927
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**

## Description

## Vulnerability Details

```
    function setMinimumCollateralization(uint256 value) external onlyAdmin {
        _checkArgument(value >= FIXED_POINT_SCALAR);
        minimumCollateralization = value;

        emit MinimumCollateralizationUpdated(value);
    }
```

there is no check to ensure that the new value is greater than or equal to the current collateralizationlowerbound. this can result in a state where collateralizationlowerbound > minimumcollateralization, violating the intended invariant that the lower bound for liquidation should always be less than or equal to the minimum required collateralization for healthy accounts.

## Impact Details

if the admin sets minimumcollateralization to a value below collateralizationlowerbound, the liquidation logic in \_liquidate can trigger on accounts that are above the new minimumcollateralization but below the old collateralizationlowerbound. this leads to unintended liquidations, where healthy accounts (per the updated minimum) are liquidated

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi#L292>

## Proof of Concept

## Proof of Concept

one , admin call setMinimumCollateralization with 50\* 1e18(because FIXED\_POINT\_SCALAR is 1e18)

second, admin call setMinimumCollateralization with 40 \* 1e18

but admin shouldnt be able to do the second step

test code :

```
pragma solidity 0.8.28;

import "src/test/AlchemistV3_6_decimals.t.sol";

contract testM is AlchemistV3Test {
    function setUp() public override {
        super.setUpMYT(6);
    }

    function test_M() public {
        vm.startPrank(admin);
        alchemist.setMinimumCollateralization(50e18); 
        alchemist.setMinimumCollateralization(40e18); 
        assertTrue(alchemist.collateralizationLowerBound() > alchemist.minimumCollateralization(), "inconsistency not created");
    }


```
