# 58531 sc critical querygraph function zero return bug causing tracking earmarking failure over progressive block intervals

**Submitted on Nov 3rd 2025 at 03:21:08 UTC by @Outliers for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58531
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/Transmuter.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hour

## Description

## Brief/Intro

Users lock up their debt tokens with the expectation of being repaid over time. However, there is a critical flaw that can bring this entire process to a halt.

The problem lies within the `queryGraph` function. It contains a condition that returns 0 if the starting and ending blocks are the same. While this may seem logical at first glance, it disrupts the system when `_earmark()` is called. If `_earmark()` attempts to retrieve data for a single block (which is a perfectly valid request), `queryGraph` returns 0. This failure in data retrieval halts the earmarking process, causing the repayment system to freeze.

As a result, users' debts do not get marked for redemption, leading to discrepancies in the system's accounting. This discrepancy makes the total debt appear larger than it should be relative to the synthetic assets. Consequently, users may find themselves repaying more MYT than necessary or may risk being liquidated unfairly.

## Vulnerability Details

Let’s examine the problematic code:

```solidity
function queryGraph(uint256 startBlock, uint256 endBlock) external view returns (uint256) {
    // The Bug: This rejects valid single-block queries
    if (endBlock <= startBlock) return 0;

    // ... rest of the function ...
}
```

The issue arises because the condition `<=` is too strict. A query for a single block, such as `queryGraph(N + 1, N + 1)`, is a legitimate request; however, this condition incorrectly treats it as invalid and returns 0.

This flaw disrupts the earmarking process here:

```solidity
function _earmark() internal {
    // ...
    // If this gets called in the same block, amount will be 0
    uint256 amount = ITransmuter(transmuter).queryGraph(lastEarmarkBlock + 1, block.number);

.....

        if (amount > 0 && liveUnearmarked != 0) {
            // Previous earmark survival
            uint256 previousSurvival = PositionDecay.SurvivalFromWeight(_earmarkWeight);
            if (previousSurvival == 0) previousSurvival = ONE_Q128;

            // Fraction of unearmarked debt being earmarked now in UQ128.128
            uint256 earmarkedFraction = _divQ128(amount, liveUnearmarked);

            _survivalAccumulator += _mulQ128(previousSurvival, earmarkedFraction);
            _earmarkWeight += PositionDecay.WeightIncrement(amount, liveUnearmarked);

            cumulativeEarmarked += amount;
        }

@audit >>>        lastEarmarkBlock = block.number;
    }
```

When `amount` is 0, no new debt is earmarked but we update the lastEarmarkBlock to block.number. This leads to a freeze in the repayment process, causing the system's core accounting—such as `totalDebt` and `cumulativeEarmarked`—to become desynchronized. These imbalances create a ripple effect throughout the entire system, affecting redemptions and collateral calculations, and may even result in underflow errors during updates.

```solidity

  /// @inheritdoc IAlchemistV3Actions
    function redeem(uint256 amount) external onlyTransmuter {
        _earmark();

@audit>>        uint256 liveEarmarked = cumulativeEarmarked;
@audit>>         if (amount > liveEarmarked) amount = liveEarmarked;

        // observed transmuter pre-balance -> potential cover
        uint256 transmuterBal = TokenUtils.safeBalanceOf(myt, address(transmuter));
        uint256 deltaYield    = transmuterBal > lastTransmuterTokenBalance ? transmuterBal - lastTransmuterTokenBalance : 0;
        uint256 coverDebt = convertYieldTokensToDebt(deltaYield);

        // cap cover so we never consume beyond remaining earmarked
@audit>>         uint256 coverToApplyDebt = amount + coverDebt > liveEarmarked ? (liveEarmarked - amount) : coverDebt;

        uint256 redeemedDebtTotal = amount + coverToApplyDebt;

       // Apply redemption weights/decay to the full amount that left the earmarked bucket
        if (liveEarmarked != 0 && redeemedDebtTotal != 0) {
  @audit>>           uint256 survival = ((liveEarmarked - redeemedDebtTotal) << 128) / liveEarmarked;
@audit>>             _survivalAccumulator = _mulQ128(_survivalAccumulator, survival);
@audit>>             _redemptionWeight += PositionDecay.WeightIncrement(redeemedDebtTotal, cumulativeEarmarked);
        }

        // earmarks are reduced by the full redeemed amount (net + cover)
        cumulativeEarmarked -= redeemedDebtTotal;

        // global borrower debt falls by the full redeemed amount
        totalDebt -= redeemedDebtTotal;

@audit>>         lastRedemptionBlock = block.number;



```

