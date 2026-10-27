# 57678 sc high liquidation fee is deducted from user but not paid to liquidator

**Submitted on Oct 28th 2025 at 05:18:12 UTC by @Anirruth for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57678
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

When liquidating a position, the contract deducts the gross seizure (debt burn + base fee) from the user’s collateral. It then sends only the net amount (gross − fee) to the transmuter and attempts to pay the fee to the liquidator. If the account’s collateral is fully drained by the gross deduction (or the leftover is less than the fee), the fee transfer never occurs. The result is a fee that is charged to the user but not paid to the liquidator and not sent to the transmuter, effectively being withheld in the contract.

## Vulnerability Details

In \_doLiquidation, the contract:

1. Deducts the full gross seizure from the account’s collateralBalance.
2. Transfers only the net amount (gross − fee) to the transmuter.
3. Pays the base fee to the liquidator only if the account’s remaining balance is still at least the fee.

```
 // update user balance and debt
        account.collateralBalance = account.collateralBalance > amountLiquidated ? account.collateralBalance - amountLiquidated : 0;
        _subDebt(accountId, debtToBurn);

        // send liquidation amount - fee to transmuter
        TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);

        // send base fee to liquidator if available
        if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
        }
```

* amountLiquidated equals the gross seizure in yield units , so line 871 reduces the account by gross. <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi#L871>
* The net (gross − fee) is sent to the transmuter (<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi#L875>).
* If the gross deduction drains the account (or leaves a leftover smaller than the fee), the conditional on line 878 fails and the liquidator receives no fee, even though that fee has already been accounted for in the gross deduction from the user. <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi#L878>

## Impact Details

Liquidators perform valid liquidations but do not receive the fee when the account is liquidated and the fee will stay in the contract.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi#L871-L880>

## Proof of Concept

## Proof of Concept

Paste the test in AlchemistV3.t.sol and run the test using `forge test -vvvv --match-path src/test/AlchemistV3.t.sol --match-test testLiquidate_FeeNotPaid_When_GrossEqualsBalance`

```solidity
function testLiquidate_FeeNotPaid_When_GrossEqualsBalance() external {
        // 0) Configure to force liquidation in normal branch and guarantee fee > 0
        vm.startPrank(alchemist.admin());
        alchemist.setDepositCap(type(uint256).max);
        // Make the liquidation trigger easy: set lower bound equal to the minimum ratio
        alchemist.setCollateralizationLowerBound(alchemist.minimumCollateralization());
        // Start with a high fee so baseFee > 0 and likely larger than leftover
        alchemist.setLiquidatorFee(9000); // 90%
        vm.stopPrank();
    
        // 1) Keep system globally healthy
        deal(address(vault), yetAnotherExternalUser, depositAmount * 5);
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 5);
        alchemist.deposit(depositAmount * 5, yetAnotherExternalUser, 0);
        vm.stopPrank();
    
        // 2) Borrower mints at the exact LTV limit (so any price drop undercollateralizes)
        deal(address(vault), address(0xbeef), depositAmount);
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        // Mint exactly to the min collateralization limit
        alchemist.mint(
            tokenId,
            alchemist.totalValue(tokenId) * FIXED_POINT_SCALAR / alchemist.minimumCollateralization(),
            address(0xbeef)
        );
        vm.stopPrank();
    
        // 3) Apply a small price drop to ensure undercollateralized vs the (now-equal) lower bound
        uint256 initSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initSupply);
        // +8% supply (~7.4% price drop) is enough because we minted right at the bound
        uint256 newSupply = initSupply + (initSupply * 800 / 10_000);
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(newSupply);
    
        // 4) Compute expected liquidation; enforce base fee > 0 and the fee gate leftover < fee
        (uint256 prevCollateral, uint256 prevDebt,) = alchemist.getCDP(tokenId);
        require(prevCollateral > 0 && prevDebt > 0, "invalid pre-state");
    
        uint256 curCollGlobal =
            alchemist.normalizeUnderlyingTokensToDebt(alchemist.getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / alchemist.totalDebt();
        (uint256 liqAmountDebt,, uint256 baseFeeDebt,) = alchemist.calculateLiquidation(
            alchemist.totalValue(tokenId),
            prevDebt,
            alchemist.minimumCollateralization(),
            curCollGlobal,
            alchemist.globalMinimumCollateralization(),
            alchemist.liquidatorFee()
        );
        // If fee computed as zero due to current fee setting, push to 99% once
        if (baseFeeDebt == 0) {
            vm.startPrank(alchemist.admin());
            alchemist.setLiquidatorFee(9900);
            vm.stopPrank();
            (liqAmountDebt,, baseFeeDebt,) = alchemist.calculateLiquidation(
                alchemist.totalValue(tokenId),
                prevDebt,
                alchemist.minimumCollateralization(),
                curCollGlobal,
                alchemist.globalMinimumCollateralization(),
                alchemist.liquidatorFee()
            );
            require(baseFeeDebt > 0, "expected base fee > 0");
        }
        uint256 liqYield = alchemist.convertDebtTokensToYield(liqAmountDebt);
        uint256 feeYieldExpected = alchemist.convertDebtTokensToYield(baseFeeDebt);
    
        // Strict fee-gate: force leftover < fee (otherwise bump fee to 99% and a tad more drop)
        if (!(prevCollateral > liqYield && (prevCollateral - liqYield) < feeYieldExpected)) {
            vm.startPrank(alchemist.admin());
            alchemist.setLiquidatorFee(9900);
            vm.stopPrank();
            // Tiny additional drop (+2%) to tighten leftover
            uint256 tighterSupply = newSupply + (initSupply * 200 / 10_000);
            IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(tighterSupply);
    
            curCollGlobal =
                alchemist.normalizeUnderlyingTokensToDebt(alchemist.getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / alchemist.totalDebt();
            (liqAmountDebt,, baseFeeDebt,) = alchemist.calculateLiquidation(
                alchemist.totalValue(tokenId),
                prevDebt,
                alchemist.minimumCollateralization(),
                curCollGlobal,
                alchemist.globalMinimumCollateralization(),
                alchemist.liquidatorFee()
            );
            require(baseFeeDebt > 0, "fee still zero");
            liqYield = alchemist.convertDebtTokensToYield(liqAmountDebt);
            feeYieldExpected = alchemist.convertDebtTokensToYield(baseFeeDebt);
            require(prevCollateral > liqYield && (prevCollateral - liqYield) < feeYieldExpected, "strict gate not satisfied");
        }
    
        // 5) Balances pre-liquidation
        uint256 transmuterPrev = IERC20(address(vault)).balanceOf(address(transmuterLogic));
        uint256 alchemistPrev  = IERC20(address(vault)).balanceOf(address(alchemist));
        vm.startPrank(externalUser);
        uint256 liqPrevVault   = IERC20(address(vault)).balanceOf(externalUser);
        uint256 liqPrevUnd     = IERC20(vault.asset()).balanceOf(externalUser);
    
        // 6) Liquidate (must be undercollateralized and normal branch)
        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenId);
        vm.stopPrank();
    
        // 7) Post-state
        (uint256 collAfter,,) = alchemist.getCDP(tokenId);
        uint256 transmuterPost = IERC20(address(vault)).balanceOf(address(transmuterLogic));
        uint256 alchemistPost  = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 liqPostVault   = IERC20(address(vault)).balanceOf(externalUser);
        uint256 liqPostUnd     = IERC20(vault.asset()).balanceOf(externalUser);
    
        // 8) Strict assertions proving fee withheld bug:
        // Normal branch: no underlying fee
        vm.assertEq(feeInUnderlying, 0);
        // fee computed (>0)
        vm.assertTrue(feeInYield > 0);
        // collateral reduced by gross (within tolerance)
        vm.assertApproxEqAbs(prevCollateral > collAfter ? (prevCollateral - collAfter) : 0, assets, minimumDepositOrWithdrawalLoss);
        // fee not paid to liquidator (gate failure)
        vm.assertEq(liqPostVault, liqPrevVault);
        vm.assertEq(liqPostUnd, liqPrevUnd);
        // transmuter received only net (assets - fee)
        vm.assertApproxEqAbs(transmuterPost - transmuterPrev, assets - feeInYield, 1e18);
        // contract only sent net; fee remained in contract
        vm.assertApproxEqAbs(alchemistPrev - alchemistPost, assets - feeInYield, minimumDepositOrWithdrawalLoss);
    }
```
