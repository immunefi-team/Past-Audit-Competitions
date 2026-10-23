# 57972 sc high liquidation doesn t update mytsharesdeposited

**Submitted on Oct 29th 2025 at 17:58:27 UTC by @OxPhantom for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57972
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hour
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

During liquidation, MYT is transferred from the Alchemist to the Transmuter, but `_mytSharesDeposited` is not decremented. This inflates the Alchemist TVL used elsewhere, double‑counts assets with the Transmuter, produces an overly lenient bad‑debt ratio, and can prematurely DoS deposits via the `depositCap` check.

## Vulnerability Details

In the liquidation flow, MYT collateral is sent to the Transmuter without reducing `_mytSharesDeposited`:

```solidity
// _doLiquidation(...)
amountLiquidated = convertDebtTokensToYield(liquidationAmount);
feeInYield = convertDebtTokensToYield(baseFee);

// update user balance and debt
account.collateralBalance = account.collateralBalance > amountLiquidated
    ? account.collateralBalance - amountLiquidated : 0;
_subDebt(accountId, debtToBurn);

// send liquidation amount - fee to transmuter (no _mytSharesDeposited update here)
TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);
```

neither in the \_forceRepay flow:

```solidity
        uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
        account.earmarked -= earmarkToRemove;

        creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield;
        account.collateralBalance -= creditToYield;

        uint256 protocolFeeTotal = creditToYield * protocolFee / BPS;

```

Yet, Alchemist TVL is derived from `_mytSharesDeposited`:

```solidity
// TVL = convertYieldTokensToUnderlying(_mytSharesDeposited)
function _getTotalUnderlyingValue() internal view returns (uint256 totalUnderlyingValue) {
    uint256 yieldTokenTVLInUnderlying = convertYieldTokensToUnderlying(_mytSharesDeposited);
    totalUnderlyingValue = yieldTokenTVLInUnderlying;
}
```

Deposits rely on `_mytSharesDeposited` to enforce `depositCap`:

```solidity
// deposit() guard
_checkState(_mytSharesDeposited + amount <= depositCap);
...
_mytSharesDeposited += amount;
```

Meanwhile, Transmuter’s bad‑debt ratio counts both Alchemist TVL and Transmuter‑held MYT, so if `_mytSharesDeposited` isn’t reduced, the same units are double‑counted in the denominator:

```solidity
// Transmuter badDebtRatio
uint256 yieldTokenBalance = TokenUtils.safeBalanceOf(alchemist.myt(), address(this));
uint256 denominator = alchemist.getTotalUnderlyingValue()
  + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance);
uint256 badDebtRatio = alchemist.totalSyntheticsIssued()
  * 10**TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator;
```

Contrast: in redemption flows the contract does decrement `_mytSharesDeposited` after transferring MYT out, so liquidation behavior is inconsistent with redemption.

## Impact Details

* Overly lenient bad‑debt ratio: inflated denominator (double‑counted TVL) reduces haircuts in Transmuter redemptions, overpaying claimants in stressed conditions.
* Deposit DoS: `_mytSharesDeposited` remains artificially high; `deposit()` can revert prematurely on `_mytSharesDeposited + amount <= depositCap`, blocking new deposits.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L871-L880> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L369> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/Transmuter.sol#L219-L226> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L760-L780>

## Proof of Concept

## Proof of Concept

This POC demonstrate the issue after a liquidation yield token are transfered to the transmuter however the total underlying value doesn't change.

```solidity
        function testLiquidate_POC_TotalUnderlyingValue() external {
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        // 1. Create a healthy account with no debt, but enough collateral to cover shortfall
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        uint256 tokenIdHealthy = AlchemistNFTHelper.getFirstTokenId(yetAnotherExternalUser, address(alchemistNFT));
        vm.stopPrank();

        // 2. Create the undercollateralized account
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        uint256 tokenIdBad = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        // Mint so that debt is just below collateral
        alchemist.mint(tokenIdBad, alchemist.totalValue(tokenIdBad) * FIXED_POINT_SCALAR / minimumCollateralization, address(0xbeef));
        vm.stopPrank();

        // 3. Drop price so that account debt > account collateral, but system collateral is still enough

        // Drop price so that bad account's collateral is less than its debt, but system collateral is still enough
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // Drop price by 70%
        uint256 modifiedVaultSupply = (initialVaultSupply * 7000 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

       uint256 totalValueBefore= alchemist.getTotalUnderlyingValue();
       uint256 balanceOfBefore= vault.balanceOf(address(alchemist));
       console.log("TotalUnderlyingValue before: ",alchemist.getTotalUnderlyingValue());
       console.log("balanceOf before: ",vault.balanceOf(address(alchemist)));
        // 4. Liquidate the undercollateralized account
        vm.startPrank(externalUser);
        alchemist.liquidate(tokenIdBad);
        vm.stopPrank();
           console.log("balanceOf after: ",vault.balanceOf(address(alchemist)));

           console.log("TotalUnderlyingValue after: ",alchemist.getTotalUnderlyingValue());
           //total value is the same
           assertEq(alchemist.getTotalUnderlyingValue(), totalValueBefore);
           // the balance decreased
           assertLt(vault.balanceOf(address(alchemist)), balanceOfBefore);
    }

```
