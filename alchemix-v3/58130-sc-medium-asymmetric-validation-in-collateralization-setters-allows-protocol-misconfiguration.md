# 58130 sc medium asymmetric validation in collateralization setters allows protocol misconfiguration breaking all borrowing

**Submitted on Oct 30th 2025 at 20:56:19 UTC by @InAllHonesty for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58130
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Intro

The `AlchemistV3::setMinimumCollateralization()` function lacks validation to ensure it remains less than or equal to globalMinimumCollateralization, while `AlchemistV3::setGlobalMinimumCollateralization()` enforces the reverse relationship. This asymmetric validation allows an admin to configure `minimumCollateralization > globalMinimumCollateralization`, breaking the protocol's core invariant and rendering the borrowing system inoperable for all users.

## Vulnerability Details

The AlchemistV3 protocol maintains a critical invariant across three collateralization thresholds: `collateralizationLowerBound ≤ minimumCollateralization ≤ globalMinimumCollateralization`

The protocol correctly enforces this invariant in two places:

```
function setCollateralizationLowerBound(uint256 value) external onlyAdmin {
    _checkArgument(value <= minimumCollateralization); // @audit enforces lower <= minimum
    _checkArgument(value >= FIXED_POINT_SCALAR);
    collateralizationLowerBound = value;
    emit CollateralizationLowerBoundUpdated(value);
}
```

```
function setGlobalMinimumCollateralization(uint256 value) external onlyAdmin {
    _checkArgument(value >= minimumCollateralization); // @audit enforces global >= minimum
    globalMinimumCollateralization = value;
    emit GlobalMinimumCollateralizationUpdated(value);
}
```

```
function setMinimumCollateralization(uint256 value) external onlyAdmin {
    _checkArgument(value >= FIXED_POINT_SCALAR);
    //@audit MISSING: _checkArgument(value <= globalMinimumCollateralization);
    minimumCollateralization = value;
    emit MinimumCollateralizationUpdated(value);
}
```

This creates an asymmetric validation vulnerability where the invariant can be violated.

## Impact on Core Protocol Functions

Once misconfigured, multiple critical functions break, depending on the configuration one or more of the following applies:

`AlchemistV3::getMaxBorrowable` - Artificially Restricts Borrowing

`AlchemistV3::_validate` - Rejects Valid Positions

`AlchemistV3::_addDebt` - Blocks All New Borrows

`AlchemistV3::calculateLiquidation`

All users maintaining collateralization between `globalMinimumCollateralization` and the inflated `minimumCollateralization` cannot borrow.

Users with higher collateralization see their borrowing capacity arbitrarily reduced based on the inflated parameter.

## Proof of Concept

Add the following test inside `AlchemistV3.t.sol`

```solidity
    function testSetMinimumCollateralization_Invalid_Above_GlobalMinimum() external {
        vm.startPrank(alOwner);
        alchemist.setGlobalMinimumCollateralization(12e17);
        alchemist.setMinimumCollateralization(20e17); // minimum collateralization should always be lower than global minimum collateralization
        assertTrue(alchemist.minimumCollateralization() > alchemist.globalMinimumCollateralization());
        vm.stopPrank();
    }
```