## Impact Details

The immediate effect of this flaw is that the transmuter can freeze, halting redemptions of debts for the affect blocks and throwing the system's accounting into disarray. This leads to a cascade of problems:

* **Broken Accounting:** The total debt and synthetic asset supply become misaligned, violating a core system invariant.
* **Unfair Costs for Users:** Users may end up repaying more MYT than necessary to clear their debts.
* **Inaccurate Collateral:** The calculations for locked collateral become incorrect, which can lead to transaction failures (like underflow errors) or inaccuracies in collateral weights.

## References

## Proof of Concept

## Proof of Concept

```solidity
function testEarmarkedFreeze_ShowProgressiveImpact() external {
    console.log("=== SETUP PHASE ===");
    
    // Seed the system with whale liquidity
    vm.startPrank(someWhale);
    IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
    vm.stopPrank();

    console.log("Initial MYT balance of 0xbeef:", vault.balanceOf(address(0xbeef)));

    // User 0xbeef opens a position
    vm.startPrank(address(0xbeef));
    uint256 depositAmount = 50e18;
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
    alchemist.deposit(depositAmount, address(0xbeef), 0);
    
    console.log("0xbeef alToken balance after deposit:", alToken.balanceOf(address(0xbeef)));

    uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    uint256 mintAmount = alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization;
    alchemist.mint(tokenIdFor0xBeef, mintAmount, address(0xbeef));
    vm.stopPrank();

    // Initial state
    (uint256 initialCollateral, uint256 initialDebt, uint initialMarked) = alchemist.getCDP(tokenIdFor0xBeef);
    console.log("\n=== INITIAL CDP STATE ===");
    console.log("Collateral:", initialCollateral);
    console.log("Debt:", initialDebt);
    console.log("Marked:", initialMarked);

    console.log("\n=== PRE-REDEMPTION POKES (5 blocks) ===");
    // Show progressive impact before redemption
    for (uint i = 1; i <= 5; i++) {
        vm.roll(block.number + 1);
        alchemist.poke(tokenIdFor0xBeef);
        
        (uint256 collateral, uint256 debt, uint marked) = alchemist.getCDP(tokenIdFor0xBeef);
      //  console.log("Block %s - Collateral: %s, Debt: %s, Marked: %s (Δ: %s)", i, collateral, debt, marked, marked - initialMarked);
    }

    console.log("\n=== CREATING REDEMPTION ===");
    console.log("0xbeef alToken balance before redemption:", alToken.balanceOf(address(0xbeef)));
    console.log("0xbeef MYT balance before redemption:", vault.balanceOf(address(0xbeef)));

    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
    transmuterLogic.createRedemption(mintAmount);
    vm.stopPrank();

    // Get state immediately after redemption creation
    (uint256 postCreateCollateral, uint256 postCreateDebt, uint postCreateMarked) = alchemist.getCDP(tokenIdFor0xBeef);
    
    console.log("\n=== POST-REDEMPTION STATE ===");
    console.log("Total Debt:", alchemist.totalSyntheticsIssued());
    console.log("Cumulative Earmarked:", alchemist.cumulativeEarmarked());
    console.log("Earmarked Weight:", alchemist.earmarkedweight());
    console.log("Survival Accumulator:", alchemist.survivalaccumulator());
    console.log("CDP - Collateral: %s, Debt: %s, Marked: %s", 
        postCreateCollateral, postCreateDebt, postCreateMarked);

    console.log("\n=== PROGRESSIVE IMPACT OVER 5 BLOCKS ===");
    uint256[] memory cumulativeEarmarked = new uint256[](5);
    uint256[] memory earmarkedWeights = new uint256[](5);
    uint256[] memory markedValues = new uint256[](5);
    
    // Track progressive earmarking (or lack thereof due to bug)
    for (uint i = 0; i < 5; i++) {
        vm.roll(block.number + 1);
        alchemist.poke(tokenIdFor0xBeef);
        
        (uint256 collateral, uint256 debt, uint marked) = alchemist.getCDP(tokenIdFor0xBeef);
        cumulativeEarmarked[i] = alchemist.cumulativeEarmarked();
        earmarkedWeights[i] = alchemist.earmarkedweight();
        markedValues[i] = marked;
        
        console.log("Block %s after redemption:", i + 1);
    //    console.log("  Cumulative Earmarked: %s (Δ: %s)",  dcumulativeEarmarked[i], i == 0 ? cumulativeEarmarked[i] - alchemist.cumulativeEarmarked() : cumulativeEarmarked[i] - cumulativeEarmarked[i-1]);
    //    console.log("  Earmarked Weight: %s", earmarkedWeights[i]);
    //    //console.log("  CDP Marked: %s (Δ: %s)", markedValues[i], markedValues[i] - postCreateMarked);
     //   console.log("  Frozen: %s", (i > 0 && cumulativeEarmarked[i] == cumulativeEarmarked[i-1]) ? "YES ❌" : "NO ✅");
        console.log("");
        
        // Show individual account state progression
        (uint rawLocked, uint lastCollateralWeight, uint lastAccruedEarmarkWeight, , uint lastSurvivalAccumulator) = 
            alchemist.seelockedindividual(tokenIdFor0xBeef);
        console.log("  Account State - RawLocked: %s, EarmarkWeight: %s, SurvivalAcc: %s", 
            rawLocked, lastAccruedEarmarkWeight, lastSurvivalAccumulator);
    }

    console.log("\n===pre CLAIMING REDEMPTION ===");
    uint256 preClaimMYT = vault.balanceOf(address(0xbeef));
    
    vm.startPrank(address(0xbeef));
    transmuterLogic.claimRedemption(1);
    vm.stopPrank();

    uint256 postClaimMYT = vault.balanceOf(address(0xbeef));
    console.log("MYT received from redemption: %s", postClaimMYT - preClaimMYT);
    console.log("Final MYT balance of 0xbeef:", postClaimMYT);

    console.log("\n=== FINAL STATE AFTER CLAIM ===");
    (uint256 finalCollateral, uint256 finalDebt, uint finalMarked) = alchemist.getCDP(tokenIdFor0xBeef);
    console.log("CDP - Collateral: %s, Debt: %s, Marked: %s", finalCollateral, finalDebt, finalMarked);
    console.log("Total Locked:", alchemist.totallocked());
    console.log("Earmarked Weight:", alchemist.earmarkedweight());
    console.log("Cumulative Earmarked:", alchemist.cumulativeEarmarked());

    // Final poke to sync everything
    console.log("\n=== FINAL SYNC ===");
    vm.startPrank(address(0xbeef));
    uint256 collateralBefore = alchemist.collateralbalancechecker(tokenIdFor0xBeef);
    alchemist.poke(tokenIdFor0xBeef);
    uint256 collateralAfter = alchemist.collateralbalancechecker(tokenIdFor0xBeef);
 //   console.log("Collateral balance - Before: %s, After: %s (Δ: %s)", 
  //      collateralBefore, collateralAfter, collateralAfter - collateralBefore);
    vm.stopPrank();

    // Show final individual state
    (uint finalRawLocked, uint finalCollateralWeight, uint finalEarmarkWeight, uint finalRedemptionWeight, uint finalSurvivalAcc) = 
        alchemist.seelockedindividual(tokenIdFor0xBeef);
    console.log("\n=== FINAL ACCOUNT STATE ===");
    console.log("RawLocked: %s", finalRawLocked);
    console.log("Collateral Weight: %s", finalCollateralWeight);
    console.log("Earmark Weight: %s", finalEarmarkWeight);
    console.log("Redemption Weight: %s", finalRedemptionWeight);
    console.log("Survival Accumulator: %s", finalSurvivalAcc);

    // Clean up - repay if any debt remains
    (uint256 cleanupCollateral, uint256 cleanupDebt, ) = alchemist.getCDP(tokenIdFor0xBeef);
    if (cleanupDebt > 0) {
        console.log("\n=== CLEANUP - REPAYING REMAINING DEBT ===");
        uint256 beefMYTBalance = vault.balanceOf(address(0xbeef));
        console.log("Remaining debt: %s, 0xbeef MYT balance: %s", cleanupDebt, beefMYTBalance);
        
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), beefMYTBalance);
        alchemist.repay(cleanupDebt, tokenIdFor0xBeef);
        vm.stopPrank();
        
        (uint256 finalCollat, uint256 finalDebt, ) = alchemist.getCDP(tokenIdFor0xBeef);
        console.log("After repayment - Collateral: %s, Debt: %s", finalCollat, finalDebt);
    }else{
       console.log("\n=== final state ===");
        uint256 beefMYTBalance = vault.balanceOf(address(0xbeef));
        console.log("Remaining debt: %s, 0xbeef MYT balance: %s", cleanupDebt, beefMYTBalance);
        
        (uint256 finalCollat, uint256 finalDebt, ) = alchemist.getCDP(tokenIdFor0xBeef);
        console.log("After repayment - Collateral: %s, Debt: %s", finalCollat, finalDebt);}

    console.log("\n=== KEY METRICS SUMMARY ===");
    console.log("Initial Cumulative Earmarked: %s", alchemist.cumulativeEarmarked());
    console.log("Final Cumulative Earmarked: %s", alchemist.cumulativeEarmarked());
 //   console.log("Earmarking Progress: %s", 
 //       (alchemist.cumulativeEarmarked() > initialMarked) ? "WORKING " : "FROZEN ");
    console.log("Total Debt Reduction: %s", initialDebt - finalDebt);
}

```

