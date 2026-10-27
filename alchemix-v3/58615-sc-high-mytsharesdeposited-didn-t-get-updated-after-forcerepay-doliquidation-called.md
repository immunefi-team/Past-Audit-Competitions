# 58615 sc high mytsharesdeposited didn t get updated after forcerepay doliquidation called

**Submitted on Nov 3rd 2025 at 15:32:38 UTC by @TianYu4n for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58615
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

In AlchemistV3.sol, the `_forceRepay` and `_doLiquidation` transfer MYT shares out of the contract (to the transmuter, liquidator, or protocol fee vault) but never decrement `_mytSharesDeposited`. Consequently, the internal ledger continues to assume those shares are still held. This drifts collateral accounting—global collateralization metrics are overstated, so liquidations can under-react—and deposit-cap enforcement rejects new deposits even after MYT has actually left the vault.

## Vulnerability Details

Within `_forceRepay` and `_doLiquidation`, the contract transfers MYT shares to external recipients(typically the transmuter, liquidator, or protocol fee vault), but `_mytSharesDeposited` is never reduced by the amount that actually left the contract. Subsequent logic such as `_getTotalUnderlyingValue()`, `calculateLiquidation`, and `depositCap` checks rely on `_mytSharesDeposited` as the canonical measure of MYT held by AlchemistV3.

Because the counter is not updated after these transfers, every invocation of `_forceRepay` or `_doLiquidation` causes the internal ledger to drift further from the real balance. This bug triggers on every liquidation or forced repayment, independent of the caller, so it is both easy to reach and repeatedly exploitable.

## Impact Details

1. **Collateral accounting drift** – Global metrics (e.g., collateralization ratios) are overstated, enabling inaccurate positions to avoid or delay liquidation, thereby increasing insolvency risk.
2. **DepositCap blockage** – The contract continues to believe the cap is saturated even after MYT shares have been removed, so new deposits are rejected, blocking legitimate users and impairing protocol operations.

## Recommendation

In both \_forceRepay and \_doLiquidation, subtract the exact MYT amount that leaves the contract (principal and any fees) from \_mytSharesDeposited, so the internal ledger stays aligned with the actual balance.

```solidity
function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
        ......
        ......

        emit ForceRepay(accountId, amount, creditToYield, protocolFeeTotal);

        uint256 protocolFeePaid = 0;
        if (account.collateralBalance > protocolFeeTotal) {
            account.collateralBalance -= protocolFeeTotal;
            // Transfer the protocol fee to the protocol fee receiver
            TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
            protocolFeePaid = protocolFeeTotal;
        }

        uint256 mytOut = protocolFeePaid;
        if (creditToYield > 0) {
            // Transfer the repaid tokens from the account to the transmuter.
            TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
            mytOut += creditToYield;
        }

        if (mytOut != 0) {
            _mytSharesDeposited -= mytOut;
        }
        return creditToYield;
    }

function _doLiquidation(uint256 accountId, uint256 collateralInUnderlying, uint256 repaidAmountInYield)
        internal
        returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying)
    {
        ......
        ......
        // update user balance and debt
        account.collateralBalance = account.collateralBalance > amountLiquidated ? account.collateralBalance - amountLiquidated : 0;
        _subDebt(accountId, debtToBurn);

        // send base fee to liquidator if available
        uint256 feePaidInYield = 0;
        if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
            feePaidInYield = feeInYield;
        }

        // send liquidation amount minus paid fee to transmuter
        uint256 transmuterPayout = amountLiquidated - feePaidInYield;
        if (transmuterPayout > 0) {
            TokenUtils.safeTransfer(myt, transmuter, transmuterPayout);
        }

        uint256 mytOut = transmuterPayout + feePaidInYield;
        if (mytOut != 0) {
            _mytSharesDeposited -= mytOut;
        }

        // Handle outsourced fee from vault
        if (outsourcedFee > 0) 
        .......
        .......
    }
```

## Proof of Concept

## Proof of Concept

```solidity
function testLiquidationReleasesDepositCapHeadroom() external {
        uint256 amount = 100e18;
        address user = address(0xbeef);
        address liquidator = address(0xface);
        address newDepositor = anotherExternalUser;

        uint256 baselineDeposits = alchemist.getTotalDeposited();
        vm.prank(alOwner);
        alchemist.setDepositCap(baselineDeposits + amount);

        vm.startPrank(user);
        SafeERC20.safeApprove(address(vault), address(alchemist), amount);
        uint256 headroom = alchemist.depositCap() - baselineDeposits;
        alchemist.deposit(headroom, user, 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(user, address(alchemistNFT));
        uint256 maxBorrowable = alchemist.getMaxBorrowable(tokenId);
        alchemist.mint(tokenId, maxBorrowable, user);
        vm.stopPrank();

        // Lower the MYT share value sharply so the position falls below the liquidation threshold.
        uint256 initialSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialSupply * 2);

        uint256 totalDepositedBefore = alchemist.getTotalDeposited();
        (, uint256 debt,) = alchemist.getCDP(tokenId);
        uint256 collateralValue = alchemist.totalValue(tokenId);
        uint256 currentRatio = collateralValue * alchemist.FIXED_POINT_SCALAR() / debt;
        assertLt(currentRatio, alchemist.collateralizationLowerBound(), "position undercollateralized");

        vm.startPrank(liquidator);
        (uint256 yieldAmount,,) = alchemist.liquidate(tokenId);// here have call to _forcepay && _doLiquidation
        vm.stopPrank();

        assertGt(yieldAmount, 0, "liquidation executed");
        uint256 totalDepositedAfter = alchemist.getTotalDeposited();
        assertLt(totalDepositedAfter, totalDepositedBefore, "collateral moved out");

        uint256 available = alchemist.depositCap() - totalDepositedAfter;
        assertGt(available, 0, "deposit cap headroom available");

        vm.startPrank(newDepositor);
        SafeERC20.safeApprove(address(vault), address(alchemist), available);
        bool depositReverted;
        uint256 totalDepositedBeforeRetry = alchemist.getTotalDeposited();
        try alchemist.deposit(available, newDepositor, 0) returns (uint256 debtValue) {
            depositReverted = false;
            assertGt(debtValue, 0, "deposit should mint debt value after fix");
        } catch (bytes memory revertData) {
            depositReverted = true;
            bytes4 selector = revertData.length >= 4 ? bytes4(revertData) : bytes4(0);
            assertEq(selector, IllegalState.selector, "unexpected revert reason");
        }
        vm.stopPrank();

        if (depositReverted) {
            assertLt(alchemist.getTotalDeposited(), alchemist.depositCap(), "deposit still blocked pre-fix");
        } else {
            assertEq(alchemist.getTotalDeposited(), totalDepositedBeforeRetry + available, "deposit should increase balance after fix");
            assertLe(alchemist.getTotalDeposited(), alchemist.depositCap(), "deposit cap exceeded after fix");
        }
    }
```
