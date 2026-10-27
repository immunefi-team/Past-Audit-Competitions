# 58076 sc insight fix unit mismatch in doliquidation collateralinunderlying collateralindebt

## #58076 \[SC-Insight] Fix unit mismatch in \_doLiquidation: collateralInUnderlying -> collateralInDebt

**Submitted on Oct 30th 2025 at 13:05:06 UTC by @wylis for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58076
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**

### Description

## Brief

The liquidation path mixes up parameter naming/units in `AlchemistV3._doLiquidation()`. The argument named `collateralInUnderlying` is actually expected to be expressed in debt tokens. While current call sites may be correct, the inconsistency creates a high risk of future regressions where a developer passes an underlying-denominated value, leading to mispriced liquidations and incorrect fee calculations.

## Details

Current code reads:

```solidity

    /// @param collateralInUnderlying The total collateral value of the account in debt tokens.
```

The parameter name is wrong, but the NatSpec is right.

The name `collateralInUnderlying` indicates “in underlying tokens,” but the doc string says “in debt tokens.” The function downstream (`calculateLiquidation`) expects a debt-denominated value, so the parameter name is misleading.

## Impact

This is submitted as an Insight (documentation/naming inconsistency). However, if a future change misuses the parameter due to the name, realistic downstream impacts include: under- or over-liquidation, incorrect fee routing, or invariant breaks under market moves.

## Recommendation

```solidity
-    /// @param collateralInUnderlying The total collateral value of the account in debt tokens.
+    /// @param collateralInDebt The total collateral value of the account in debt tokens.
     /// @param repaidAmountInYield The amount of debt repaid in yield tokens.
-    function _doLiquidation(uint256 accountId, uint256 collateralInUnderlying, uint256 repaidAmountInYield)
+    function _doLiquidation(uint256 accountId, uint256 collateralInDebt, uint256 repaidAmountInYield)
         internal
         returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying)
     {
         Account storage account = _accounts[accountId];

         (uint256 liquidationAmount, uint256 debtToBurn, uint256 baseFee, uint256 outsourcedFee) = calculateLiquidation(
-            collateralInUnderlying,
+            collateralInDebt,
             ...
         );
```

### Proof of Concept

### Proof of Concept

N/A for doc report.
