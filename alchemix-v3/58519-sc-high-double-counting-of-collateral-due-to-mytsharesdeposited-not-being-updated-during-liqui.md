# 58519 sc high double counting of collateral due to mytsharesdeposited not being updated during liquidations

**Submitted on Nov 2nd 2025 at 23:50:45 UTC by @DeusVult for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58519
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Protocol insolvency

## Description

**Description:**

* The Alchemist tracks protocol TVL for MYT shares via the internal counter `_mytSharesDeposited`, and `getTotalUnderlyingValue()` computes TVL from this counter: it returns `convertYieldTokensToUnderlying(_mytSharesDeposited)`.
* Several pathways transfer MYT shares out of the Alchemist without decrementing `_mytSharesDeposited`, leaving the Alchemist’s reported TVL unchanged even though shares actually left the contract:
  * `_forceRepay`: Transfers MYT to the Transmuter and to the protocol fee receiver, but never decrements `_mytSharesDeposited`.
  * `_liquidate` early-exit branches (repayment-only, no liquidation): Pays the repayment fee to the liquidator from Alchemist MYT, but never decrements `_mytSharesDeposited`.
  * `_doLiquidation` (actual liquidation): Transfers MYT to the Transmuter and potentially to the liquidator as fee, but never decrements `_mytSharesDeposited`.
* The Transmuter computes a “bad-debt ratio” using a denominator equal to $TVL\_{alchemist} + underlying(\text{Transmuter MYT})$. Because `_mytSharesDeposited` is not decreased when MYT leaves the Alchemist for the Transmuter, the same shares are effectively counted twice in the denominator: once via the Alchemist’s stale TVL and once via the Transmuter’s live balance.

**Impact:**

* Suppressed scaling during claimRedemption:
  * The bad-debt ratio becomes artificially small because the denominator is inflated by double counting, reducing or eliminating the intended scaling of redemptions under insolvency. Early claimants can redeem too much at par, to the detriment of later participants.
* Under-liquidation:
  * `AlchemixV3::calculateLiquidation` uses the global collateralization underlying/total\_debt. With an overstated `_getTotalUnderlyingValue()`, the system appears healthier, causing lighter liquidations or skipping high-LTV full liquidations that should occur, leaving the system riskier and less solvent than reported.

## Proof of Concept

**Proof of Concept:**

* The provided test demonstrates the issue end-to-end with logs and assertions:
  * Fees are zeroed to isolate share accounting.
  * A borrower deposits MYT and mints near the minimum collateralization, then a Transmuter redemption is created so earmarks accrue.
  * A price move makes the borrower liquidatable.
  * Before liquidation, the test snapshots:
    * Alchemist TVL: `getTotalUnderlyingValue()` (which relies on `_mytSharesDeposited`),
    * Transmuter’s MYT share balance, and
    * The “Transmuter denominator” denom = TVL + underlying(transmuter).
  * After triggering `alchemist.liquidate(tokenIdA)`, MYT shares move from the Alchemist to the Transmuter:
    * The Alchemist’s actual MYT ERC20 balance drops by the amount moved.
    * However, `getTotalUnderlyingValue()` remains unchanged because `_mytSharesDeposited` was not decremented in the liquidation paths.
    * The denominator used by the Transmuter increases exactly by the underlying value of the shares that moved, even though total system underlying did not increase. This proves double counting: those shares are still counted inside the Alchemist’s TVL while also being counted again as Transmuter-held MYT.
  * The assertions confirm:
    * Alchemist ERC20 balance decreased as expected,
    * TVL (from `_mytSharesDeposited`) incorrectly stayed the same,
    * The combined denominator increased by the underlying value of the transferred shares, evidencing the double count.

To run this POC paste the below code in `AlchemixV3.t.sol` and run the POC using the command `forge test --mt testPOC_MYT_DoubleCounting_After_Liquidation_Moves_Shares_To_Transmuter_WITH_LOGS`

