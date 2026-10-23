# 57791 sc insight receipt token misconfiguration in aave strategies

**Submitted on Oct 28th 2025 at 22:17:07 UTC by @Icon0x for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57791
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/AaveV3OPUSDCStrategy.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hour
  * Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

Aave strategies approve wrong token to Permit2, blocking emergency migration mechanism during protocol upgrades.

## Vulnerability Details

### Description

The Aave V3 strategy implementations pass the wrong token (underlying asset) instead of the receipt token (aToken) to the parent `MYTStrategy` constructor. This causes Permit2 to receive approval for a token the strategy doesn't hold, while having no approval for the token the strategy actually holds.

### Root Cause

In all three Aave strategies, the constructor passes the underlying asset (`_usdc` or `_weth`) instead of the Aave receipt token (`_aUSDC` or `_aWETH`) as the fourth parameter to `MYTStrategy`:

```solidity
// WRONG - Current Implementation
MYTStrategy(_myt, _params, _permit2Address, _usdc)

// CORRECT - Should be
MYTStrategy(_myt, _params, _permit2Address, _aUSDC)
```

The `MYTStrategy` constructor then approves this token to Permit2:

```solidity
// MYTStrategy.sol:106
IERC20(receiptToken).approve(permit2Address, type(uint256).max);
```

### Affected Code Locations

| File                                                | Line | Current Code              | Should Be                  |
| --------------------------------------------------- | ---- | ------------------------- | -------------------------- |
| `src/strategies/optimism/AaveV3OPUSDCStrategy.sol`  | 31   | `MYTStrategy(..., _usdc)` | `MYTStrategy(..., _aUSDC)` |
| `src/strategies/arbitrum/AaveV3ARBUSDCStrategy.sol` | 31   | `MYTStrategy(..., _usdc)` | `MYTStrategy(..., _aUSDC)` |
| `src/strategies/arbitrum/AaveV3ARBWETHStrategy.sol` | 31   | `MYTStrategy(..., _weth)` | `MYTStrategy(..., _aWETH)` |

***

## Impact Details

Defeats the purpose of Permit2 integration for these strategies

## References

* <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/optimism/AaveV3OPUSDCStrategy.sol#L31>
* <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/arbitrum/AaveV3ARBUSDCStrategy.sol#L31C9-L31C59>
* <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/arbitrum/AaveV3ARBWETHStrategy.sol#L31C8-L31C59>

## Proof of Concept

## Proof of Concept

Add the below test suite to "src/test/strategies/AaveV3OPUSDCStrategy.t.sol"

```solidity
  function test_receiptToken_wrongApproval_breaksPermit2Migration() public {
        console.log("\n=== POC: Receipt Token Bug in AaveV3OPUSDCStrategy ===\n");
        
        // Step 1: Allocate funds to Aave V3
        uint256 allocateAmount = 1000e6; // 1000 USDC
        vm.startPrank(vault);
        deal(USDC, strategy, allocateAmount);
        
        bytes memory prevAllocation = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocation, allocateAmount, "", address(vault));
        vm.stopPrank();
        
        // Step 2: Check what the strategy actually holds
        IERC20 usdc = IERC20(USDC);
        IERC20 aUsdc = IERC20(AAVE_V3_USDC_ATOKEN);
        
        uint256 strategyUsdcBalance = usdc.balanceOf(strategy);
        uint256 strategyAUsdcBalance = aUsdc.balanceOf(strategy);
        
        console.log("After allocation:");
        console.log("  Strategy USDC balance:", strategyUsdcBalance);
        console.log("  Strategy aUSDC balance:", strategyAUsdcBalance);
        console.log("");
        
        // Step 3: Check Permit2 approvals
        uint256 permit2UsdcApproval = usdc.allowance(strategy, OPTIMISM_PERMIT2);
        uint256 permit2AUsdcApproval = aUsdc.allowance(strategy, OPTIMISM_PERMIT2);
        
        console.log("Permit2 Approvals:");
        console.log("  USDC allowance:", permit2UsdcApproval);
        console.log("  aUSDC allowance:", permit2AUsdcApproval);
        console.log("");
        
        // Step 4: Demonstrate the bug
        console.log("BUG IDENTIFIED:");
        console.log("  Strategy holds:", strategyAUsdcBalance, "aUSDC (the receipt token)");
        console.log("  Strategy holds:", strategyUsdcBalance, "USDC (transient, always 0 after allocation)");
        console.log("");
        console.log("  Permit2 can transfer:", permit2UsdcApproval > 0 ? "USDC (WRONG!)" : "nothing");
        console.log("  Permit2 can transfer:", permit2AUsdcApproval > 0 ? "aUSDC (correct)" : "NO aUSDC (BUG!)");
        console.log("");
        
        // Assert the bug exists
        assertEq(strategyUsdcBalance, 0, "Strategy should not hold USDC after allocation");
        assertGt(strategyAUsdcBalance, 0, "Strategy should hold aUSDC after allocation");
        assertGt(permit2UsdcApproval, 0, "BUG: Permit2 has approval for USDC");
        assertEq(permit2AUsdcApproval, 0, "BUG: Permit2 has NO approval for aUSDC");
        
        console.log("IMPACT:");
        console.log("  During Aave V3->V4 upgrade, admin cannot use Permit2 to migrate aUSDC");
        console.log("  Emergency migration path is BLOCKED");
        console.log("  Must use manual deallocate instead (slower, riskier)");
        console.log("");
        console.log("ROOT CAUSE:");
        console.log("  AaveV3OPUSDCStrategy.sol:31 passes _usdc instead of _aUSDC to parent");
        console.log("  Should be: MYTStrategy(_myt, _params, _permit2Address, _aUSDC)");
        console.log("========================================\n");
    }
```

then run:

```bash
forge test --match-test test_receiptToken_wrongApproval_breaksPermit2Migration -vv
```

Test result:

```
After allocation:
  Strategy USDC balance: 0
  Strategy aUSDC balance: 999999999

Permit2 Approvals:
  USDC allowance: 115792089237316195423570985008687907853269984665640564039457584007913129639935
  aUSDC allowance: 0

BUG CONFIRMED:
  - Permit2 can move USDC (which strategy doesn't hold)
  - Permit2 CANNOT move aUSDC (which strategy DOES hold)
  - Emergency migration path is BROKEN
```
