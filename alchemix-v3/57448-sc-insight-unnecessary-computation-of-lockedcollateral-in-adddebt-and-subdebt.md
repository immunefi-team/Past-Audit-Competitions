# 57448 sc insight unnecessary computation of lockedcollateral in adddebt and subdebt&#x20;

**Submitted on Oct 26th 2025 at 10:01:53 UTC by @algiz for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57448
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**

## Description

## Brief/Intro

Locked collateral is computed in both `_addDebt()` and `_subDebt()`, although it has already been updated by the `_sync()`

## Vulnerability Details

Both `_addDebt()` and `_subDebt()` calculate the locked collateral:

```solidity
uint256 lockedCollateral = convertDebtTokensToYield(account.debt) * minimumCollateralization / FIXED_POINT_SCALAR;
```

which makes an external call to the Morpho vault.

* `_addDebt()` is called from `_mint()`, which is called in `_mint()` and `_mintFrom()`
* `_subDebt()` is called from `burn()`, `repay()`, `_forceRepay()` and `_doLiquidation()` (which is called from `_liquidate()`)

In any of those cases the `_sync()` function has been called prior, hence the locked collateral has been already calculated and up to date in `rawLocked`:

```solidity
// Update locked collateral
        account.rawLocked = convertDebtTokensToYield(account.debt) * minimumCollateralization / FIXED_POINT_SCALAR;
```

## Impact Details

Unnecessary calculation with external call that costs additional gas.

## References

n/a

## Proof of Concept

## Proof of Concept

provided in the Impact details section
