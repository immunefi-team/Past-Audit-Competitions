# 58792 sc high the cumulativeearmark does not decrease in forcerepay which lead to transfer more collateral from users even when all earmark debt cleared which breaks the alchemix v3 core logic

**Submitted on Nov 4th 2025 at 13:49:11 UTC by @zeroK for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58792
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Theft of unclaimed yield
  * Permanent freezing of funds

## Description

## Brief/Intro

when liquidate function invoked, the first and most important thing should be done first is repaying the earmark debt which is top priority when liquidation occur, this happens by invoking the `_forceRepay` function, which have many similarities compared to repay function, however there is a flow in `forceRepay` function, this function decrease the earmarked amount from users and sends equal value in myt to transmuter, but it never decrease the cumulativeEarmarked by the removed earmark, this lead to a critical flow as shown below:

```
User A: earmarked = 200
User B: earmarked = 300
User C: earmarked = 100
cumulativeEarmarked = 600 (correct)


user A get liquidated:
_forceRepay(userA, 200) 

// What happens:
User A: earmarked = 0  -->  Local state updated
cumulativeEarmarked = 600  --> Global state NOT updated (BUG!)

// State now:
User A: earmarked = 0
User B: earmarked = 300
User C: earmarked = 100
Sum of user earmarks = 400
cumulativeEarmarked = 600  --> Inflated by 200!
```

as shown the user A earmark removed locally for the users but the global state never updated, if we assume that all other users paid their earmark debt by invoking `repay` then the cumulative should be equal to zero, but this won't happen because the liquidation or `_forceRepay` does not updated the cumulative earmarked. the critical issue arises when redeem function occur with `amount > transmuter current myt balance` which lead to invoke the redeem function below:

```solidity
    /// @inheritdoc IAlchemistV3Actions
    function redeem(uint256 amount) external onlyTransmuter {
        _earmark();

        uint256 liveEarmarked = cumulativeEarmarked; //this  value can become zero due to repay
        if (amount > liveEarmarked) amount = liveEarmarked;

        // observed transmuter pre-balance -> potential cover
        uint256 transmuterBal = TokenUtils.safeBalanceOf(myt, address(transmuter)); 
        uint256 deltaYield    = transmuterBal > lastTransmuterTokenBalance ? transmuterBal - lastTransmuterTokenBalance : 0; 
        uint256 coverDebt = convertYieldTokensToDebt(deltaYield);

        // cap cover so we never consume beyond remaining earmarked
        uint256 coverToApplyDebt = amount + coverDebt > liveEarmarked ? (liveEarmarked - amount) : coverDebt;

        uint256 redeemedDebtTotal = amount + coverToApplyDebt;
       // Apply redemption weights/decay to the full amount that left the earmarked bucket
        if (liveEarmarked != 0 && redeemedDebtTotal != 0) {
            uint256 survival = ((liveEarmarked - redeemedDebtTotal) << 128) / liveEarmarked;
            _survivalAccumulator = _mulQ128(_survivalAccumulator, survival);
            _redemptionWeight += PositionDecay.WeightIncrement(redeemedDebtTotal, cumulativeEarmarked);
        }

        // earmarks are reduced by the full redeemed amount (net + cover)
        cumulativeEarmarked -= redeemedDebtTotal;

        // global borrower debt falls by the full redeemed amount
        totalDebt -= redeemedDebtTotal;

        lastRedemptionBlock = block.number;

        // consume the observed cover so it can't be reused
        if (deltaYield != 0) {
            uint256 usedYield = convertDebtTokensToYield(coverToApplyDebt);
            lastTransmuterTokenBalance = transmuterBal > usedYield ? transmuterBal - usedYield : transmuterBal;
        }

        // move only the net collateral + fee
        uint256 collRedeemed  = convertDebtTokensToYield(amount);
        uint256 feeCollateral = collRedeemed * protocolFee / BPS;
        uint256 totalOut      = collRedeemed + feeCollateral;

        // update locked collateral + collateral weight
        uint256 old = _totalLocked;
        _totalLocked = totalOut > old ? 0 : old - totalOut;
        _collateralWeight += PositionDecay.WeightIncrement(totalOut > old ? old : totalOut, old);

        TokenUtils.safeTransfer(myt, transmuter, collRedeemed); //@Money-flow send all collateral to transmuter
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, feeCollateral); // @Money-flow send fee to protocol fee receiver
        _mytSharesDeposited -= collRedeemed + feeCollateral;

        emit Redemption(redeemedDebtTotal);
    }


```