Result

```solidity

Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] testEarmarkedFreeze_ShowProgressiveImpact() (gas: 4261479)
Logs:
  === SETUP PHASE ===
  Initial MYT balance of 0xbeef: 200000000000000000000000
  0xbeef alToken balance after deposit: 0
  
=== INITIAL CDP STATE ===
  Collateral: 50000000000000000000
  Debt: 45000000000000000004
  Marked: 0
  
=== PRE-REDEMPTION POKES (5 blocks) ===
  
=== CREATING REDEMPTION ===
  0xbeef alToken balance before redemption: 45000000000000000004
  0xbeef MYT balance before redemption: 199950000000000000000000
  
=== POST-REDEMPTION STATE ===
  Total Debt: 45000000000000000004
  Cumulative Earmarked: 0
  Earmarked Weight: 0
  Survival Accumulator: 0
  CDP - Collateral: 50000000000000000000, Debt: 45000000000000000004, Marked: 0
  
=== PROGRESSIVE IMPACT OVER 5 BLOCKS ===
  Block 1 after redemption:
  
    Account State - RawLocked: 49999999999999999999, EarmarkWeight: 0, SurvivalAcc: 0
  Block 2 after redemption:
  
    Account State - RawLocked: 49999999999999999999, EarmarkWeight: 0, SurvivalAcc: 0
  Block 3 after redemption:
  
    Account State - RawLocked: 49999999999999999999, EarmarkWeight: 0, SurvivalAcc: 0
  Block 4 after redemption:
  
    Account State - RawLocked: 49999999999999999999, EarmarkWeight: 0, SurvivalAcc: 0
  Block 5 after redemption:
  
    Account State - RawLocked: 49999999999999999999, EarmarkWeight: 0, SurvivalAcc: 0
  
===pre CLAIMING REDEMPTION ===
  
=== CLAIMING REDEMPTION ===
  MYT received from redemption: 44954957234589041101
  Final MYT balance of 0xbeef: 199994954957234589041101
  
=== FINAL STATE AFTER CLAIM ===
  CDP - Collateral: 4550043236301369858, Debt: 42808219178081, Marked: 0
  Total Locked: 4550043236301369858
  Earmarked Weight: 26589352779673134787221095497755325187
  Cumulative Earmarked: 0
  
=== FINAL SYNC ===
  
=== FINAL ACCOUNT STATE ===
  RawLocked: 47564687975645
  Collateral Weight: 4596438422007443907515029410642248225
  Earmark Weight: 26589352779673134787221095497755325187
  Redemption Weight: 170141183460469231731687303715884105729
  Survival Accumulator: 0
  
=== CLEANUP - REPAYING REMAINING DEBT ===
  Remaining debt: 42808219178081, 
  0xbeef MYT balance: 199994954957234589041101
  After repayment - Collateral: 4550042808219178078, Debt: 0
  
=== KEY METRICS SUMMARY ===
  Initial Cumulative Earmarked: 0
  Final Cumulative Earmarked: 0
  Total Debt Reduction: 44999957191780821923

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 20.76ms (7.69ms CPU time)

Ran 1 test suite in 118.12ms (20.76ms CPU time)
```

