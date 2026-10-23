# 56817 sc high forcerepay doesn t decrement mytsharesdeposited inflating tvl

**Submitted on Oct 20th 2025 at 22:30:52 UTC by @Cryptor for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56817
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Protocol insolvency

## Description

## Brief/Intro

ForceRepay doesn't decrement \_mytSharesDeposited, inflating TVL

## Vulnerability Details

The \_mytSharesDeposited variable is a critical accounting metric in the AlchemistV3 contract, designed to track the total amount of myt (yield-bearing tokens) held by the contract. This value is the sole input for the \_getTotalUnderlyingValue() function, which calculates the protocol's Total Value Locked (TVL).

Several functions that transfer myt tokens out of the contract fail to decrement \_mytSharesDeposited, leading to a persistent discrepancy between the accounted TVL and the actual on-chain balance. The primary functions affected are:

\_forceRepay(): When repaying earmarked debt during a liquidation, this function transfers creditToYield to the transmuter and protocolFeeTotal to the protocol fee receiver. However, it does not decrease \_mytSharesDeposited to reflect these outflows. \_doLiquidation(): This function transfers amountLiquidated to the transmuter and feeInYield to the liquidator, but \_mytSharesDeposited is not updated. \_resolveRepaymentFee(): When a repayment fee is paid to a liquidator, the fee amount is transferred out, but again, \_mytSharesDeposited is not decremented.

Example:

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L852-L895>

```solidity
  function _doLiquidation(uint256 accountId, uint256 collateralInUnderlying, uint256 repaidAmountInYield)
        internal
        returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying)
    {
        Account storage account = _accounts[accountId];

        (uint256 liquidationAmount, uint256 debtToBurn, uint256 baseFee, uint256 outsourcedFee) = calculateLiquidation(
            collateralInUnderlying,
            account.debt,
            minimumCollateralization,
            normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt,
            globalMinimumCollateralization,
            liquidatorFee
        );

        amountLiquidated = convertDebtTokensToYield(liquidationAmount);
        feeInYield = convertDebtTokensToYield(baseFee);

        // update user balance and debt
        account.collateralBalance = account.collateralBalance > amountLiquidated ? account.collateralBalance - amountLiquidated : 0;
        _subDebt(accountId, debtToBurn);

        // send liquidation amount - fee to transmuter
        TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);

        // send base fee to liquidator if available
        if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
        }

        // Handle outsourced fee from vault
        if (outsourcedFee > 0) {
            uint256 vaultBalance = IFeeVault(alchemistFeeVault).totalDeposits();
            if (vaultBalance > 0) {
                uint256 feeBonus = normalizeDebtTokensToUnderlying(outsourcedFee);
                feeInUnderlying = vaultBalance > feeBonus ? feeBonus : vaultBalance;
                IFeeVault(alchemistFeeVault).withdraw(msg.sender, feeInUnderlying);
            }
        }

        emit Liquidated(accountId, msg.sender, amountLiquidated + repaidAmountInYield, feeInYield, feeInUnderlying);
        return (amountLiquidated + repaidAmountInYield, feeInYield, feeInUnderlying);
    }

```

This contrasts with other functions like withdraw(), burn(), and repay(), which correctly update the metric, creating an accounting inconsistency that is triggered specifically during liquidations and forced repayments.

Example:

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L391-L415>

```solidity
unction withdraw(uint256 amount, address recipient, uint256 tokenId) external returns (uint256) {
        _checkArgument(recipient != address(0));
        _checkForValidAccountId(tokenId);
        _checkArgument(amount > 0);
        _checkAccountOwnership(IAlchemistV3Position(alchemistPositionNFT).ownerOf(tokenId), msg.sender);
        _earmark();

        _sync(tokenId);

        uint256 lockedCollateral = convertDebtTokensToYield(_accounts[tokenId].debt) * minimumCollateralization / FIXED_POINT_SCALAR;
        _checkArgument(_accounts[tokenId].collateralBalance - lockedCollateral >= amount);

        _accounts[tokenId].collateralBalance -= amount;

        // Assure that the collateralization invariant is still held.
        _validate(tokenId);

        // Transfer the yield tokens to msg.sender
        TokenUtils.safeTransfer(myt, recipient, amount);
        _mytSharesDeposited -= amount;

        emit Withdraw(amount, tokenId, recipient);

        return amount;
    }
```

## Impact Details

The primary and most severe impact is the under-enforcement of liquidations, leading to an increased risk of bad debt for the protocol.

The liquidation mechanism relies on the global system health, represented by alchemistCurrentCollateralization, which is derived directly from the inflated TVL. The calculateLiquidation function has a critical branch: if the system is globally undercollateralized (alchemistCurrentCollateralization < alchemistMinimumCollateralization), it enforces a full liquidation of the unhealthy position to protect the protocol.

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

add the following test to src/test/AlchemistV3.t.sol

