# 58086 sc high mis accounting of myt outflows inflates tvl distorts collateralization and can dos deposits liquidations

**Submitted on Oct 30th 2025 at 14:51:58 UTC by @Freescore for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58086
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The AlchemistV3 contract tracks MYT collateral via an internal counter `_mytSharesDeposited` which is used to compute TVL and enforce the deposit cap. Several MYT outflow paths omit decrementing this counter (e.g., forced repayment and liquidation), leaving TVL overstated and causing system-wide effects: distorted global collateralization for liquidation math, incorrect redemption scaling, and premature deposit-cap rejections (DoS) for new deposits.

## Vulnerability Details

AlchemistV3 reports TVL as the underlying value of `_mytSharesDeposited`:

* TVL: `getTotalUnderlyingValue()` -> `convertYieldTokensToUnderlying(_mytSharesDeposited)` (src/AlchemistV3.sol:1239)
* Deposit cap enforcement: `_mytSharesDeposited + amount <= depositCap` (src/AlchemistV3.sol:369)

However, `_mytSharesDeposited` is not reduced when the contract transfers MYT out in several flows:

* Forced repayment (MYT to Transmuter, protocol fee to fee receiver):
  * MYT to Transmuter: src/AlchemistV3.sol:777–780
  * Protocol fee transfer: src/AlchemistV3.sol:771–775
  * No decrement of `_mytSharesDeposited`.
* Liquidation (MYT to Transmuter and to liquidator as base fee):
  * Transfers: src/AlchemistV3.sol:874–881
  * No decrement of `_mytSharesDeposited`.
* Repayment fee payout during liquidation (liquidator fee in MYT):
  * Fee transfer: src/AlchemistV3.sol:825–841 (post `_resolveRepaymentFee`)
  * No decrement of `_mytSharesDeposited`.

By contrast, other paths adjust `_mytSharesDeposited` correctly:

* deposit: +amount (src/AlchemistV3.sol:383)
* withdraw: −amount (src/AlchemistV3.sol:410)
* redeem (Transmuter): −(redeemed + feeCollateral) (src/AlchemistV3.sol:638)
* burn/repay protocol fee in MYT: (src/AlchemistV3.sol:485, 541)

Because TVL derives from `_mytSharesDeposited`, real outflows go unreflected in TVL, overstating system collateral and breaking downstream logic.

## Impact Details

* Overstated TVL distorts global collateralization used in liquidation math:
  * Liquidation inputs use `normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt` (src/AlchemistV3.sol:862).
  * Inflated TVL can misclassify liquidation environment (e.g., avoid outsourced fee branch, miscompute debtToBurn), degrading risk controls.
* Deposit-cap DoS:
  * New deposits are blocked when `_mytSharesDeposited + amount > depositCap` even if real on-chain balance allows, because `_mytSharesDeposited` remained incorrectly high while ERC20 balance decreased on outflows.
* Inaccurate redemption haircuts:
  * Transmuter bad-debt scaling relies on Alchemist’s total underlying value. Inflated TVL can understate bad debt, paying out more to redeemers than appropriate, risking solvency.
* Governance cap mismatch:
  * `setDepositCap` checks live ERC20 balance (src/AlchemistV3.sol:236–241), while enforcement uses `_mytSharesDeposited`. If outflows aren’t accounted, admin can set a cap lower than `_mytSharesDeposited` (because balance dropped), creating a contradictory state that blocks deposits.

Overall severity: Medium-High. The issue is deterministic and reachable during normal operations (forced repay, liquidation) and can produce persistent DoS on deposits, mis-liquidations, and mispriced redemptions.

## References

* TVL calculation: src/AlchemistV3.sol:1239
* Deposit cap enforcement: src/AlchemistV3.sol:369
* Forced repay MYT outflow: src/AlchemistV3.sol:771–780
* Liquidation MYT outflows: src/AlchemistV3.sol:874–881
* Correct decrement examples: deposit/withdraw/redeem/burn-fee (src/AlchemistV3.sol:383, 410, 638, 485, 541)
* Liquidation collateralization input: src/AlchemistV3.sol:862

## Proof of Concept

## Proof of Concept

Two Foundry tests demonstrate the bug:

1. Forced repayment outflow (no fees):
   * Name: `testMisAccountingOnOutflows_ForceRepay()`
   * Steps:
     * Set protocol and repayment fees to 0; set `collateralizationLowerBound = minimumCollateralization`.
     * User deposits and mints to minimum collateralization.
     * Create and mature a Transmuter redemption equal to the debt (fully earmarked).
     * Record:
       * `tvlBefore = alchemist.getTotalUnderlyingValue()`
       * `mytBalBefore = IERC20(vault).balanceOf(alchemist)`
     * Call `alchemist.liquidate(tokenId)`; only a forced repayment occurs.
     * Assert:
       * `outflow = mytBalBefore - mytBalAfter ≈ yieldAmount` (MYT left to Transmuter)
       * `tvlAfter ≈ tvlBefore` (TVL unchanged despite ERC20 outflow)