```solidity
function testPOC_MYT_DoubleCounting_After_Liquidation_Moves_Shares_To_Transmuter_WITH_LOGS() external {
    // ─────────────────────────────────────────────────────────────────────────────
    // 0) Turn off all fees so only share accounting matters
    // ─────────────────────────────────────────────────────────────────────────────
    vm.startPrank(alOwner);
    alchemist.setProtocolFee(0);
    alchemist.setRepaymentFee(0);
    alchemist.setLiquidatorFee(0);
    vm.stopPrank();


    console.log("=== CONFIG ===");
    console.log("ProtocolFee     (bps):", alchemist.protocolFee());
    console.log("RepaymentFee    (bps):", alchemist.repaymentFee());
    console.log("LiquidatorFee   (bps):", alchemist.liquidatorFee());
    console.log("");


    // ─────────────────────────────────────────────────────────────────────────────
    // 1) Borrower A: deposit & mint near minimum collateralization
    // ─────────────────────────────────────────────────────────────────────────────
    uint256 amount = 100e18;


    vm.startPrank(address(0xBEEF));
    SafeERC20.safeApprove(address(vault), address(alchemist), amount);
    alchemist.deposit(amount, address(0xBEEF), 0);
    uint256 tokenIdA = AlchemistNFTHelper.getFirstTokenId(address(0xBEEF), address(alchemistNFT));
    uint256 maxMintA = alchemist.totalValue(tokenIdA) * FIXED_POINT_SCALAR / alchemist.minimumCollateralization();
    alchemist.mint(tokenIdA, maxMintA, address(0xBEEF));
    vm.stopPrank();


    console.log("=== SETUP ===");
    {
        (uint256 coll,,) = alchemist.getCDP(tokenIdA);
        console.log("A.collateral (MYT shares):", coll);
        console.log("A.maxMint (debt):", maxMintA);
    }
    console.log("");


    // ─────────────────────────────────────────────────────────────────────────────
    // 2) Create a redemption so earmarks accrue
    // ─────────────────────────────────────────────────────────────────────────────
    uint256 totalDebtBefore = alchemist.totalDebt();
    vm.startPrank(address(0xDAD));
    IERC20(address(alToken)).approve(address(transmuterLogic), totalDebtBefore / 2);
    transmuterLogic.createRedemption(totalDebtBefore / 2);
    vm.stopPrank();


    // Let earmarks build
    vm.roll(block.number + (5_256_000 / 2));
    alchemist.poke(tokenIdA);


    {
        (,, uint256 earmarkA) = alchemist.getCDP(tokenIdA);
        console.log("Earmarks built. A.earmarked (debt):", earmarkA);
    }
    console.log("");


    // ─────────────────────────────────────────────────────────────────────────────
    // 3) Price drop to make A liquidatable (use ~+5.9% supply bump)
    // ─────────────────────────────────────────────────────────────────────────────
    uint256 initialSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialSupply);
    uint256 modifiedSupply = (initialSupply * 590 / 10_000) + initialSupply; // +5.9%
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedSupply);


    console.log("=== PRICE ACTION ===");
    console.log("Yield token supply (initial):", initialSupply);
    console.log("Yield token supply (modified):", modifiedSupply);
    console.log("");


    // ─────────────────────────────────────────────────────────────────────────────
    // 4) Snapshot the "Transmuter denominator" BEFORE liquidation:
    //    denom = Alchemist TVL (based on _mytSharesDeposited) + underlying(transmuter MYT)
    // ─────────────────────────────────────────────────────────────────────────────
    uint256 transBalBefore = IERC20(address(vault)).balanceOf(address(transmuterLogic));
    uint256 tvlBefore      = alchemist.getTotalUnderlyingValue();
    uint256 denomBefore    = tvlBefore + alchemist.convertYieldTokensToUnderlying(transBalBefore);
    uint256 alchBalBefore  = IERC20(address(vault)).balanceOf(address(alchemist));


    console.log("=== BEFORE LIQUIDATION ===");
    console.log("Alchemist.getTotalUnderlyingValue() (tvlBefore):", tvlBefore);
    console.log("Transmuter MYT shares (transBalBefore):", transBalBefore);
    console.log("Transmuter MYT underlying:", alchemist.convertYieldTokensToUnderlying(transBalBefore));
    console.log("denomBefore = TVL + transmuterUnderlying:", denomBefore);
    console.log("Alchemist MYT ERC20 balance (shares) alchBalBefore:", alchBalBefore);
    console.log("");


    // ─────────────────────────────────────────────────────────────────────────────
    // 5) Liquidate A → MYT moves from Alchemist to Transmuter but _mytSharesDeposited
    //    is NOT decremented (buggy paths: _forceRepay / _doLiquidation)
    // ─────────────────────────────────────────────────────────────────────────────
    vm.prank(externalUser);
    (uint256 assets, uint256 feeYield, uint256 feeUnderlying) = alchemist.liquidate(tokenIdA);
    require(feeYield == 0 && feeUnderlying == 0, "fees must be zero for this POC");


    // ─────────────────────────────────────────────────────────────────────────────
    // 6) Observe post-state and prove double counting
    // ─────────────────────────────────────────────────────────────────────────────
    uint256 transBalAfter  = IERC20(address(vault)).balanceOf(address(transmuterLogic));
    uint256 alchBalAfter   = IERC20(address(vault)).balanceOf(address(alchemist));
    uint256 tvlAfter       = alchemist.getTotalUnderlyingValue(); // uses _mytSharesDeposited (stale, bug)
    uint256 denomAfter     = tvlAfter + alchemist.convertYieldTokensToUnderlying(transBalAfter);
    uint256 deltaTrans     = transBalAfter - transBalBefore;


    console.log("=== AFTER LIQUIDATION ===");
    console.log("assets moved to Transmuter (MYT shares):", assets);
    console.log("feeYield:", feeYield, " feeUnderlying:", feeUnderlying);
    console.log("Transmuter MYT shares (transBalAfter):", transBalAfter);
    console.log("Delta Transmuter MYT shares (deltaTrans):", deltaTrans);
    console.log("Alchemist MYT ERC20 balance (shares) alchBalAfter:", alchBalAfter);
    console.log("Delta Alchemist MYT ERC20 balance (shares):", int256(alchBalAfter) - int256(alchBalBefore));
    console.log("");


    console.log("TVL (from _mytSharesDeposited) tvlBefore:", tvlBefore);
    console.log("TVL (from _mytSharesDeposited) tvlAfter :", tvlAfter);
    console.log("TVL unchanged B (BUG symptom) ", tvlAfter == tvlBefore);
    console.log("");


    uint256 transUnderlyingBefore = alchemist.convertYieldTokensToUnderlying(transBalBefore);
    uint256 transUnderlyingAfter  = alchemist.convertYieldTokensToUnderlying(transBalAfter);
    uint256 deltaUnderlying       = transUnderlyingAfter - transUnderlyingBefore;


    console.log("Transmuter underlying before:", transUnderlyingBefore);
    console.log("Transmuter underlying after :", transUnderlyingAfter);
    console.log("Delta Transmuter underlying:", deltaUnderlying);
    console.log("");


    console.log("denomBefore (TVL + transUnderlying):", denomBefore);
    console.log("denomAfter  (TVL + transUnderlying):", denomAfter);
    console.log("Delta denom:", denomAfter - denomBefore);
    console.log("Expected Delta denom (underlying(deltaTrans)):", deltaUnderlying);
    console.log("Double count present? ", (denomAfter - denomBefore) == deltaUnderlying);
    console.log("");


    // Assertions that correspond to the printed narrative:
    assertEq(alchBalBefore - alchBalAfter, deltaTrans, "Alchemist ERC20 balance should drop by the shares moved");
    assertEq(tvlAfter, tvlBefore, "BUG: TVL (based on _mytSharesDeposited) stayed the same after shares left");
    // 6c) Denominator used by Transmuter increases by the underlying value of deltaTrans,
    //     even though total system underlying has not increases, this is  double counting
    assertApproxEqAbs(denomAfter - denomBefore, deltaUnderlying, 1, "BUG: Denominator bumped exactly by moved underlying (double counting)");
}
```
