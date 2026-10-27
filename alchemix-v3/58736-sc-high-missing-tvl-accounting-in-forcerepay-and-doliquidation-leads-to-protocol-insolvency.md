# 58736 sc high missing tvl accounting in forcerepay and doliquidation leads to protocol insolvency

**Submitted on Nov 4th 2025 at 10:33:19 UTC by @dray for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58736
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The `_forceRepay` and `_doLiquidation` functions in `AlchemistV3.sol` transfer MYT shares out of the contract during liquidations but fail to update the global `_mytSharesDeposited` counter that tracks total value locked (TVL). This accounting omission causes the protocol's internal TVL to permanently diverge from actual on-chain balances, creating phantom collateral that doesn't exist. Over time, as liquidations accumulate, the protocol becomes insolvent—unable to fulfill all legitimate withdrawal and redemption claims despite appearing healthy in its accounting. This silent deterioration eventually leads to failed withdrawals for later users and deposit-cap denial-of-service as the inflated TVL fills the deposit cap with non-existent collateral.

## Vulnerability Details

### Root Cause

The protocol maintains a critical state variable `_mytSharesDeposited` that tracks the total MYT shares held by the Alchemist contract:

```solidity
// Line 134
uint256 private _mytSharesDeposited;
```

This variable is used to calculate the protocol's total TVL:

```solidity
// Lines 1237-1241
function _getTotalUnderlyingValue() internal view returns (uint256 totalUnderlyingValue) {
    uint256 yieldTokenTVLInUnderlying = convertYieldTokensToUnderlying(_mytSharesDeposited);
    totalUnderlyingValue = yieldTokenTVLInUnderlying;
}
```

The variable is correctly updated in most operations:

* **Incremented** on deposits (line 383)
* **Decremented** on withdrawals (line 410)
* **Decremented** on burn operations for protocol fees (lines 485, 541)
* **Decremented** on redemptions (line 638)

However, two critical liquidation paths fail to decrement this counter when transferring shares out:

### Vulnerable Path 1: `_forceRepay`

In `AlchemistV3.sol` lines 738-780, when forced repayment occurs:

```solidity
function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
    // ... debt calculation logic ...
    
    uint256 creditToYield = convertDebtTokensToYield(credit);
    
    // ... earmark removal ...
    
    creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield;
    account.collateralBalance -= creditToYield;  // ✅ User balance updated
    
    uint256 protocolFeeTotal = creditToYield * protocolFee / BPS;
    
    emit ForceRepay(accountId, amount, creditToYield, protocolFeeTotal);
    
    if (account.collateralBalance > protocolFeeTotal) {
        account.collateralBalance -= protocolFeeTotal;
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
        // Missing: _mytSharesDeposited -= protocolFeeTotal;
    }
    
    if (creditToYield > 0) {
        // Transfer the repaid tokens to the transmuter
        TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
        // Missing: _mytSharesDeposited -= creditToYield;
    }
    
    return creditToYield;
}
```

**The Bug:** While `creditToYield` shares are transferred to the transmuter (line 778) and potentially `protocolFeeTotal` shares to the fee receiver (line 775), `_mytSharesDeposited` is never decremented. The contract balance decreases, but the internal accounting remains unchanged.

### Vulnerable Path 2: `_doLiquidation`

Similarly, in `_doLiquidation` at lines 843-890:

```solidity
function _doLiquidation(uint256 accountId, uint256 collateralInUnderlying, uint256 repaidAmountInYield) 
    internal returns (uint256, uint256, uint256) 
{
    // ... liquidation calculation ...
    
    amountLiquidated = convertDebtTokensToYield(liquidationAmount);
    feeInYield = convertDebtTokensToYield(baseFee);
    
    // ... account updates ...
    
    // Transfer liquidated collateral to liquidator
    if (amountLiquidated > 0) {
        TokenUtils.safeTransfer(myt, msg.sender, amountLiquidated);
        // Missing: _mytSharesDeposited -= amountLiquidated;
    }
    
    // Transfer fee to liquidator
    if (feeInYield > 0) {
        TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
        //  Missing: _mytSharesDeposited -= feeInYield;
    }
    
    // Outsource fee from fee vault if needed
    if (outsourcedFee > 0) {
        // ... fee vault logic ...
        // Missing: _mytSharesDeposited -= amountFromVault (if applicable);
    }
    
    return (repaidAmountInYield + amountLiquidated, feeInYield, outsourcedFee);
}
```

## Impact Details