```solidity
        /// Demonstrates mis-accounting on MYT outflows: Alchemist transfers MYT out
    /// during forced repayment but does not decrement its internal `_mytSharesDeposited`,
    /// so `getTotalUnderlyingValue()` remains unchanged while the ERC20 balance drops.
    function testMisAccountingOnOutflows_ForceRepay() external {
        // No protocol or repayment fees to isolate the outflow to the transmuter transfer only
        vm.startPrank(alOwner);
        alchemist.setProtocolFee(0);
        alchemist.setRepaymentFee(0);
        // Raise liquidation sensitivity so a position at min collateralization triggers forced repay
        alchemist.setCollateralizationLowerBound(alchemist.minimumCollateralization());
        vm.stopPrank();

        // User deposits and mints to minimum collateralization
        uint256 depositAmount = 100e18;
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        uint256 mintAmount = alchemist.totalValue(tokenId) * FIXED_POINT_SCALAR / alchemist.minimumCollateralization();
        alchemist.mint(tokenId, mintAmount, address(0xbeef));
        vm.stopPrank();

        // Create a transmuter redemption to fully earmark debtor's debt and mature it
        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
        transmuterLogic.createRedemption(mintAmount);
        vm.stopPrank();
        vm.roll(block.number + 5_256_000);

        // Record protocol TVL (based on _mytSharesDeposited) and actual MYT balance before
        uint256 tvlBefore = alchemist.getTotalUnderlyingValue();
        uint256 mytBalBefore = IERC20(address(vault)).balanceOf(address(alchemist));

        // Liquidate -> forces a repayment from earmarked debt only (no liquidation since debt cleared)
        vm.startPrank(externalUser);
        (uint256 yieldAmount, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenId);
        vm.stopPrank();

        // No fees expected; pure repayment in MYT to transmuter
        assertGt(yieldAmount, 0, "expected repayment outflow");
        assertEq(feeInYield, 0, "unexpected fee in yield");
        assertEq(feeInUnderlying, 0, "unexpected fee in underlying");

        // Actual ERC20 MYT balance decreased by the repayment outflow
        uint256 mytBalAfter = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 outflow = mytBalBefore - mytBalAfter;
        assertApproxEqAbs(outflow, yieldAmount, 1, "erc20 outflow should equal repayment outflow");

        // But TVL based on _mytSharesDeposited remains unchanged (mis-accounting)
        uint256 tvlAfter = alchemist.getTotalUnderlyingValue();
        assertApproxEqAbs(tvlAfter, tvlBefore, 1, "TVL should remain unchanged despite MYT outflow");
    }
```

2. Liquidation with base fee (no protocol/repayment fees):
   * Name: `testMisAccountingOnOutflows_Liquidation()`
   * Steps:
     * Set liquidator fee > 0; set protocol and repayment fees to 0.
     * User deposits and mints; drop MYT price to breach lower bound.
     * Record TVL and ERC20 MYT balance.
     * Call `alchemist.liquidate(tokenId)`.
     * Assert:
       * `yieldAmount > 0`
       * `outflow = mytBalBefore - mytBalAfter ≈ yieldAmount` (MYT sent to Transmuter + fee to liquidator)
       * `tvlAfter ≈ tvlBefore` (TVL unchanged despite MYT outflow)

```solidity
 /// Demonstrates mis-accounting on MYT outflows during a liquidation with a base fee:
    /// Alchemist transfers MYT to Transmuter (net of fee) and to liquidator (fee),
    /// actual ERC20 MYT balance decreases by yieldAmount, but TVL (based on _mytSharesDeposited) remains unchanged.
    function testMisAccountingOnOutflows_Liquidation() external {
        // Configure fees: liquidator fee > 0; no protocol or repayment fees
        vm.startPrank(alOwner);
        alchemist.setProtocolFee(0);
        alchemist.setRepaymentFee(0);
        alchemist.setLiquidatorFee(300); // 3%
        vm.stopPrank();

        // Debtor deposits and mints to minimum collateralization
        uint256 depositAmount = 100e18;
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        uint256 mintAmount = alchemist.totalValue(tokenId) * FIXED_POINT_SCALAR / alchemist.minimumCollateralization();
        alchemist.mint(tokenId, mintAmount, address(0xbeef));
        vm.stopPrank();

        // No earmark created; pure liquidation path (no forced repay first)
        // Drop MYT share price sufficiently to breach lower bound collateralization
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply * 20);

        // Record TVL and actual ERC20 MYT balance before
        uint256 tvlBefore = alchemist.getTotalUnderlyingValue();
        uint256 mytBalBefore = IERC20(address(vault)).balanceOf(address(alchemist));

        // Liquidate and collect outputs
        vm.startPrank(externalUser);
        (uint256 yieldAmount, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenId);
        vm.stopPrank();

        // Expect liquidation (not only repay): some MYT leaves to the transmuter; fee may be in yield or outsourced in underlying
        assertGt(yieldAmount, 0, "expected liquidation outflow");

        // Actual ERC20 balance decreased by MYT outflow = yieldAmount
        uint256 mytBalAfter = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 outflow = mytBalBefore - mytBalAfter;
        assertApproxEqAbs(outflow, yieldAmount, 1, "erc20 outflow should equal function-reported yieldAmount");

        // TVL remains unchanged due to missing _mytSharesDeposited decrement
        uint256 tvlAfter = alchemist.getTotalUnderlyingValue();
        assertApproxEqAbs(tvlAfter, tvlBefore, 1, "TVL should remain unchanged despite liquidation outflow");
    }

```

These tests are included in `src/test/AlchemistV3.t.sol` and pass, proving that MYT leaves the Alchemist but TVL (computed from `_mytSharesDeposited`) does not reflect the outflow.
