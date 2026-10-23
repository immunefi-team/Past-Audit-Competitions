# 58336 sc medium additive update to survival accumulator causing overflow&#x20;

**Submitted on Nov 1st 2025 at 11:20:56 UTC by @w3llyc4de20Ik2nn1 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58336
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The additive update to `_survivalAccumulator` in `_earmark()` causes unbounded linear growth and uint256 overflow by summing weighted survival fractions instead of multiplicatively compounding them, corrupting cumulative decay tracking and enabling protocol insolvency through distorted debt and collateralization calculations.

## Vulnerability Details

The additive update to `_survivalAccumulator` in the `_earmark` function of the `AlchemistV3` contract causes unbounded linear growth and eventual uint256 overflow by summing weighted survival fractions over multiple calls, corrupting cumulative decay tracking instead of multiplicatively compounding them to remain bounded ≤1 as intended in the matching redeem logic.

```solidity
// ============================================ // IN redeem() - Uses MULTIPLICATION // ============================================ function redeem(uint256 amount) external onlyTransmuter { _earmark(); 

uint256 liveEarmarked = cumulativeEarmarked; 
if (amount > liveEarmarked) amount = liveEarmarked; 
 
uint256 transmuterBal = TokenUtils.safeBalanceOf(myt, address(transmuter)); 
uint256 deltaYield    = transmuterBal > lastTransmuterTokenBalance ? transmuterBal - lastTransmuterTokenBalance : 0; 
uint256 coverDebt = convertYieldTokensToDebt(deltaYield); 
uint256 coverToApplyDebt = amount + coverDebt > liveEarmarked ? (liveEarmarked - amount) : coverDebt; 
uint256 redeemedDebtTotal = amount + coverToApplyDebt; 
 
// LOOK HERE: _survivalAccumulator is MULTIPLIED 
if (liveEarmarked != 0 && redeemedDebtTotal != 0) { 
   uint256 survival = ((liveEarmarked - redeemedDebtTotal) << 128) / liveEarmarked; 
   _survivalAccumulator = _mulQ128(_survivalAccumulator, survival);  // ← MULTIPLICATION 
   _redemptionWeight += PositionDecay.WeightIncrement(redeemedDebtTotal, cumulativeEarmarked); 
} 
 
// ... rest of redemption logic 
 

} 

// ============================================ // IN _earmark() - Uses ADDITION (BUG!) // ============================================ function _earmark() internal { if (totalDebt == 0) return; if (block.number <= lastEarmarkBlock) return; 

uint256 transmuterCurrentBalance = TokenUtils.safeBalanceOf(myt, address(transmuter)); 
uint256 transmuterDifference = transmuterCurrentBalance > lastTransmuterTokenBalance ? transmuterCurrentBalance - lastTransmuterTokenBalance : 0; 
uint256 amount = ITransmuter(transmuter).queryGraph(lastEarmarkBlock + 1, block.number); 
uint256 coverInDebt = convertYieldTokensToDebt(transmuterDifference); 
amount = amount > coverInDebt ? amount - coverInDebt : 0; 
 
lastTransmuterTokenBalance = transmuterCurrentBalance; 
uint256 liveUnearmarked = totalDebt - cumulativeEarmarked; 
if (amount > liveUnearmarked) amount = liveUnearmarked; 
 
if (amount > 0 && liveUnearmarked != 0) { 
   uint256 previousSurvival = PositionDecay.SurvivalFromWeight(_earmarkWeight); 
   if (previousSurvival == 0) previousSurvival = ONE_Q128; 
   uint256 earmarkedFraction = _divQ128(amount, liveUnearmarked); 
 
   //  BUG HERE: _survivalAccumulator uses ADDITION instead of MULTIPLICATION 
   _survivalAccumulator += _mulQ128(previousSurvival, earmarkedFraction);  // ← ADDITION (WRONG!) 
   _earmarkWeight += PositionDecay.WeightIncrement(amount, liveUnearmarked); 
 
   cumulativeEarmarked += amount; 
} 
 
lastEarmarkBlock = block.number; 
 

} 

```

