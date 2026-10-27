# #41272 \[SC-Insight] Unnecessary precision loss due to division before multiplication in \`getDistribution()\`

**Submitted on Mar 13th 2025 at 07:50:55 UTC by @h2134 for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41272
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Yeet.sol>
* **Impacts:**

## Description

## Brief/Intro

The calculation in `getDistribution()` performs division before multiplication, this can lead to unnecessary funds loss to the protocol and users.

## Vulnerability Details

When calculates values to pot/yeetback/stakers/public goods/treasury, in `getDistribution()`, division operations are performed before multiplication operations, causes unnecessary precision loss.

[Yeet::getDistribution()](https://github.com/yeet-protocol/contracts/blob/16b706d0c771f22e8382401e77dc33a7f4dc5fb6/src/Yeet.sol#L429-L442):

```solidity
    function getDistribution(uint256 yeetAmount) public view returns (uint256, uint256, uint256, uint256, uint256) {
        uint256 scale = gameSettings.SCALE();

@>      uint256 valueAfterTax = (yeetAmount / scale) * (scale - TAX_PER_YEET);
@>      uint256 valueToYeetBack = (yeetAmount / scale) * (YEETBACK_PERCENTAGE);
@>      uint256 valueToPot = (yeetAmount / scale) * (scale - YEETBACK_PERCENTAGE - TAX_PER_YEET);
        uint256 tax = yeetAmount - valueAfterTax;

@>      uint256 valueToStakers = (tax / scale) * TAX_TO_STAKERS;
@>      uint256 publicGoods = (tax / scale) * TAX_TO_PUBLIC_GOODS;
@>      uint256 teamRevenue = (tax / scale) * TAX_TO_TREASURY;

        return (valueToPot, valueToYeetBack, valueToStakers, publicGoods, teamRevenue);
    }
```

## Impact Details

Funds distributed to pot/yeetback/stakers/public goods/treasury can be less than expected, the loss is up to 9999 wei and is minimum though.

## References

Perform multiplication operations first.

```solidity
    function getDistribution(uint256 yeetAmount) public view returns (uint256, uint256, uint256, uint256, uint256) {
        uint256 scale = gameSettings.SCALE();

        uint256 valueAfterTax = (yeetAmount * (scale - TAX_PER_YEET) / scale);
        uint256 valueToYeetBack = (yeetAmount * (YEETBACK_PERCENTAGE) / scale);
        uint256 valueToPot = (yeetAmount * (scale - YEETBACK_PERCENTAGE - TAX_PER_YEET) / scale);
        uint256 tax = yeetAmount - valueAfterTax;

        uint256 valueToStakers = (tax * TAX_TO_STAKERS / scale);
        uint256 publicGoods = (tax * TAX_TO_PUBLIC_GOODS / scale);
        uint256 teamRevenue = (tax * TAX_TO_TREASURY / scale);

        return (valueToPot, valueToYeetBack, valueToStakers, publicGoods, teamRevenue);
    }
```

## Proof of Concept

## Proof of Concept

Run the below test in `Yeet.Test.sol`:

```solidity
    function testAudit_GetDistributionPrecisionLoss() public {
        uint256 yeetAmount = 1e18 + 9999;
        (uint256 valueToPot, uint256 valueToYeetback, uint256 valueToStakers, uint256 publicGoods, uint256 teamRevenue) = yeet.getDistribution(yeetAmount);

        uint256 scale = 10000;
        uint256 TAX_PER_YEET = yeet.TAX_PER_YEET();
        uint256 YEETBACK_PERCENTAGE = yeet.YEETBACK_PERCENTAGE();

        uint256 expectedValueToPot = (yeetAmount * (scale - YEETBACK_PERCENTAGE - TAX_PER_YEET)) / scale;

        uint256 sum = valueToPot + valueToYeetback + valueToStakers + publicGoods + teamRevenue;
        console.log("sum:", sum);

        assertLt(valueToPot, expectedValueToPot);
    }
```