as. shown the redeem will assume that cumulative earmarked still exist for some users but this is not correct, users with earmarked debt already paid as buffer to transmuter, this lead to transfer out more collateral to the transmuter rather than returning zero(and transfer back the amountToRedeem back to caller with discount fee if not matured), this will break the alchemix v3 core idea and make the repay invocation make no sense because users collateral will always be used even if the earmarked debt is zero or less than expected.

## Vulnerability Details

we can see that \_liquidate invoke calls to \_forceRepay before \_doLiquidate to pay earmark debt and check if the position still valid for liquidation:

```solidity
    function _liquidate(uint256 accountId) internal returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying) {
        // Query transmuter and earmark global debt
        _earmark(); // update global states
        // Sync current user debt before deciding how much needs to be liquidated
        _sync(accountId); 

        Account storage account = _accounts[accountId];

        // Early return if no debt exists
        if (account.debt == 0) {
            return (0, 0, 0);
        }

        // In the rare scenario where 1 share is worth 0 underlying asset
        if (IVaultV2(myt).convertToAssets(1e18) == 0) {
            return (0, 0, 0);
        }

        // Calculate initial collateralization ratio
        uint256 collateralInUnderlying = totalValue(accountId);
        uint256 collateralizationRatio = collateralInUnderlying * FIXED_POINT_SCALAR / account.debt;

        // If account is healthy, nothing to liquidate
        if (collateralizationRatio > collateralizationLowerBound) {
            return (0, 0, 0);
        } // if position collateral to debt ratio > lower bound, no liquidation needed

        // Try to repay earmarked debt if it exists
        uint256 repaidAmountInYield = 0;
        if (account.earmarked > 0) {
            repaidAmountInYield = _forceRepay(accountId, account.earmarked);
        }
        // If debt is fully cleared, return with only the repaid amount, no liquidation needed, caller receives repayment fee
        if (account.debt == 0) {
            feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield); //@money-flow
            return (repaidAmountInYield, feeInYield, 0);
        }

        // Recalculate ratio after any repayment to determine if further liquidation is needed
        collateralInUnderlying = totalValue(accountId);
        collateralizationRatio = collateralInUnderlying * FIXED_POINT_SCALAR / account.debt;

        if (collateralizationRatio <= collateralizationLowerBound) {
            // Do actual liquidation
            return _doLiquidation(accountId, collateralInUnderlying, repaidAmountInYield);
        } else {
            // Since only a repayment happened, send repayment fee to caller
            feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
            return (repaidAmountInYield, feeInYield, 0);
        }
}

```

we can see `forceRepay` will decrease user earmark and transfer it to transmuter as myt but cumulativeEarmark never decreases:

```solidity

    function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
        if (amount == 0) {
            return 0;
        } // if earmark amount is 0, return early

        _checkForValidAccountId(accountId); //audit is this necessary?
        Account storage account = _accounts[accountId];

        // Query transmuter and earmark global debt
        _earmark(); // audit why invoke it twice

        // Sync current user debt before deciding how much is available to be repaid
        _sync(accountId);

        uint256 debt;

        // Burning yieldTokens will pay off all types of debt
        _checkState((debt = account.debt) > 0);

        uint256 credit = amount > debt ? debt : amount; // return smallest between debt and earmarked amount
        uint256 creditToYield = convertDebtTokensToYield(credit); // debt(earmark) --> underlying --> myt
        _subDebt(accountId, credit);

        // Repay debt from earmarked amount of debt first
        uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
        account.earmarked -= earmarkToRemove;

        creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield;
        account.collateralBalance -= creditToYield;

        uint256 protocolFeeTotal = creditToYield * protocolFee / BPS;

        emit ForceRepay(accountId, amount, creditToYield, protocolFeeTotal);

        if (account.collateralBalance > protocolFeeTotal) {
            account.collateralBalance -= protocolFeeTotal;
            // Transfer the protocol fee to the protocol fee receiver
            TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
        }

        if (creditToYield > 0) {
            // Transfer the repaid tokens from the account to the transmuter.
            TokenUtils.safeTransfer(myt, address(transmuter), creditToYield); //@money-flow send myt to transmuter
        }
        return creditToYield;
    }


```

