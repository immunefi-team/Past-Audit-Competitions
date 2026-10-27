# 58383 sc high due to cumulativeearmarked not being updated in alchemix forcerepay user funds are locked longer due to slower debt decay and calculation of system collaterization rate is inc&#x20;

**Submitted on Nov 1st 2025 at 20:13:46 UTC by @DeoGratias for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58383
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency
  * Temporary freezing of funds for at least 24 hour

## Description

### Due to `cumulativeEarmarked` not being updated in `Alchemix::_forceRepay` user funds are locked longer due to slower debt decay and calculation of system collaterization rate is Incorrect

**Description:** When a position is liquidated, `AlchemistV3::_liquidate()` first calls `AlchemistV3::_forceRepay(accountId, account.earmarked)` to clear the position’s earmarked debt using its collateral. Inside \_forceRepay, the account’s local earmarked is reduced, but the global cumulativeEarmarked is not decremented to reflect that removal.

Because redeem() uses cumulativeEarmarked as the live denominator for decay/weight math, an inflated cumulativeEarmarked causes subsequent redemptions to be allocated over a larger-than-real bucket. Remaining users’ earmarks decay too slowly, delaying their debt reduction and effectively keeping their funds locked longer than intended.

```solidity
 if (liveEarmarked != 0 && redeemedDebtTotal != 0) {
            uint256 survival = ((liveEarmarked - redeemedDebtTotal) << 128) / liveEarmarked;
            _survivalAccumulator = _mulQ128(_survivalAccumulator, survival);
            _redemptionWeight += PositionDecay.WeightIncrement(redeemedDebtTotal, cumulativeEarmarked); //<-----
        }
```

As well because `AlchemistV3::_forceRepay(accountId, account.earmarked)` decrements `totalDebt` through `AlchemistV3::subDebt` without updating cumulativeEarmarked, `AlchemistV3::redeem` subtracts that phantom portion from totalDebt a second time (`totalDebt -= redeemedDebtTotal`, redeemedDebtTotal is inflated because cumulativeEarmark is inflated). This results in a defalted totalDebt which subsequently affects liquidations because it affects calculation of system collaterization (alchemistCurrentCollateralization uses totalDebt in the denominator). This makes liquidations less likely when they should happen.

```solidity
(uint256 liquidationAmount, uint256 debtToBurn, uint256 baseFee, uint256 outsourcedFee) = calculateLiquidation(
            collateralInUnderlying,
            account.debt,
            minimumCollateralization,
            normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt, //<-------
            globalMinimumCollateralization,
            liquidatorFee
        );
```

By contrast, repay() correctly decrements both the account’s earmarked and the global cumulativeEarmarked.

**Impact:**

Users debt experience slower-than-correct earmark decay compared to the amount of collateral they lose after any \_forceRepay event. This delays their redemptions and extending time-to-unlock.

Deflated Total Debt -> Inflated System collaterization -> Affects liqudations -> Affects Protocol Solvency.

## Proof of Concept

**Proof of Concept:**

The provided test `testPOC_ForceRepay_SlowsDecay_WithBurn()` demonstrates:

1. Two users A & B mint near the minimum collateralization and build earmarks.
2. A triggers liquidation. `_forceRepay()` zeros **A’s local `earmarked`** but **does not** reduce `cumulativeEarmarked`. The test asserts:
   * `alchemist.cumulativeEarmarked() > (A.earmarked + B.earmarked)` (A’s removed earmark still counted globally).
3. A small `redeem()` is executed. Because `redeem()` uses the **inflated** `cumulativeEarmarked` as denominator, B’s observed earmark drop is **less** than the ideal drop computed using the true live sum of earmarks. The test verifies:
   * The total debt reduction matches the inflated-denominator cap.
   * `B_ChangeInEarMark * liveEarmarkSum < B_EarmarkBeforeRedeem * ObservedTotalDebtReduction` (i.e., B’s earmark decays too slowly, and is not proportional to the amount of debt reduced in the system).
   * As well we assert totalDebt < Sum(A.debt + b.debt) even after they have been synced.

To run the POC paste the below code in `AlchemistV3.t.sol` and run the test using the command `forge test --mt testPOC_ForceRepay_SlowsDown_DebtDecay`

