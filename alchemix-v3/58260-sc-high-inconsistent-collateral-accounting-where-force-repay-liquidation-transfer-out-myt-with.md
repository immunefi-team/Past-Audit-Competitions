# 58260 sc high inconsistent collateral accounting where force repay liquidation transfer out myt without adjusting tvl

**Submitted on Oct 31st 2025 at 19:35:03 UTC by @Anirruth for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58260
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds
  * Protocol insolvency

## Description

## Brief/Intro

When Alchemist V3 sends MYT out of the contract during force-repay and liquidation flows, it does not decrement \_mytSharesDeposited unlike the redeem function. Because getTotalUnderlyingValue() is computed from \_mytSharesDeposited, the system overstates TVL after such transfers. This misreporting weakens liquidation thresholds, can under-liquidate risky positions.

## Vulnerability Details

Intended behavior (consistent path: redeem): When MYT leaves the Alchemist during redemptions, \_mytSharesDeposited is decremented, keeping TVL aligned with the actual share balance.

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi#L636-L638>

```
TokenUtils.safeTransfer(myt, transmuter, collRedeemed);
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, feeCollateral);
        _mytSharesDeposited -= collRedeemed + feeCollateral;
```

Actual behavior (inconsistent paths): During force-repay and liquidation, the contract transfers MYT out but does not adjust \_mytSharesDeposited.

In forcerepay :

* <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi#L771-L780>

In liquidation:

* <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi#L824-L828>
* <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi#L839-L840>

```
if (account.collateralBalance > protocolFeeTotal) {
            account.collateralBalance -= protocolFeeTotal;
            // Transfer the protocol fee to the protocol fee receiver
            TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
        }

        if (creditToYield > 0) {
            // Transfer the repaid tokens from the account to the transmuter.
            TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
        }
```

Therefore, after a force-repay or liquidation, the contract’s actual MYT balance decreases but TVL (based on \_mytSharesDeposited) remains unchanged, creating an accounting drift.

## Impact Details

Misreported TVL inflates the system’s perceived collateralization and feeds directly into liquidation math:

* Liquidation computes a “global collateralization” input using \_getTotalUnderlyingValue(); overstated TVL makes the system appear healthier than it is, reducing liquidation severity or preventing needed liquidations.

Under-liquidation of risky positions increases the chance that price moves or further redemptions push the system into bad debt.

## References

* <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi#L636-L638>
* <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi#L824-L828>
* <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi#L839-L840>

## Proof of Concept

## Proof of Concept

Paste the following test in AlchemistV3.t.sol and run the command `forge test --match-test test_MytSharesDeposited_NotDecremented_OnForceRepayOrLiquidation`

```
function test_MytSharesDeposited_NotDecremented_OnForceRepayOrLiquidation() external {
        // 1) User deposits MYT (vault shares) and mints to the max so CR == minimumCollateralization
        uint256 deposit = 1_000e18;
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), deposit);
        alchemist.deposit(deposit, address(0xbeef), 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));

        // Mint to the maximum borrowable so initial ratio == lower bound threshold
        uint256 maxBorrow = alchemist.getMaxBorrowable(tokenId);
        // Ensure we are allowed to mint
        alchemist.mint(tokenId, maxBorrow, address(0xbeef));
        vm.stopPrank();

        // Sanity: system state matches MYT balance and TVL
        uint256 balBefore = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 tvlBefore = alchemist.getTotalUnderlyingValue();
        assertGt(balBefore, 0);
        // Before any bug-triggering path, TVL should mirror MYT balance
        assertEq(tvlBefore, alchemist.convertYieldTokensToUnderlying(balBefore));

        // 2) Make the position undercollateralized by reducing share price (increase mock yield token supply)
        // This ensures liquidation executes and transfers MYT out of Alchemist.
        _manipulateYieldTokenPrice(1000); // ~10% supply increase -> share price down sufficiently

        // 3) Trigger liquidation, which always performs _earmark() + _sync() and then:
        //    - repays earmarked via _forceRepay (sending MYT from Alchemist to Transmuter), and
        //    - if still under threshold, also performs _doLiquidation (more outbound MYT transfers).
        (uint256 amountLiquidated,,) = alchemist.liquidate(tokenId);

        // 4) After liquidation/force-repay, the Alchemist contract sent out MYT but did NOT decrement _mytSharesDeposited.
        //    Therefore, the actual MYT balance fell while getTotalUnderlyingValue() (based on _mytSharesDeposited) did not.
        uint256 balAfter = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 tvlAfter = alchemist.getTotalUnderlyingValue();

        // We must have transferred some MYT out via force-repay and/or liquidation.
        // If earmark was tiny, amountLiquidated can be zero yet force-repay still moved MYT; we assert on balance delta.
        assertLt(balAfter, balBefore, "Alchemist MYT balance should decrease after repay/liquidation");

        // Strict proof of bug: TVL (derived from _mytSharesDeposited) should now exceed what the contract actually holds.
        uint256 underlyingFromActualBal = alchemist.convertYieldTokensToUnderlying(balAfter);
        assertGt(tvlAfter, underlyingFromActualBal, "TVL based on _mytSharesDeposited should be greater than underlying from actual balance");
    }
```
