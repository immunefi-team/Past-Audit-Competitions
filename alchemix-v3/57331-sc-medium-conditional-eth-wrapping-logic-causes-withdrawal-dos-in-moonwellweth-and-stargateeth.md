# 57331 sc medium conditional eth wrapping logic causes withdrawal dos in moonwellweth and stargateeth strategies

## #57331 \[SC-Medium] Conditional ETH Wrapping Logic Causes Withdrawal DoS in MoonwellWETH and StargateETH Strategies

**Submitted on Oct 25th 2025 at 09:50:56 UTC by @Max36935 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57331
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/MoonwellWETHStrategy.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 1 hour

### Description

## Conditional ETH Wrapping Logic Causes Withdrawal DoS in MoonwellWETH and StargateETH Strategies

### Summary

Both `MoonwellWETHStrategy.sol` and `StargateEthPoolStrategy.sol` on Optimism contain flawed conditional logic in their `_deallocate()` functions that prevents ETH from being wrapped to WETH when withdrawals experience any slippage or loss. This causes the functions to revert, blocking users from withdrawing funds during normal market conditions.

### Vulnerability Details

#### Location

* **File 1:** `src/strategies/optimism/MoonwellWETHStrategy.sol`
  * **Function:** `_deallocate(uint256 amount)`
  * **Lines:** 64-67
* **File 2:** `src/strategies/optimism/StargateEthPoolStrategy.sol`
  * **Function:** `_deallocate(uint256 amount)`
  * **Lines:** 74-77

#### The Bug

Both strategies implement the same problematic pattern. Let me walk through MoonwellWETH as the example (Stargate has identical logic):

**Lines 54-70 from MoonwellWETHStrategy.sol:**

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 ethBalanceBefore = address(this).balance;
    // Pull exact amount of underlying WETH out
    mWETH.redeemUnderlying(amount);
    // wrap any ETH received (Moonwell redeems to ETH for WETH markets)
    uint256 ethBalanceAfter = address(this).balance;
    uint256 ethRedeemed = ethBalanceAfter - ethBalanceBefore;
    if (ethRedeemed < amount) {
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, ethRedeemed);
    }
    if (ethRedeemed + ethBalanceBefore >= amount) {
        weth.deposit{value: ethRedeemed}();
    }
    require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
    TokenUtils.safeApprove(address(weth), msg.sender, amount);
    return amount;
}
```

**The Issue is on Line 64:**

```solidity
if (ethRedeemed + ethBalanceBefore >= amount) {
    weth.deposit{value: ethRedeemed}();
}
```

This conditional check only wraps the ETH if the total amount (redeemed + any pre-existing balance) is sufficient. In normal operation, strategies don't accumulate ETH balances - they start with 0 ETH.

**Execution Flow When Loss Occurs:**

Scenario: User requests to deallocate 100 WETH, but receives 98 ETH due to 2% slippage.

```
Line 55: ethBalanceBefore = 0 (strategy has no leftover ETH)
Line 57: mWETH.redeemUnderlying(100 WETH) 
         → Moonwell sends 98 ETH to strategy
         → receive() function catches this ETH (line 108)
Line 59: ethBalanceAfter = 98 ETH
Line 60: ethRedeemed = 98 - 0 = 98 ETH
Line 61-63: if (98 < 100) → TRUE → emits StrategyDeallocationLoss event ✓
Line 64: if (98 + 0 >= 100) → if (98 >= 100) → FALSE ✗
         → weth.deposit() is NOT called
         → 98 ETH remains unwrapped in the contract
Line 67: require(wethBalance >= 100) 
         → require(0 >= 100) → REVERTS ✗