```solidity
function testPOC_ForceRepay_SlowsDown_DebtDecay() external {
    // ─────────────────────────────────────────────────────────────────────────────
    // 0) Disable fees so only earmark/weight math matters
    // ─────────────────────────────────────────────────────────────────────────────
    vm.startPrank(alOwner);
    alchemist.setProtocolFee(0);
    alchemist.setRepaymentFee(0);
    vm.stopPrank();
    console.log("\n[0] Fees disabled: protocolFee=%e, repaymentFee=%e",
        alchemist.protocolFee(), alchemist.repaymentFee());

    // ─────────────────────────────────────────────────────────────────────────────
    // 1) Two borrowers (A & B) mint MAX at (near) minimum collateralization
    // ─────────────────────────────────────────────────────────────────────────────
    uint256 collateralDeposit = 100e18;

    // A mints
    vm.startPrank(address(0xBEEF));
    SafeERC20.safeApprove(address(vault), address(alchemist), collateralDeposit);
    alchemist.deposit(collateralDeposit, address(0xBEEF), 0);
    uint256 cdpIdA = AlchemistNFTHelper.getFirstTokenId(address(0xBEEF), address(alchemistNFT));
    uint256 maxMintA = (alchemist.totalValue(cdpIdA) * FIXED_POINT_SCALAR) / alchemist.minimumCollateralization();
    alchemist.mint(cdpIdA, maxMintA, address(0xBEEF));
    vm.stopPrank();
    console.log("\n[1] A minted:");
    console.log("  cdpIdA               : %e", cdpIdA);
    console.log("  deposit (MYT shares) : %e", collateralDeposit);
    console.log("  maxMintA (alDebt)    : %e", maxMintA);

    // B mints
    vm.startPrank(externalUser);
    SafeERC20.safeApprove(address(vault), address(alchemist), collateralDeposit);
    alchemist.deposit(collateralDeposit, externalUser, 0);
    uint256 cdpIdB = AlchemistNFTHelper.getFirstTokenId(externalUser, address(alchemistNFT));
    uint256 maxMintB = (alchemist.totalValue(cdpIdB) * FIXED_POINT_SCALAR) / alchemist.minimumCollateralization();
    alchemist.mint(cdpIdB, maxMintB, externalUser);
    vm.stopPrank();
    console.log("\n[1] B minted:");
    console.log("  cdpIdB               : %e", cdpIdB);
    console.log("  deposit (MYT shares) : %e", collateralDeposit);
    console.log("  maxMintB (alDebt)    : %e", maxMintB);

    console.log("\n[1] Totals after mint:");
    console.log("  totalDebt            : %e", alchemist.totalDebt());
    console.log("  totalSyntheticsIssued: %e", alchemist.totalSyntheticsIssued());
    console.log("  cumulativeEarmarked  : %e", alchemist.cumulativeEarmarked());

    // ─────────────────────────────────────────────────────────────────────────────
    // 2) Stake 50% of total debt into the Transmuter → earmarks will start accruing
    //    Capture the redemption position ID and advance to (just past) full maturity
    // ─────────────────────────────────────────────────────────────────────────────
    uint256 totalDebtBefore = alchemist.totalDebt(); // == totalSyntheticsIssued
    uint256 stake1 = totalDebtBefore / 5; // 20%
    uint256 stake2 = totalDebtBefore / 5; // another 20% (total 40%)

    vm.startPrank(address(0xDAD));
    IERC20(address(alToken)).approve(address(transmuterLogic), stake1);
    transmuterLogic.createRedemption(stake1);
    vm.stopPrank();

    vm.startPrank(address(0xBEEF));
    IERC20(address(alToken)).approve(address(transmuterLogic), stake2);
    transmuterLogic.createRedemption(stake2);
    vm.stopPrank();

    console.log("\n[2] Redemption created by 0xDAD:");
    console.log("  stake (alDebt)       : %e", stake1);
    console.log("  transmuter.totalLocked: %e", ITransmuter(transmuterLogic).totalLocked());

    console.log("\n[2] Redemption created by 0xBEEF:");
    console.log("  stake (alDebt)       : %e", stake2);
    console.log("  transmuter.totalLocked: %e", ITransmuter(transmuterLogic).totalLocked());

    // Roll to full maturity so the position has fully transmuted (maximizes redeem effect)
    ITransmuter.StakingPosition memory pos = transmuterLogic.getPosition(1);
    console.log("  position[1]: start=%e, mature=%e, amount=%e", pos.startBlock, pos.maturationBlock, pos.amount);
    vm.roll(pos.maturationBlock + 1);
    console.log("  rolled to block      : %e", block.number);

    // sync local earmarks for A & B to reflect accrual up to now
    alchemist.poke(cdpIdA);
    alchemist.poke(cdpIdB);
    {
        (uint256 cA, uint256 dA, uint256 eA) = alchemist.getCDP(cdpIdA);
        (uint256 cB, uint256 dB, uint256 eB) = alchemist.getCDP(cdpIdB);
        console.log("\n[2] After maturity & poke:");
        console.log("  A: coll=%e, debt=%e, earmark=%e", cA, dA, eA);
        console.log("  B: coll=%e, debt=%e, earmark=%e", cB, dB, eB);
        console.log("  cumulativeEarmarked  : %e", alchemist.cumulativeEarmarked());
    }

    // ─────────────────────────────────────────────────────────────────────────────
    // 3) A does a SMALL (partial) burn of *unearmarked* debt to hover near threshold
    // ─────────────────────────────────────────────────────────────────────────────
    (, uint256 debtA_beforeBurn, uint256 earmarkA_beforeBurn) = alchemist.getCDP(cdpIdA);
    uint256 unearmarkedDebtA = debtA_beforeBurn > earmarkA_beforeBurn ? debtA_beforeBurn - earmarkA_beforeBurn : 0;
    console.log("\n[3] A pre-burn:");
    console.log("  debtA_beforeBurn     : %e", debtA_beforeBurn);
    console.log("  earmarkA_beforeBurn  : %e", earmarkA_beforeBurn);
    console.log("  unearmarkedDebtA     : %e", unearmarkedDebtA);

    // Burn amount = 10% of unearmarked portion, but never exceeding burn capacity
    uint256 remainingBurnCapacity = alchemist.totalSyntheticsIssued() - ITransmuter(transmuterLogic).totalLocked();
    uint256 burnAmountA = unearmarkedDebtA / 10;
    if (burnAmountA > remainingBurnCapacity) burnAmountA = remainingBurnCapacity;

    console.log("  remainingBurnCapacity: %e", remainingBurnCapacity);
    console.log("  burnAmountA          : %e", burnAmountA);

    if (burnAmountA > 0) {
        vm.roll(block.number + 1); // avoid CannotRepayOnMintBlock
        vm.startPrank(address(0xBEEF));
        IERC20(address(alToken)).approve(address(alchemist), 0);
        IERC20(address(alToken)).approve(address(alchemist), burnAmountA);
        alchemist.burn(burnAmountA, cdpIdA);
        vm.stopPrank();
    }

    // State after partial burn
    (uint256 collateralA_preDrop, uint256 debtA_afterBurn,) = alchemist.getCDP(cdpIdA);
    console.log("\n[3] A after burn:");
    console.log("  collateralA_preDrop  : %e", collateralA_preDrop);
    console.log("  debtA_afterBurn      : %e", debtA_afterBurn);

    // ─────────────────────────────────────────────────────────────────────────────
    // 4) Precise price drop: make A just *under* the lower bound to ensure liquidation
    // ─────────────────────────────────────────────────────────────────────────────
    uint256 collatLowerBound = alchemist.collateralizationLowerBound(); // 1e18
    require(debtA_afterBurn > 0 && collateralA_preDrop > 0, "bad inputs");

    // pfBoundary = (lowerBound * debt) / collateral
    uint256 priceFactorAtBoundary = (collatLowerBound * debtA_afterBurn) / collateralA_preDrop;
    // Nudge 0.1% lower
    uint256 priceFactorTarget = (priceFactorAtBoundary * 999) / 1000; // 0.999 * boundary

    // Supply scales inversely with price; to multiply price by pf (<1), multiply supply by 1/pf
    uint256 tokenSupplyBefore = IERC20(address(mockStrategyYieldToken)).totalSupply();
    uint256 tokenSupplyTarget = (tokenSupplyBefore * 1e18 + priceFactorTarget - 1) / priceFactorTarget; // ceil-div
    console.log("\n[4] Price drop tuning:");
    console.log("  collatLowerBound     : %e", collatLowerBound);
    console.log("  priceFactorBoundary  : %e", priceFactorAtBoundary);
    console.log("  priceFactorTarget    : %e", priceFactorTarget);
    console.log("  tokenSupplyBefore    : %e", tokenSupplyBefore);
    console.log("  tokenSupplyTarget    : %e", tokenSupplyTarget);

    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(tokenSupplyTarget);

    // ─────────────────────────────────────────────────────────────────────────────
    // 5) Liquidate A → triggers _forceRepay(A.earmarked). Bug: global cumulativeEarmarked not decremented
    // ─────────────────────────────────────────────────────────────────────────────
    vm.prank(externalUser);
    alchemist.liquidate(cdpIdA);

    (, /*debtA_afterForceRepay*/, uint256 earmarkA_afterForceRepay) = alchemist.getCDP(cdpIdA);
    console.log("\n[5] After liquidation (forceRepay path):");
    console.log("  A.earmark afterForce : %e", earmarkA_afterForceRepay);

    // Compare live locals vs global
    (,, uint256 earmarkB_beforeRedeem) = alchemist.getCDP(cdpIdB);
    uint256 liveEarmarkSum = earmarkA_afterForceRepay + earmarkB_beforeRedeem;

    uint256 cumulativeEarmarkAfterForce = alchemist.cumulativeEarmarked();
    console.log("  B.earmark beforeRedeem: %e", earmarkB_beforeRedeem);
    console.log("  liveEarmarkSum (A+B)  : %e", liveEarmarkSum);
    console.log("  cumulativeEarmarked   : %e", cumulativeEarmarkAfterForce);

    assertEq(earmarkA_afterForceRepay, 0, "A.earmarked not zero after _forceRepay");
    assertGt(
        cumulativeEarmarkAfterForce,
        liveEarmarkSum,
        "global cumulativeEarmarked should still include A's removed earmark (desync/bug)"
    );

    // ─────────────────────────────────────────────────────────────────────────────
    // 6) Claim the Transmuter redemption (user path) → internal alchemist.redeem(...)
    // ─────────────────────────────────────────────────────────────────────────────
    uint256 cumulativeEarmarkBeforeRedeem = alchemist.cumulativeEarmarked();
    console.log("\n[6] Before claim:");
    console.log("  cumulativeEarmarked   : %e", cumulativeEarmarkBeforeRedeem);

    // The redemption position owner claims
    vm.prank(address(0xDAD));
    transmuterLogic.claimRedemption(1);

    uint256 cumulativeEarmarkAfterRedeem  = alchemist.cumulativeEarmarked();
    uint256 observedTotalDebtReduction    = cumulativeEarmarkBeforeRedeem - cumulativeEarmarkAfterRedeem;
    console.log("\n[6] After claim:");
    console.log("  cumulativeEarmarked   : %e", cumulativeEarmarkAfterRedeem);
    console.log("  observedDebtReduction : %e", observedTotalDebtReduction);
    require(observedTotalDebtReduction > 0, "no redemption occurred");

    // B’s actual earmark drop using the (buggy) global denominator
    alchemist.poke(cdpIdB);
    (,, uint256 earmarkB_afterRedeem) = alchemist.getCDP(cdpIdB);
    uint256 B_ChangeInEarMark = earmarkB_beforeRedeem - earmarkB_afterRedeem;
    console.log("  B.earmark afterRedeem : %e", earmarkB_afterRedeem);
    console.log("  B.changeInEarmark     : %e", B_ChangeInEarMark);

    // Ideal vs observed via cross-multiplication
    // ideal share = observedTotalDebtReduction * (earmarkB_beforeRedeem / liveEarmarkSum)
    uint256 lhs_observed = B_ChangeInEarMark * liveEarmarkSum;
    uint256 rhs_ideal    = earmarkB_beforeRedeem * observedTotalDebtReduction;
    console.log("  cross-mult observed   : %e", lhs_observed);
    console.log("  cross-mult ideal      : %e", rhs_ideal);

    assertLt(
        lhs_observed,
        rhs_ideal,
        "B's earmark did not decay slower than ideal - expected slower due to inflated global denominator"
    );

    // Final quick snapshot
    {
        (uint256 cA2, uint256 dA2, uint256 eA2) = alchemist.getCDP(cdpIdA);
        (uint256 cB2, uint256 dB2, uint256 eB2) = alchemist.getCDP(cdpIdB);
        console.log("\n[final] Snapshots:");
        console.log("  A: coll=%e, debt=%e, earmark=%e", cA2, dA2, eA2);
        console.log("  B: coll=%e, debt=%e, earmark=%e", cB2, dB2, eB2);
        console.log("  cumulativeEarmarked   : %e", alchemist.cumulativeEarmarked());
        console.log("  totalDebt             : %e", alchemist.totalDebt());
        console.log("  totalSyntheticsIssued : %e", alchemist.totalSyntheticsIssued());
        assertNotEq(dA2 + dB2, alchemist.totalDebt(), "debt sum mismatch");
    }
}
```

