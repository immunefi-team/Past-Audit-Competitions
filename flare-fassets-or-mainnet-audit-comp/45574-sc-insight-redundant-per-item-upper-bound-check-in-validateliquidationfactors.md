# #45574 \[SC-Insight] Redundant Per‑Item Upper Bound Check in \`validateLiquidationFactors\`

**Submitted on May 17th 2025 at 07:43:47 UTC by @chista0x for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45574
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/SettingsValidators.sol>
* **Impacts:**

## Description

## Brief/Introduction

In the `validateLiquidationFactors` function of `SettingsValidators.sol`, the per-item requirement

```solidity
require(liquidationFactors[i] > SafePct.MAX_BIPS, "factor not above 1");
```

is executed inside the loop for every index. However, because the loop already enforces a strictly increasing sequence (`liquidationFactors[i] > liquidationFactors[i – 1]`), it suffices to check the first element against `SafePct.MAX_BIPS` and infer the rest will also satisfy it. Moving that check outside the loop reduces gas costs by eliminating a redundant comparison on every iteration.

## Optimization Details

Current loop logic:

```solidity
for (uint256 i = 0; i < liquidationFactors.length; i++) {
    require(liquidationFactors[i] > SafePct.MAX_BIPS, "factor not above 1");
    require(vaultCollateralFactors[i] <= liquidationFactors[i], "vault collateral factor higher than total");
    require(i == 0 || liquidationFactors[i] > liquidationFactors[i - 1], "factors not increasing");
}
```

* **Redundancy:** After verifying `liquidationFactors[0] > MAX_BIPS`, the monotonic increase check guarantees

  ```
  liquidationFactors[1] > liquidationFactors[0] > MAX_BIPS,
  liquidationFactors[2] > liquidationFactors[1] > MAX_BIPS,
  …
  ```

  so all subsequent indices automatically satisfy `> MAX_BIPS`.

## Recommendation

Refactor the function to perform the upper‑bound check once before the loop, then omit it inside:

```diff
 function validateLiquidationFactors(
     uint256[] memory liquidationFactors,
     uint256[] memory vaultCollateralFactors
 )
     internal pure
 {
     require(liquidationFactors.length == vaultCollateralFactors.length, "lengths not equal");
     require(liquidationFactors.length >= 1, "at least one factor required");

+    // Only the first factor needs an explicit > MAX_BIPS check;
+    // monotonicity ensures all others are larger.
     require(liquidationFactors[0] > SafePct.MAX_BIPS, "factor not above 1");

     for (uint256 i = 0; i < liquidationFactors.length; i++) {
-        require(liquidationFactors[i] > SafePct.MAX_BIPS, "factor not above 1");
         require(vaultCollateralFactors[i] <= liquidationFactors[i], "vault collateral factor higher than total");
         require(i == 0 || liquidationFactors[i] > liquidationFactors[i - 1], "factors not increasing");
     }
 }
```

## Impact (Insight)

This change reduces the number of SLOADs and comparisons by one per loop iteration, lowering gas consumption for callers—particularly important when many factor pairs are validated. The semantic behavior remains identical, making this an easy, low‑risk optimization.

## Proof of Concept

## Proof of Concept

test:

```
        it("Optimized setLiquidationPaymentFactors", async () => {
            const tx = await assetManager.setLiquidationPaymentFactors([10001, 10002, 10003, 10004, 10005], [10001, 10002, 10003, 10004, 10005], { from: assetManagerController });
        });

```

Gas report for orginal and optimized function:

Orginal : 199,324

Optimized: 199,036
