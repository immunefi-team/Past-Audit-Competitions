# #41873 \[SC-Insight] Protocol fee loss due to incorrect fee calculation in MoneyBrinter.sol

**Submitted on Mar 19th 2025 at 03:16:10 UTC by @OxSimao for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41873
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/MoneyBrinter.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

`MoneyBrinter.sol` incorrectly undercharges borrowing fee due to a math error.

## Vulnerability Details

`MoneyBrinter.sol` implements `previewWithdraw()` and `previewRedeem()` and calculates the result including the exit fee, `exitFeeBasisPoints`. However, this calculation is incorrect.

Suppose there are 1000 assets and shares, user wants to withdraw 100 assets and calls `withdraw()`, internally calling `previewWithdraw()` to calculate the fee. `feeBasisPoints` is 1000 (`10%`) and `_BASIS_POINT_SCALE` is 1e4.

```solidity
uint256 fee = assets * feeBasisPoints / _BASIS_POINT_SCALE = 100 * 1000 / 10000 = 10;
return super.previewWithdraw(assets + fee) = super.previewWithdraw(100 + 10) = 110 * totalSupply / totalAssets = 110 * 1000 / 1000 = 110;
```

So, calling `previewWithdraw()` with 100 assets returns 110 shares. However, this fee is smaller than `feeBasisPoints = 10%`, as 110 shares should give `110 * 0.9 = 99` shares, which is 99 assets also given same total supply and total assets. The actual fee applied is:

1. 110 shares are worth 110 assets;
2. out of 110 assets, 100 assets are withdrawn
3. `1 - 100/110 = 0.091`, so the fee is actually `9.1%`. This represents a `0.9%` fee loss.

Issue is similar for `previewRedeem()`.

```solidity
function previewWithdraw(uint256 assets) public view override returns (uint256) {
    uint256 fee = _feeOnRaw(assets, exitFeeBasisPoints);
    return super.previewWithdraw(assets + fee);
}

function previewRedeem(uint256 shares) public view override returns (uint256) {
    uint256 assets = super.previewRedeem(shares);
    return assets - _feeOnTotal(assets, exitFeeBasisPoints);
}
```

## Impact Details

Significant fee loss, if fee is `10%`, actual fee is `9.1%`, showing a `0.9%` fee loss.

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/MoneyBrinter.sol?utm\\_source=immunefi#L142-L150>

## Proof of Concept

## Proof of Concept

Described in the report above but it boils down to:

1. User calls `MoneyBrinter::withdraw()`, but instead of paying an exit fee of `10%`, pays `9.1%`.