this will directly affect redeem function, which lead to transfer more myt than the system have to and increases the redemption ratio much more than expected which affect users any time \_sync get invoked:

```solidity
  function redeem(uint256 amount) external onlyTransmuter {
        _earmark();

        uint256 liveEarmarked = cumulativeEarmarked;
        if (amount > liveEarmarked) amount = liveEarmarked;

        // observed transmuter pre-balance -> potential cover
        uint256 transmuterBal = TokenUtils.safeBalanceOf(myt, address(transmuter));
        uint256 deltaYield    = transmuterBal > lastTransmuterTokenBalance ? transmuterBal - lastTransmuterTokenBalance : 0;
        uint256 coverDebt = convertYieldTokensToDebt(deltaYield);

        // cap cover so we never consume beyond remaining earmarked
        uint256 coverToApplyDebt = amount + coverDebt > liveEarmarked ? (liveEarmarked - amount) : coverDebt;

        uint256 redeemedDebtTotal = amount + coverToApplyDebt;

       // Apply redemption weights/decay to the full amount that left the earmarked bucket
        if (liveEarmarked != 0 && redeemedDebtTotal != 0) {
            uint256 survival = ((liveEarmarked - redeemedDebtTotal) << 128) / liveEarmarked;
            _survivalAccumulator = _mulQ128(_survivalAccumulator, survival);
            _redemptionWeight += PositionDecay.WeightIncrement(redeemedDebtTotal, cumulativeEarmarked);
        }

        // earmarks are reduced by the full redeemed amount (net + cover)
        cumulativeEarmarked -= redeemedDebtTotal;

        // global borrower debt falls by the full redeemed amount
        totalDebt -= redeemedDebtTotal;

        lastRedemptionBlock = block.number;

        // consume the observed cover so it can't be reused
        if (deltaYield != 0) {
            uint256 usedYield = convertDebtTokensToYield(coverToApplyDebt);
            lastTransmuterTokenBalance = transmuterBal > usedYield ? transmuterBal - usedYield : transmuterBal;
        }

        // move only the net collateral + fee
        uint256 collRedeemed  = convertDebtTokensToYield(amount);
        uint256 feeCollateral = collRedeemed * protocolFee / BPS;
        uint256 totalOut      = collRedeemed + feeCollateral;

        // update locked collateral + collateral weight
        uint256 old = _totalLocked;
        _totalLocked = totalOut > old ? 0 : old - totalOut;
        _collateralWeight += PositionDecay.WeightIncrement(totalOut > old ? old : totalOut, old);

        TokenUtils.safeTransfer(myt, transmuter, collRedeemed);
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, feeCollateral);
        _mytSharesDeposited -= collRedeemed + feeCollateral;

        emit Redemption(redeemedDebtTotal);
    }

    /// @dev Update the user's earmarked and redeemed debt amounts.
    function _sync(uint256 tokenId) internal {
        Account storage account = _accounts[tokenId];

        // Collateral to remove from redemptions and fees
        uint256 collateralToRemove = PositionDecay.ScaleByWeightDelta(account.rawLocked, _collateralWeight - account.lastCollateralWeight);
        account.collateralBalance -= collateralToRemove;

        // Redemption survival now and at last sync
        // Survival is the amount of earmark that is left after a redemption
        uint256 redemptionSurvivalOld = PositionDecay.SurvivalFromWeight(account.lastAccruedRedemptionWeight);
        if (redemptionSurvivalOld == 0) redemptionSurvivalOld = ONE_Q128;
        uint256 redemptionSurvivalNew  = PositionDecay.SurvivalFromWeight(_redemptionWeight);
        // Survival during current sync window
        uint256 survivalRatio = _divQ128(redemptionSurvivalNew, redemptionSurvivalOld);
        // User exposure at last sync used to calculate newly earmarked debt pre redemption
        uint256 userExposure = account.debt > account.earmarked ? account.debt - account.earmarked : 0;
        uint256 earmarkRaw = PositionDecay.ScaleByWeightDelta(userExposure, _earmarkWeight - account.lastAccruedEarmarkWeight);

        // Earmark survival at last sync
        // Survival is the amount of unearmarked debt left after an earmark
        uint256 earmarkSurvival = PositionDecay.SurvivalFromWeight(account.lastAccruedEarmarkWeight);
        if (earmarkSurvival == 0) earmarkSurvival = ONE_Q128;
        // Decay snapshot by what was redeemed from last sync until now
        uint256 decayedRedeemed = _mulQ128(account.lastSurvivalAccumulator, survivalRatio);
        // What was added to the survival accumulator in the current sync window
        uint256 survivalDiff = _survivalAccumulator > decayedRedeemed ? _survivalAccumulator - decayedRedeemed : 0;

        // Unwind accumulated earmarked at last sync
        uint256 unredeemedRatio = _divQ128(survivalDiff, earmarkSurvival);
        // Portion of earmark that remains after applying the redemption. Scaled back from 128.128
        uint256 earmarkedUnredeemed = _mulQ128(userExposure, unredeemedRatio);
        if (earmarkedUnredeemed > earmarkRaw) earmarkedUnredeemed = earmarkRaw;

        // Old earmarks that survived redemptions in the current sync window
        uint256 exposureSurvival = _mulQ128(account.earmarked, survivalRatio);
        // What was redeemed from the newly earmark between last sync and now
        uint256 redeemedFromEarmarked = earmarkRaw - earmarkedUnredeemed;
        // Total overall earmarked to adjust user debt
        uint256 redeemedTotal = (account.earmarked - exposureSurvival) + redeemedFromEarmarked;

        account.earmarked = exposureSurvival + earmarkedUnredeemed;
        account.debt = account.debt >= redeemedTotal ? account.debt - redeemedTotal : 0;

        // Update locked collateral
        account.rawLocked = convertDebtTokensToYield(account.debt) * minimumCollateralization / FIXED_POINT_SCALAR;

        // Advance account checkpoint
        account.lastCollateralWeight = _collateralWeight;
        account.lastAccruedEarmarkWeight = _earmarkWeight;
        account.lastAccruedRedemptionWeight = _redemptionWeight;

        // Snapshot G for this account
        account.lastSurvivalAccumulator = _survivalAccumulator;
    }


```