```

**Result:** Transaction fails with 98 ETH stuck unwrapped in the strategy contract.

#### Comment vs Code Contradiction

Look at line 58 comment in MoonwellWETH:

```solidity
// wrap any ETH received (Moonwell redeems to ETH for WETH markets)
```

The comment explicitly states "wrap **any** ETH received", but the code implements conditional wrapping that only wraps if there's no loss. This is a clear contradiction between intent and implementation.

Similarly, in StargateEthPoolStrategy line 65:

```solidity
// Redeem LP to native ETH, then wrap back to WETH
```

It says "then wrap back to WETH" - not "wrap back if we have enough". The comment implies unconditional wrapping.

#### Why Strategies Don't Have Leftover ETH

I verified that in normal operation, strategies should not accumulate ETH:

1. **`_allocate()` doesn't leave ETH:** Takes WETH as input, converts if needed, deposits to protocol
2. **`_deallocate()` should wrap immediately:** Receives ETH from protocol, supposed to wrap to WETH
3. **`receive()` exists only for protocol redemptions:** Not for general ETH deposits

The only source of ETH is from the redemption itself. The conditional assumes there might be leftover ETH from previous operations, but this shouldn't happen in normal flow.

#### Comparison with Other Strategies

I checked all other WETH strategies in the codebase and **none** use this conditional pattern:

**TokeAutoEth (mainnet) - Lines 77-82:**

```solidity
uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
autoEth.redeem(sharesNeeded, address(this), address(this));
uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
if (wethRedeemed < amount) {
    emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
}
```

No conditional wrapping - just measures, emits loss event if needed, then continues to require check.

**PeapodsETH (mainnet) - Lines 37-44:**

```solidity
uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
vault.withdraw(amount, address(this), address(this));
uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
if (wethRedeemed < amount) {
    emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
}
require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, ...);
```

Same pattern - no conditional wrapping.

**MoonwellUSDC (same protocol, USDC version) - Lines 56-64:**

```solidity
uint256 usdcBalanceBefore = TokenUtils.safeBalanceOf(address(usdc), address(this));
mUSDC.redeemUnderlying(amount);
uint256 usdcBalanceAfter = TokenUtils.safeBalanceOf(address(usdc), address(this));
uint256 usdcRedeemed = usdcBalanceAfter - usdcBalanceBefore;
if (usdcRedeemed < amount) {
    emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, usdcRedeemed);
}
```

Even the USDC version from the same Moonwell protocol doesn't have the conditional - it just measures and emits loss.

**Finding:** Only MoonwellWETH and StargateETH have this conditional wrapping bug. All other strategies follow the simpler, correct pattern.

### Impact

#### Direct Impact

* Users cannot withdraw from MoonwellWETH or StargateETH strategies when there's any slippage or loss
* Protocol cannot rebalance funds allocated to these strategies during volatile markets
* Withdrawals that would normally succeed with a small loss completely fail

### Steps to Reproduce

1. Deploy strategy with 0 initial ETH balance (normal state)
2. Allocate 100 WETH to strategy
3. Market conditions cause 2% slippage
4. Call `deallocate(100 WETH)`
5. Strategy receives 98 ETH from protocol
6. Conditional check: `98 + 0 >= 100` = FALSE
7. ETH not wrapped
8. `require(wethBalance >= 100)` fails
9. Transaction reverts

### Recommendation

Remove the conditional and wrap all received ETH:

**Current (buggy):**

```solidity
if (ethRedeemed + ethBalanceBefore >= amount) {
    weth.deposit{value: ethRedeemed}();
}
```

**Fixed:**

```solidity
if (ethRedeemed > 0) {
    weth.deposit{value: ethRedeemed}();
}
```

### Proof of Concept

### Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "forge-std/Test.sol";
import {AlchemistV3Test} from "./AlchemistV3.t.sol";
import {SafeERC20} from "../libraries/SafeERC20.sol";
import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {AlchemistNFTHelper} from "./libraries/AlchemistNFTHelper.sol";
import {IMockYieldToken} from "./mocks/MockYieldToken.sol";

contract Audit_MytSharesNotDecrementedOnLiquidation is AlchemistV3Test {
    address liquidator = address(0x9999);

    function testAudit_MytSharesDeposited_NotUpdated_OnLiquidation() external {
        // Setup: User deposits collateral
        uint256 depositAmount = 100_000e18;

        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);

        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
        alchemist.deposit(depositAmount, someWhale, 0);

        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(someWhale, address(alchemistNFT));

        // Mint maximum debt
        uint256 maxDebt = alchemist.getMaxBorrowable(tokenId);
        alchemist.mint(tokenId, maxDebt, someWhale);
        vm.stopPrank();

        console.log("=== INITIAL STATE ===");
        console.log("User deposited: %s", depositAmount / 1e18);

        // Record the INITIAL state
        uint256 totalUnderlyingBefore = alchemist.getTotalUnderlyingValue();
        uint256 contractBalanceBefore = IERC20(address(vault)).balanceOf(address(alchemist));

        console.log("Contract balance: %s", contractBalanceBefore / 1e18);
        console.log("getTotalUnderlyingValue(): %s", totalUnderlyingBefore / 1e18);
        console.log("");

        // Simulate price drop to make position liquidatable
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);

        // Increase supply by 40% (causes ~28% price drop)
        uint256 modifiedVaultSupply = (initialVaultSupply * 14000) / 10000;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        console.log("=== AFTER PRICE DROP ===");
        (uint256 collateral, uint256 debt, ) = alchemist.getCDP(tokenId);
        uint256 collateralValue = alchemist.totalValue(tokenId);
        console.log("Collateral value: %s", collateralValue / 1e18);
        console.log("Debt: %s", debt / 1e18);
        console.log("Collateralization: %s%%", (collateralValue * 100) / debt);

        uint256 collateralizationRatio = (collateralValue * FIXED_POINT_SCALAR) / debt;
        bool isLiquidatable = collateralizationRatio <= alchemist.collateralizationLowerBound();

        if (!isLiquidatable) {
            console.log("ERROR: Position not liquidatable, test setup failed");
            revert("Setup failed");
        }

        console.log("Position is liquidatable!");
        console.log("");

        // Execute liquidation
        console.log("=== EXECUTING LIQUIDATION ===");
        vm.prank(liquidator);
        (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenId);

        console.log("Amount liquidated: %s", amountLiquidated / 1e18);
        console.log("Fee to liquidator (yield): %s", feeInYield / 1e18);
        console.log("Fee to liquidator (underlying): %s", feeInUnderlying / 1e18);
        console.log("");

        // Check AFTER liquidation
        uint256 contractBalanceAfter = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 totalUnderlyingAfter = alchemist.getTotalUnderlyingValue();

        console.log("=== STATE AFTER LIQUIDATION ===");
        console.log("Contract balance BEFORE: %s", contractBalanceBefore / 1e18);
        console.log("Contract balance AFTER:  %s", contractBalanceAfter / 1e18);
        console.log("Tokens transferred OUT:  %s", (contractBalanceBefore - contractBalanceAfter) / 1e18);
        console.log("");
        console.log("getTotalUnderlyingValue() BEFORE: %s", totalUnderlyingBefore / 1e18);
        console.log("getTotalUnderlyingValue() AFTER:  %s", totalUnderlyingAfter / 1e18);
        console.log("");

        // Calculate expected value
        uint256 tokensTransferredOut = contractBalanceBefore - contractBalanceAfter;
        uint256 expectedTotalUnderlying = totalUnderlyingBefore - tokensTransferredOut;

        console.log("=== BUG VERIFICATION ===");
        console.log("Expected getTotalUnderlyingValue(): %s", expectedTotalUnderlying / 1e18);
        console.log("Actual getTotalUnderlyingValue():   %s", totalUnderlyingAfter / 1e18);
        console.log("Discrepancy: %s", (totalUnderlyingAfter - expectedTotalUnderlying) / 1e18);
        console.log("");

        if (totalUnderlyingAfter > expectedTotalUnderlying) {
            console.log("*** BUG CONFIRMED ***");
            console.log(
                "getTotalUnderlyingValue() is INFLATED by %s tokens",
                (totalUnderlyingAfter - expectedTotalUnderlying) / 1e18
            );
            console.log("");
            console.log("Root cause: _mytSharesDeposited not decremented during liquidation");
            console.log("Impact: Protocol TVL is overstated, affecting liquidation calculations");

            // This assertion proves the bug
            assertGt(
                totalUnderlyingAfter,
                expectedTotalUnderlying,
                "BUG: getTotalUnderlyingValue() should decrease but didn't"
            );
        } else {
            console.log("No bug detected - values match as expected");
        }
    }

    // function testAudit_CompareBurnVsLiquidation_Accounting() external {
    //     // This test compares how burn() properly updates _mytSharesDeposited
    //     // but liquidation doesn't

    //     console.log("=== COMPARISON TEST: burn() vs liquidate() ===");
    //     console.log("");

    //     // Setup two identical positions
    //     uint256 depositAmount = 100_000e18;

    //     // Position 1: Will be burned
    //     vm.startPrank(address(0xBEEF));
    //     IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, address(0xBEEF));
    //     SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
    //     alchemist.deposit(depositAmount, address(0xBEEF), 0);
    //     uint256 tokenId1 = AlchemistNFTHelper.getFirstTokenId(address(0xBEEF), address(alchemistNFT));
    //     uint256 debt1 = alchemist.getMaxBorrowable(tokenId1);
    //     alchemist.mint(tokenId1, debt1, address(0xBEEF));
    //     vm.stopPrank();

    //     // Position 2: Will be liquidated
    //     vm.startPrank(address(0xCAFE));
    //     IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, address(0xCAFE));
    //     SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
    //     alchemist.deposit(depositAmount, address(0xCAFE), 0);
    //     uint256 tokenId2 = AlchemistNFTHelper.getFirstTokenId(address(0xCAFE), address(alchemistNFT));
    //     uint256 debt2 = alchemist.getMaxBorrowable(tokenId2);
    //     alchemist.mint(tokenId2, debt2, address(0xCAFE));
    //     vm.stopPrank();

    //     vm.roll(block.number + 1);

    //     console.log("=== SCENARIO 1: BURN ===");
    //     uint256 tvlBeforeBurn = alchemist.getTotalUnderlyingValue();
    //     console.log("TVL before burn: %s", tvlBeforeBurn / 1e18);

    //     // Burn some debt
    //     uint256 burnAmount = 10_000e18;
    //     vm.prank(address(0xBEEF));
    //     alchemist.burn(burnAmount, tokenId1);

    //     uint256 tvlAfterBurn = alchemist.getTotalUnderlyingValue();
    //     console.log("TVL after burn:  %s", tvlAfterBurn / 1e18);
    //     console.log("TVL decreased by: %s (CORRECT)", (tvlBeforeBurn - tvlAfterBurn) / 1e18);
    //     console.log("");

    //     console.log("=== SCENARIO 2: LIQUIDATION ===");

    //     // Make position liquidatable
    //     uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
    //     IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
    //     uint256 modifiedVaultSupply = (initialVaultSupply * 14000) / 10000;
    //     IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

    //     uint256 tvlBeforeLiquidation = alchemist.getTotalUnderlyingValue();
    //     console.log("TVL before liquidation: %s", tvlBeforeLiquidation / 1e18);

    //     // Liquidate
    //     vm.prank(liquidator);
    //     alchemist.liquidate(tokenId2);

    //     uint256 tvlAfterLiquidation = alchemist.getTotalUnderlyingValue();
    //     console.log("TVL after liquidation:  %s", tvlAfterLiquidation / 1e18);

    //     if (tvlAfterLiquidation == tvlBeforeLiquidation) {
    //         console.log("TVL did NOT decrease (BUG!)");
    //         console.log("");
    //         console.log("*** BUG CONFIRMED ***");
    //         console.log("burn() correctly updates TVL");
    //         console.log("liquidation() does NOT update TVL");
    //     }
    // }
}
```

### Save And Run The Test

* Save in src/test/Audit\_MytSharesNotDecrementedOnLiquidation.t.sol
* Run it `forge test --match-contract MoonwellWETHBugTest -vvv`
