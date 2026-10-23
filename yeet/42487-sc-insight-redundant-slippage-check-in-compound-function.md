# #42487 \[SC-Insight] Redundant Slippage Check in \`compound\` Function

**Submitted on Mar 24th 2025 at 09:14:50 UTC by @Bluedragon for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42487
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/MoneyBrinter.sol>
* **Impacts:**

## Description

## Summary:

The `compound` function in the `MoneyBrinter` contract includes a redundant slippage check for `islandTokensMinted`. This check is already performed in the Kodiak router during the swap process, making it redundant check and useless gas consumption. Removing this check will simplify the code without affecting functionality and gas optimized.

## Vulnerability Details:

In the `compound` function, the following check is performed:

```solidity
require(islandTokensMinted >= stakingParams.amountSharesMin, "MoneyBrinter: not enough island tokens minted");
```

* **Issue**: This check ensures that the amount of `islandTokensMinted` meets the minimum required amount (`amountSharesMin`) to prevent slippage. However, this check is redundant because the Kodiak router already performs the same slippage check during the swap process.
* **Root Cause**: The slippage check is duplicated in both the `MoneyBrinter` contract and the Kodiak router, leading to unnecessary code complexity.

## Impact:

1. **Code Complexity**: The redundant check adds unnecessary code complexity to the `compound` function.
2. **Gas Overhead**: The redundant check incurs a extra gas consumption.

## Proof of Concept

## Proof of Concept:

1. Review the `compound` function in the `MoneyBrinter` contract.
2. Observe the redundant slippage check for `islandTokensMinted`.
3. As you as see the [Kodiak router.\_addLiquidity](https://beratrail.io/address/0x679a7C63FC83b6A4D9C1F931891d705483d4791F/contract/80094/code) already performs the same slippage check during the swap process.