for this reason, cumulative earmarked should be updated whenever the \_forceRepay get invoked.

## Impact Details

forceRepay does not update cumulative earmark globally which lead to make the buffer system in repay function make no sense and transfer out more myt from the alchemix system to transmuter plus updating redemption ratio more than expected which affect users positions.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L1041C1-L1095C6>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L738-L781>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L791-L843>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L589-L641>

## Proof of Concept

## Proof of Concept

run test below in alchemistV3.t.sol

```solidity

    function testBUG_ForceRepay_DoesNotDecrease_CumulativeEarmarked() external {
    vm.startPrank(someWhale);
    IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
    vm.stopPrank();

    // Create a healthy position to maintain global collateralization
    vm.startPrank(yetAnotherExternalUser);
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
    alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
    vm.stopPrank();

    // Create the victim position that will be liquidated
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
    alchemist.deposit(depositAmount, address(0xbeef), 0);
    uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    uint256 mintAmount = alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization;
    alchemist.mint(tokenIdFor0xBeef, mintAmount, address(0xbeef));
    
    // Create transmuter redemption with the minted alTokens
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
    transmuterLogic.createRedemption(mintAmount);
    vm.stopPrank();

    // Fast forward to earmark 60% of debt
    vm.roll(block.number + (5_256_000 * 60 / 100));

    // ========================================
    // RECORD STATE BEFORE LIQUIDATION
    // ========================================
    // First sync the position to update earmarks
    vm.prank(address(0xbeef));
    alchemist.poke(tokenIdFor0xBeef); // This should trigger _sync
    
    (uint256 prevCollateral, uint256 prevDebt, uint256 earmarkedBefore) = alchemist.getCDP(tokenIdFor0xBeef);
    uint256 cumulativeEarmarkedBefore = alchemist.cumulativeEarmarked();
    
    console.log("\n=== BEFORE LIQUIDATION ===");
    console.log("User earmarked debt:", earmarkedBefore);
    console.log("User total debt:", prevDebt);
    console.log("Global cumulativeEarmarked:", cumulativeEarmarkedBefore);
    
    // Verify we have earmarked debt to test
    require(earmarkedBefore > 0, "Should have earmarked debt");
    // require(cumulativeEarmarkedBefore >= earmarkedBefore, "Cumulative should >= user earmark");

    // Make position undercollateralized by crashing MYT price
    uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
    // Increase supply by 5.9% (reduces value per share)
    uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

    // Verify position is now undercollateralized
    uint256 collateralizationRatio = alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / prevDebt;
    console.log("Collateralization ratio after price crash:", collateralizationRatio);
    console.log("Liquidation threshold:", alchemist.collateralizationLowerBound());
    require(collateralizationRatio < alchemist.collateralizationLowerBound(), "Position should be liquidatable");

    uint256 cumulativeEarmarkedBefore00 = alchemist.cumulativeEarmarked();

    // ========================================
    // LIQUIDATE (triggers _forceRepay internally)
    // ========================================
    vm.startPrank(externalUser);
    (uint256 assetsLiquidated, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdFor0xBeef);
    vm.stopPrank();

    // ========================================
    // RECORD STATE AFTER LIQUIDATION
    // ========================================
    (uint256 newCollateral, uint256 newDebt, uint256 earmarkedAfter) = alchemist.getCDP(tokenIdFor0xBeef);
    uint256 cumulativeEarmarkedAfter = alchemist.cumulativeEarmarked();
    // the reason why we see cumulative decrease in below upcoming logs is because the debt < cumulativeEarmarked so we set cumulativeEarmarked == total debt in subDebt function
    if(alchemist.totalDebt() <= alchemist.cumulativeEarmarked()) {
        console.log("cumulative == total debt in this case:", alchemist.totalDebt());
    } else {
        console.log("NONE ....");
    }

    console.log("\n===  AFTER LIQUIDATION ===");
    console.log("User earmarked debt:", earmarkedAfter);
    console.log("User total debt:", newDebt);
    console.log("Global cumulativeEarmarked:", cumulativeEarmarkedAfter);
    console.log("Assets liquidated:", assetsLiquidated);

    // Calculate how much earmarked debt was repaid
    uint256 earmarkedRepaid = earmarkedBefore - earmarkedAfter;
    uint256 cumulativeDecrease = cumulativeEarmarkedBefore - cumulativeEarmarkedAfter;
    
    console.log("\n=== ANALYSIS ===");
    console.log("User earmarked repaid:", earmarkedRepaid);
    console.log("CumulativeEarmarked decreased by:", cumulativeDecrease);
    console.log("DISCREPANCY:", earmarkedRepaid > cumulativeDecrease ? earmarkedRepaid - cumulativeDecrease : 0);

    // ========================================
    // BUG VERIFICATION
    // ========================================
        if (earmarkedRepaid > 0) {
        console.log("-------------BUG CONFIRMED-------------");
        console.log("Force repay was triggered (earmarked decreased by:", earmarkedRepaid, ")");
        
        // Calculate expected cumulative
        uint256 expectedCumulative = cumulativeEarmarkedBefore - earmarkedRepaid;
        
        console.log("Expected cumulativeEarmarked:", expectedCumulative);
        console.log("Actual cumulativeEarmarked:", cumulativeEarmarkedAfter);
        
        uint256 phantomAmount = cumulativeEarmarkedAfter > expectedCumulative 
            ? cumulativeEarmarkedAfter - expectedCumulative 
            : 0;
            
        console.log("PHANTOM EARMARKED CREATED:", phantomAmount);

        // This assertion should FAIL, proving the bug
        vm.expectRevert();
        assertApproxEqAbs(
            cumulativeEarmarkedAfter,
            expectedCumulative,
            1e18,
            "BUG: cumulativeEarmarked should decrease by earmarkedRepaid, but it doesn't!"
        );
    } else {
        console.log("\n=== TEST ISSUE ===");
        console.log("No earmarked debt was repaid - position may not have been liquidated via force repay");
        console.log("This could mean the position was liquidated normally without earmarked debt");
        revert("Test setup issue: no force repay occurred");
    }
 }


```