\###1. Direct Protocol Insolvency: Where the TVL drift creates a scenario where the protocol cannot honor all user claims.

#### 2. Collateralization Ratio Manipulation

## References

**Primary Issues:**

* [`src/AlchemistV3.sol#L738-780`](file:///9276110/src/AlchemistV3.sol#L738-780) - `_forceRepay` missing TVL decrements
* [`src/AlchemistV3.sol#L843-890`](file:///9276110/src/AlchemistV3.sol#L843-890) - `_doLiquidation` missing TVL decrements

**TVL State Variable:**

* [`src/AlchemistV3.sol#L134`](file:///9276110/src/AlchemistV3.sol#L134) - `_mytSharesDeposited` declaration
* [`src/AlchemistV3.sol#L1237-1241`](file:///9276110/src/AlchemistV3.sol#L1237-1241) - TVL calculation using `_mytSharesDeposited`

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.28;

import {AlchemistV3Test} from "./AlchemistV3.t.sol";
import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {IVaultV2} from "../../lib/vault-v2/src/interfaces/IVaultV2.sol";
import {IMockYieldToken} from "./mocks/MockYieldToken.sol";
import {SafeERC20} from "../libraries/SafeERC20.sol";
import {AlchemistNFTHelper} from "./libraries/AlchemistNFTHelper.sol";

/// @notice PoC showing `_forceRepay` leaks TVL by not adjusting `_mytSharesDeposited` after
///         transferring MYT shares out of the Alchemist contract.
contract ForceRepayTVLDriftPoC is AlchemistV3Test {
    function testForceRepayLeavesPhantomTVL() external {
        // Eliminate repayment-fee side effects so we isolate the TVL drift from _forceRepay itself.
        vm.prank(alOwner);
        alchemist.setRepaymentFee(0);

        // Helper deposit keeps protocol solvent and ensures there are extra shares in the system.
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        // Borrower deposits and mints the maximum possible debt.
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        uint256 maxBorrowable = alchemist.getMaxBorrowable(tokenId);
        alchemist.mint(tokenId, maxBorrowable, address(0xbeef));
        vm.stopPrank();

        // Redeemer earmarks the full borrower debt so liquidation triggers the _forceRepay path.
        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), maxBorrowable);
        transmuterLogic.createRedemption(maxBorrowable);
        vm.stopPrank();

        // Allow redemption window to elapse and sync borrower so all debt is earmarked.
        vm.roll(block.number + 5_256_000);
        vm.prank(address(0xbeef));
        alchemist.poke(tokenId);

        // Slash the yield token price so the position becomes liquidatable.
        uint256 supply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(supply * 2);
        vm.prank(address(0xbeef));
        alchemist.poke(tokenId);

        // Sanity: recorded TVL should align with actual vault holdings before liquidation.
        uint256 sharesBefore = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 reportedBefore = alchemist.getTotalUnderlyingValue();
        uint256 onChainBefore = IVaultV2(vault).convertToAssets(sharesBefore);
        assertApproxEqAbs(reportedBefore, onChainBefore, 5);

        // Liquidator calls liquidation; with full earmark this only executes _forceRepay + fee payout.
        vm.startPrank(externalUser);
        (uint256 repaidYield, uint256 feeInYield,) = alchemist.liquidate(tokenId);
        vm.stopPrank();
        assertEq(feeInYield, 0, "repayment fee disabled for this PoC");
        assertGt(repaidYield, 0, "expected forced repayment to occur");

        // Shares actually left the contract...
        uint256 sharesAfter = IERC20(address(vault)).balanceOf(address(alchemist));
        assertEq(sharesBefore - sharesAfter, repaidYield, "shares transferred to transmuter");

        // ...but `_mytSharesDeposited` never updated, so reported TVL stays constant.
        uint256 reportedAfter = alchemist.getTotalUnderlyingValue();
        assertEq(reportedAfter, reportedBefore, "phantom shares still counted in TVL");

        // Ground truth TVL dropped by the amount repaid to the transmuter.
        uint256 onChainAfter = IVaultV2(vault).convertToAssets(sharesAfter);
        assertLt(onChainAfter, reportedAfter, "actual assets lower than reported TVL");

        uint256 drift = reportedAfter - onChainAfter;
        uint256 repaidUnderlying = IVaultV2(vault).convertToAssets(repaidYield);
        assertApproxEqAbs(drift, repaidUnderlying, 5, "missing TVL equals repaid amount");
    }

}
```