The Problem:

`redeem()` does:

```solidity

_survivalAccumulator = _mulQ128(_survivalAccumulator, survival);  // Compound multiplication 
```

`_earmark()` does:

```solidity

_survivalAccumulator += _mulQ128(previousSurvival, earmarkedFraction);  // Addition 
```

This line adds the product of the previous survival rate and the earmarked fraction to `_survivalAccumulator` on every earmark call, leading to linear accumulation that grows unbounded over time (e.g., repeated partial earmarks sum fractions exceeding 1.0), eventually overflowing uint256 and corrupting the fixed-point value; the intended behavior is multiplicative compounding to track cumulative survival rates bounded ≤1.0, as seen in redeem's similar update.

## Impact Details

The overflow corrupts position decay and survival calculations, allowing manipulated earmarks to distort debt tracking and potentially cause protocol insolvency through incorrect collateralization assessments.

Why This is a Bug:

Survival rates should be compounded multiplicatively, not added:

If 90% survives event 1, then 90% of remainder survives event 2

Total survival = 0.9 × 0.9 = 0.81 (not 0.9 + 0.9 = 1.8)

## Recommended Mitigation

Replace the additive update with a multiplicative compound of `_survivalAccumulator` by the blended survival fraction for the new earmark (prior survival for the earmarked portion + 1.0 for fresh unearmarked debt), ensuring bounded growth while preserving the weighted survival intent.

Fixed Code Snippet

```solidity

// In _earmark, replace the if-block update with: 
if (amount > 0 && liveUnearmarked != 0) { 
   uint256 previousSurvival = PositionDecay.SurvivalFromWeight(_earmarkWeight); 
   if (previousSurvival == 0) previousSurvival = ONE_Q128; 
 
   uint256 earmarkedFraction = _divQ128(amount, liveUnearmarked); 
 
   // Blended survival: prior for earmarked portion + 1.0 for fresh unearmarked 
   uint256 blendedSurvival = _mulQ128(previousSurvival, earmarkedFraction) + _divQ128(ONE_Q128 * (ONE_Q128 - earmarkedFraction), ONE_Q128); 
   _survivalAccumulator = _mulQ128(_survivalAccumulator, blendedSurvival);  // Multiplicative compound 
 
   _earmarkWeight += PositionDecay.WeightIncrement(amount, liveUnearmarked); 
   cumulativeEarmarked += amount; 
} 
```

(Apply analogously to the simulation in `_calculateUnrealizedDebt` for consistency.)

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L738>

## Proof of Concept

## Proof of Concept

First put the test in the AlchemistV3.t.sol Run with forge test --match-test testSurvivalAccumulatorUnboundedGrowthProof –vvvv