```
function test_TVLInflated_AfterForceRepay_MissingMytSharesDecrement() external {
        // Ensure protocol fee is zero to isolate the TVL accounting surface.
        vm.prank(alOwner);
        alchemist.setProtocolFee(0);

        // 1) Seed pooled collateral so the repayment fee can be paid from contract balance.
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        // 2) Victim: deposit and mint max LTV.
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        uint256 victimTokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        uint256 victimMint = alchemist.totalValue(victimTokenId) * FIXED_POINT_SCALAR / alchemist.minimumCollateralization();
        alchemist.mint(victimTokenId, victimMint, address(0xbeef));
        vm.stopPrank();

        // 3) Fully earmark victim’s debt via redemption and mature it.
        deal(address(alToken), anotherExternalUser, victimMint);
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), victimMint);
        transmuterLogic.createRedemption(victimMint);
        vm.stopPrank();
        vm.roll(block.number + 5_256_000);

        // 4) Drop share price so required shares > victim collateral (forces repay-only path with a fee).
        (uint256 victimCollateralBefore,,) = alchemist.getCDP(victimTokenId);
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        uint256 modifiedVaultSupply = (initialVaultSupply * 2) + 1; // ~50% price drop
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);
        (, uint256 victimDebtBefore,) = alchemist.getCDP(victimTokenId);
        uint256 requiredShares = alchemist.convertDebtTokensToYield(victimDebtBefore);
        require(requiredShares > victimCollateralBefore, "pre: required shares should exceed victim collateral");

        // 5) Snapshot accounting vs on-chain before liquidation.
        uint256 sharesAccountingBefore = alchemist.getTotalDeposited(); // _mytSharesDeposited
        uint256 sharesOnChainBefore = IERC20(address(vault)).balanceOf(address(alchemist));
        // Use the same converter for apples-to-apples comparisons.
        uint256 underlyingOnChainBefore = alchemist.convertYieldTokensToUnderlying(sharesOnChainBefore);
        uint256 underlyingFromAccountingSharesBefore = alchemist.convertYieldTokensToUnderlying(sharesAccountingBefore);

        assertApproxEqAbs(sharesAccountingBefore, sharesOnChainBefore, 1, "pre: shares mismatch");
        assertApproxEqAbs(underlyingFromAccountingSharesBefore, underlyingOnChainBefore, 1, "pre: underlying (from shares) mismatch");

        // 6) Liquidate victim (repay-only path pays fee in yield from pooled collateral).
        vm.startPrank(externalUser);
        (uint256 assetsLiquidated, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(victimTokenId);
        vm.stopPrank();
        assertEq(feeInUnderlying, 0, "repay-only: no underlying fee");
        require(assetsLiquidated > 0, "repay-only: assetsLiquidated > 0");
        require(feeInYield > 0, "repay-only: repayment fee > 0");

        // 7) Post state and deltas.
        uint256 sharesAccountingAfter = alchemist.getTotalDeposited();
        uint256 sharesOnChainAfter = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 underlyingOnChainAfter = alchemist.convertYieldTokensToUnderlying(sharesOnChainAfter);
        uint256 underlyingFromAccountingSharesAfter = alchemist.convertYieldTokensToUnderlying(sharesAccountingAfter);

        uint256 onChainSharesDelta = sharesOnChainBefore - sharesOnChainAfter;
        uint256 accountingSharesDelta = sharesAccountingBefore - sharesAccountingAfter;

        // On-chain outflows = repayment + repayment fee.
        assertEq(onChainSharesDelta, assetsLiquidated + feeInYield, "on-chain shares drop = repayment + fee");

        // If fixed: accounting reflects both repayment and fee; if buggy: misses fee decrement.
        bool feeReflectedInAccounting = (accountingSharesDelta == onChainSharesDelta);

        if (feeReflectedInAccounting) {
            assertEq(accountingSharesDelta, assetsLiquidated + feeInYield, "fixed: accounting should include repayment + fee");
            assertApproxEqAbs(underlyingFromAccountingSharesAfter, underlyingOnChainAfter, 1, "fixed: no TVL inflation (from shares)");
        } else {
            // Bug path: fee not deducted from _mytSharesDeposited.
            assertEq(onChainSharesDelta - accountingSharesDelta, feeInYield, "bug: accounting misses fee");
            uint256 onChainUnderlyingDelta = underlyingOnChainBefore - underlyingOnChainAfter;
            uint256 accountingUnderlyingDelta = underlyingFromAccountingSharesBefore - underlyingFromAccountingSharesAfter;

            assertApproxEqAbs(
                onChainUnderlyingDelta,
                alchemist.convertYieldTokensToUnderlying(assetsLiquidated + feeInYield),
                1,
                "bug: on-chain underlying delta == convert(repayment + fee)"
            );
            assertApproxEqAbs(
                accountingUnderlyingDelta + alchemist.convertYieldTokensToUnderlying(feeInYield),
                alchemist.convertYieldTokensToUnderlying(assetsLiquidated + feeInYield),
                1,
                "bug: accounting underlying + fee gap should match on-chain delta"
            );

            // Collateralization inflated due to overstated TVL.
            uint256 totalDebtAfter = alchemist.totalDebt();
            require(totalDebtAfter > 0, "pre: totalDebtAfter > 0");
            uint256 collInflated = alchemist.normalizeUnderlyingTokensToDebt(alchemist.getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebtAfter;
            uint256 collReal = alchemist.normalizeUnderlyingTokensToDebt(underlyingOnChainAfter) * FIXED_POINT_SCALAR / totalDebtAfter;
            assertTrue(collInflated > collReal, "bug: inflated global collateralization");
        }
    }
```