### Output

```
[PASS] testPOC_ForceRepay_SlowsDown_DebtDecay() (gas: 5967718)
Logs:
  
[0] Fees disabled: protocolFee=0e0, repaymentFee=0e0
  
[1] A minted:
    cdpIdA               : 1e0
    deposit (MYT shares) : 1e20
    maxMintA (alDebt)    : 9.0000000000000000009e19
  
[1] B minted:
    cdpIdB               : 2e0
    deposit (MYT shares) : 1e20
    maxMintB (alDebt)    : 9.0000000000000000009e19
  
[1] Totals after mint:
    totalDebt            : 1.80000000000000000018e20
    totalSyntheticsIssued: 1.80000000000000000018e20
    cumulativeEarmarked  : 0e0
  
[2] Redemption created by 0xDAD:
    stake (alDebt)       : 3.6000000000000000003e19
    transmuter.totalLocked: 7.2000000000000000006e19
  
[2] Redemption created by 0xBEEF:
    stake (alDebt)       : 3.6000000000000000003e19
    transmuter.totalLocked: 7.2000000000000000006e19
    position[1]: start=1e0, mature=5.256001e6, amount=3.6000000000000000003e19
    rolled to block      : 5.256002e6
  
[2] After maturity & poke:
    A: coll=1e20, debt=9.0000000000000000009e19, earmark=3.6000000000000000003e19
    B: coll=1e20, debt=9.0000000000000000009e19, earmark=3.6000000000000000003e19
    cumulativeEarmarked  : 7.2000000000000000006e19
  
[3] A pre-burn:
    debtA_beforeBurn     : 9.0000000000000000009e19
    earmarkA_beforeBurn  : 3.6000000000000000003e19
    unearmarkedDebtA     : 5.4000000000000000006e19
    remainingBurnCapacity: 1.08000000000000000012e20
    burnAmountA          : 5.4e18
  
[3] A after burn:
    collateralA_preDrop  : 1e20
    debtA_afterBurn      : 8.4600000000000000009e19
  
[4] Price drop tuning:
    collatLowerBound     : 1.05263157895e18
    priceFactorBoundary  : 8.905263157917e17
    priceFactorTarget    : 8.896357894759083e17
    tokenSupplyBefore    : 1e24
    tokenSupplyTarget    : 1.124055497575146068053877e24
  
[5] After liquidation (forceRepay path):
    A.earmark afterForce : 0e0
    B.earmark beforeRedeem: 3.6000000000000000003e19
    liveEarmarkSum (A+B)  : 3.6000000000000000003e19
    cumulativeEarmarked   : 7.2000000000000000006e19
  
[6] Before claim:
    cumulativeEarmarked   : 7.2000000000000000006e19
  redeemedDebtTotal: 3.6000000000000000003e19
  
[6] After claim:
    cumulativeEarmarked   : 3.6000000000000000003e19
    observedDebtReduction : 3.6000000000000000003e19
    B.earmark afterRedeem : 1.8000000000000000002e19
    B.changeInEarmark     : 1.8000000000000000001e19
    cross-mult observed   : 6.48000000000000000090000000000000000003e38
    cross-mult ideal      : 1.296000000000000000216000000000000000009e39
  
[final] Snapshots:
    A: coll=5.9534002087294741501e19, debt=4.8600000000000000006e19, earmark=0e0
    B: coll=9.9999999999999999999e19, debt=7.2000000000000000008e19, earmark=1.8000000000000000002e19
    cumulativeEarmarked   : 3.6000000000000000003e19
    totalDebt             : 1.02600000000000000012e20
    totalSyntheticsIssued : 1.38600000000000000015e20

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 38.81ms (16.36ms CPU time)
```