```solidity
function testSurvivalAccumulatorUnboundedGrowthProof() external { // This test proves the additive bug in _survivalAccumulator causes unbounded growth // by showing that after MANY earmark cycles, the calculations become corrupted 

uint256 depositAmount = 100000e18; 
uint256 mintAmount = 90000e18; 
 
// Create position 1 
vm.startPrank(address(0xbeef)); 
SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount); 
alchemist.deposit(depositAmount, address(0xbeef), 0); 
uint256 tokenId1 = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT)); 
alchemist.mint(tokenId1, mintAmount, address(0xbeef)); 
vm.stopPrank(); 
 
// Create position 2 
vm.startPrank(externalUser); 
SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount); 
alchemist.deposit(depositAmount, externalUser, 0); 
uint256 tokenId2 = AlchemistNFTHelper.getFirstTokenId(externalUser, address(alchemistNFT)); 
alchemist.mint(tokenId2, mintAmount, externalUser); 
vm.stopPrank(); 
 
// Total debt: 180000e18 
uint256 initialTotalDebt = alchemist.totalDebt(); 
 
// Create many small redemptions to trigger MANY earmark cycles 
// Each earmark adds to _survivalAccumulator instead of multiplying 
vm.startPrank(anotherExternalUser); 
SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 180000e18); 
 
// Create 50 small redemptions over time to trigger 50 earmark cycles 
uint256 numRedemptions = 50; 
uint256 redemptionSize = 1000e18; // Small to trigger many cycles 
 
for (uint256 i = 0; i < numRedemptions; i++) { 
   transmuterLogic.createRedemption(redemptionSize); 
    
   // Advance blocks to accumulate earmarkable debt 
   vm.roll(block.number + 200000); // 200k blocks per cycle 
    
   // Trigger earmark by poking - this ADDS to _survivalAccumulator 
   alchemist.poke(tokenId1); 
} 
vm.stopPrank(); 
 
// After 50 earmark cycles with additions, _survivalAccumulator has grown unboundedly 
// Each cycle added: _mulQ128(previousSurvival, earmarkedFraction) 
// With 50 cycles, the accumulator has summed 50 terms 
 
// Get state before redemption claim 
uint256 cumulativeEarmarkedBefore = alchemist.cumulativeEarmarked(); 
(, uint256 debt1Before, uint256 earmark1Before) = alchemist.getCDP(tokenId1); 
(, uint256 debt2Before, uint256 earmark2Before) = alchemist.getCDP(tokenId2); 
 
// Now claim first redemption - this uses the corrupted _survivalAccumulator 
vm.roll(block.number + 5_256_000); 
 
vm.startPrank(anotherExternalUser); 
transmuterLogic.claimRedemption(1); // Claims 1000e18 
vm.stopPrank(); 
 
// Sync positions to apply redemption with corrupted accumulator 
alchemist.poke(tokenId1); 
alchemist.poke(tokenId2); 
 
// Get state after redemption 
uint256 cumulativeEarmarkedAfter = alchemist.cumulativeEarmarked(); 
(, uint256 debt1After, uint256 earmark1After) = alchemist.getCDP(tokenId1); 
(, uint256 debt2After, uint256 earmark2After) = alchemist.getCDP(tokenId2); 
 
// Calculate reductions 
uint256 globalEarmarkReduction = cumulativeEarmarkedBefore - cumulativeEarmarkedAfter; 
uint256 debt1Reduction = debt1Before - debt1After; 
uint256 debt2Reduction = debt2Before - debt2After; 
uint256 totalDebtReduction = debt1Reduction + debt2Reduction; 
 
// BUG PROOF: With the additive accumulator after 50 cycles: 
// - _survivalAccumulator has grown far beyond ONE_Q128 (the max for multiplication) 
// - When used in _sync(), it causes incorrect calculations 
// - The position-level debt reductions don't match the global earmark reduction 
 
// Expected: totalDebtReduction ≈ globalEarmarkReduction ≈ 1000e18 
// Actual: Due to corrupted accumulator, they diverge significantly 
 
uint256 discrepancy = totalDebtReduction > globalEarmarkReduction 
   ? totalDebtReduction - globalEarmarkReduction 
   : globalEarmarkReduction - totalDebtReduction; 
 
// After 50 additive cycles, the discrepancy should be substantial (>1%) 
uint256 discrepancyPercent = (discrepancy * 100e18) / globalEarmarkReduction; 
 
assertTrue( 
   discrepancyPercent > 1e18, // More than 1% discrepancy 
   "BUG PROVEN: 50 additive cycles caused >1% calculation error in survival accumulator" 
); 
 
// Additional proof: The total debt should have decreased by the redemption amount 
// But the corrupted accumulator causes incorrect tracking 
uint256 actualTotalDebtAfter = alchemist.totalDebt(); 
uint256 expectedTotalDebtAfter = initialTotalDebt - 1000e18; 
 
uint256 debtTrackingError = actualTotalDebtAfter > expectedTotalDebtAfter 
   ? actualTotalDebtAfter - expectedTotalDebtAfter 
   : expectedTotalDebtAfter - actualTotalDebtAfter; 
 
assertTrue( 
   debtTrackingError > 1e18, 
   "IMPACT PROVEN: Corrupted accumulator causes debt tracking errors >1e18" 
); 
 

} 
```
