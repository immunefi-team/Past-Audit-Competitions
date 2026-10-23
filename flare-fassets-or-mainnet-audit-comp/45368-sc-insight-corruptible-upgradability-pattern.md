# #45368 \[SC-Insight] Corruptible Upgradability Pattern

**Submitted on May 13th 2025 at 11:10:07 UTC by @Anirruth for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45368
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-labs-ltd/fassets/blob/main/docs/ImmunefiScope.md>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Corruptible Upgradability Pattern

## Description

## Brief/Intro

Upgradeable smart contracts without proper storage gaps present a critical vulnerability where storage collisions can occur during contract upgrades. When new variables are added contract during an upgrade, they can overwrite existing variables, corrupting the contract state

## Vulnerability Details

The vulnerability occurs when upgradeable contracts fail to implement storage gaps (\_\_gap) to reserve space for future state variables. If the owner performs an upgrade and wants to add a new variable the storage slots would be overwritten. The following contract are upgradable contracts but doesn't implement storage gaps : CoreVaultManager, FAsset, AssetManagerController, CollateralPool.

## Impact Details

Storage collisions can cause contract variables to be overwritten or misread, leading to corrupted state and unexpected behavior. The entire contract logic may be compromised, causing core functionalities to fail or behave unexpectedly.

## References

Implement `uint[50] private __gap;` for the upgradable contracts mentioned above.

## Proof of Concept

## Proof of Concept

* The owner wants to upgrade the contract and wants to introduce new variables.
* Without gaps, the newly introduces variable would overwrite existing storage slots and corrupt the variables.
* With storage gaps the new variables could use the gap storage slots.
