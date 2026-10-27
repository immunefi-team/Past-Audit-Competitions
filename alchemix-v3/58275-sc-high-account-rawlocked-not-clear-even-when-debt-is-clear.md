# 58275 sc high account rawlocked not clear even when debt is clear

**Submitted on Oct 31st 2025 at 22:10:41 UTC by @emmac002 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58275
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Users lose funds

## Description

## Brief/Intro

A bug in the \_subDebt function causes borrower rawLocked not clear even after full debt repayment (account.debt == 0) when minimumCollateralization/convertDebtTokensToYield is increased post-borrow. This occurs because \_subDebt caps the toFree amount at \_totalLocked but still computes lockedCollateral using the new higher ratio, leaving a residual in account.rawLocked.

## Vulnerability Details

The vulnerability lies in the \_subDebt function, which is called to reduce a position’s debt. The function correctly caps the amount of collateral to free (toFree) at the total locked amount (\_totalLocked), but fails to adjust account.rawLocked when the required locked collateral exceeds \_totalLocked due to a higher minimumCollateralizatio/convertDebtTokensToYield.

```
function _addDebt(uint256 tokenId, uint256 amount) internal {
    Account storage account = _accounts[tokenId];

    // Update collateral variables
    uint256 toLock = convertDebtTokensToYield(amount) * minimumCollateralization / FIXED_POINT_SCALAR;
    uint256 lockedCollateral = convertDebtTokensToYield(account.debt) * minimumCollateralization / FIXED_POINT_SCALAR;

    if (account.collateralBalance - lockedCollateral < toLock) revert Undercollateralized();

    account.rawLocked = lockedCollateral + toLock;
    _totalLocked += toLock;
    account.debt += amount;
    totalDebt += amount;
}
```

1. User addDebt(100) and the minimumCollateralization = 1.5, toLock = 100 \* 1.5 = 150, lockedCollateral = 0, account.rawLocked = 150, \_totalLocked = 150, totalDebt = 100, account.debt = 100

```
function _subDebt(uint256 tokenId, uint256 amount) internal {
    Account storage account = _accounts[tokenId];

    // Update collateral variables
    uint256 toFree = convertDebtTokensToYield(amount) * minimumCollateralization / FIXED_POINT_SCALAR;
    uint256 lockedCollateral = convertDebtTokensToYield(account.debt) * minimumCollateralization / FIXED_POINT_SCALAR;

    // For cases when someone above minimum LTV gets liquidated.
@>   if (toFree > _totalLocked) {
@>       toFree = _totalLocked;
    }

    account.debt -= amount;
    totalDebt -= amount;
    _totalLocked -= toFree;
    account.rawLocked = lockedCollateral - toFree;
```

2. Admin increases minimumCollateralization or convertDebtTokensToYield increases, User subDebt(100) and the minimumCollateralization = 2, toFree = 100 \* 2 = 200, lockedCollateral = 100 \* 2 = 200, toFree (200) > \_totalLocked (150), so toFree = 150, account.rawLocked = 200 - 150 = 50, account.debt = 100 - 100 = 0

```
function _sync(uint256 tokenId) internal {
    Account storage account = _accounts[tokenId];

    // Collateral to remove from redemptions and fees
    uint256 collateralToRemove = PositionDecay.ScaleByWeightDelta(account.rawLocked, _collateralWeight - account.lastCollateralWeight);
    account.collateralBalance -= collateralToRemove;
```

3. If in the \_sync function \_collateralWeight is not 0, then collateralToRemove will be incorrectly deducted from account.collateralBalance, even when the user has cleared the debt.

## Impact Details

Any borrower may lose funds if account.rawLocked is not properly cleared

## Solution

```
+       uint256 toFreeV2;
        if (toFree > _totalLocked) {
-            toFree = _totalLocked;
+.           toFreeV2= _totalLocked;
        }

        account.debt -= amount;
        totalDebt -= amount;
-        _totalLocked -= toFree;
+.       _totalLocked -= toFreeV2;
        account.rawLocked = lockedCollateral - toFree;
```

## References

(<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi#L936-L947>)

## Proof of Concept

## Proof of Concept

```
In _subDebt function:

account.debt -= amount;
account.rawLocked = lockedCollateral - toFree;
console.log("_subDebt account.debt:", account.debt);
console.log("_subDebt account.rawLocked:", account.rawLocked);
```

Test:

```
function testRawLockedNotClear() external {
    uint256 amount = 100e18;
    uint256 minimumCollateralization = alchemist.minimumCollateralization();
    console.log("original minimumCollateralization:", minimumCollateralization);

    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
    alchemist.deposit(amount, address(0xbeef), 0);
    console.log("After Deposit:", vault.balanceOf(address(0xbeef)));
    // a single position nft would have been minted to 0xbeef
    uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    alchemist.mint(tokenId, amount / 2, address(0xbeef));

    vm.roll(block.number + 1);

    vm.startPrank(address(0xdead));
    alchemist.setMinimumCollateralization(minimumCollateralization+FIXED_POINT_SCALAR);
    minimumCollateralization = alchemist.minimumCollateralization();

    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(alToken), address(alchemist), amount / 2);
    alchemist.repay(amount / 2, tokenId);
    vm.stopPrank();
    (, uint256 userDebt,) = alchemist.getCDP(tokenId);
    assertEq(userDebt, 0);
}
```

Output:

```
  _subDebt account.debt: 0
  _subDebt account.rawLocked: 50000000000000000000
```