Correct state

```solidity

Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] testEarmarkedFreeze_ShowProgressiveImpact() (gas: 3168003)
Logs:
  === SETUP PHASE ===
  Initial MYT balance of 0xbeef: 200000000000000000000000
  0xbeef alToken balance after deposit: 0
  
=== INITIAL CDP STATE ===
  Collateral: 50000000000000000000
  Debt: 45000000000000000004
  Marked: 0
  
=== PRE-REDEMPTION POKES (5 blocks) ===
  
=== CREATING REDEMPTION ===
  0xbeef alToken balance before redemption: 45000000000000000004
  0xbeef MYT balance before redemption: 199950000000000000000000
  
===pre CLAIMING REDEMPTION ===
  
=== CLAIMING REDEMPTION ===
  MYT received from redemption: 44955000000000000004
  Final MYT balance of 0xbeef: 199994955000000000000004
  
=== FINAL STATE AFTER CLAIM ===
  CDP - Collateral: 4549999999999999995, Debt: 0, Marked: 0
  Total Locked: 4549999999999999995
  Earmarked Weight: 170141183460469231731687303715884105729
  Cumulative Earmarked: 0
  
=== FINAL SYNC ===
  
=== FINAL ACCOUNT STATE ===
  RawLocked: 0
  Collateral Weight: 4596456644555066734126429258866559846
  Earmark Weight: 170141183460469231731687303715884105729
  Redemption Weight: 170141183460469231731687303715884105729
  Survival Accumulator: 0
  
=== final state ===
  Remaining debt: 0, 0xbeef MYT balance: 199994955000000000000004
  After repayment - Collateral: 4549999999999999995, Debt: 0
  
=== KEY METRICS SUMMARY ===
  Initial Cumulative Earmarked: 0
  Final Cumulative Earmarked: 0
  Total Debt Reduction: 45000000000000000004

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 20.06ms (6.20ms CPU time)

```

bug gives a final balance of 199994954957234589041101 , normal state gives 199994955000000000000004
