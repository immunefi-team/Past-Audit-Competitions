# 58435 sc high systemic accounting bug leads to protocol insolvency

**Submitted on Nov 2nd 2025 at 10:53:02 UTC by @teoslaf1 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58435
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Summary

All strategy contracts across all chains (Mainnet, Arbitrum, Optimism, Base) return incorrect deallocation amounts during normal vault operations, causing permanent accounting corruption that compounds with every operation and leads to protocol insolvency. The bug requires no attacker - it triggers automatically during legitimate vault rebalancing by admin/operators. This affects every strategy (Euler, Aave, Compound, Morpho, Tokemak, etc.), every asset (USDC, WETH), and every user, making it a systemic protocol-wide vulnerability with no recovery mechanism.

While the bug exists in the \_deallocate() implementation of all strategy contracts, I'm reporting src/MYTStrategy.sol as the primary affected file because it's the base contract that all strategies inherit from and contains the deallocate() function that propagates the incorrect return values to the vault's accounting system.

## Vulnerability Details

### Root Cause

Every strategy's `_deallocate()` function calculates the actual amount received from external protocols but returns the requested amount instead:

```solidity
// File: src/strategies/mainnet/EulerUSDCStrategy.sol (lines 34-47)
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 usdcBalanceBefore = TokenUtils.safeBalanceOf(address(usdc), address(this));

    // Withdraw from Euler vault
    vault.withdraw(amount, address(this), address(this));

    uint256 usdcBalanceAfter = TokenUtils.safeBalanceOf(address(usdc), address(this));
    uint256 usdcRedeemed = usdcBalanceAfter - usdcBalanceBefore;  // ← Calculates actual

    if (usdcRedeemed < amount) {
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, usdcRedeemed);
    }

    require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount,
            "Strategy balance is less than the amount needed");
    TokenUtils.safeApprove(address(usdc), msg.sender, amount);

    return amount;  // bug: Returns requested, not usdcRedeemed
}
```

The strategy:

1. Calculates `usdcRedeemed` (actual amount received)
2. Detects if `usdcRedeemed < amount` (loss occurred)
3. Emits a loss event
4. But returns `amount` instead of `usdcRedeemed`

### How Vault Tracking Breaks

The Morpho vault updates its internal tracking based on the returned value:

```solidity
// File: src/MYTStrategy.sol (lines 119-134)
function deallocate(bytes memory data, uint256 assets, bytes4 selector, address sender)
    external
    onlyVault
    returns (bytes32[] memory strategyIds, int256 change)
{
    uint256 oldAllocation = abi.decode(data, (uint256));
    uint256 amountDeallocated = _deallocate(assets);  // ← Gets wrong value!
    uint256 newAllocation = oldAllocation - amountDeallocated;
    return (ids(), int256(newAllocation) - int256(oldAllocation));
}

// Morpho vault then updates:
allocation[strategyId] -= amountReturned;  // ← Uses wrong value!
```

### Why Actual Amount Differs from Requested

External protocols return less than requested due to: ERC4626 rounding (Euler's `toSharesUp()` rounds against withdrawers), protocol fees (0.01-0.1%), slippage on LP unwinding, and market stress (1-5% during liquidations).

## Impact Details

This bug causes protocol insolvency because:

1. Vault's internal accounting becomes permanently incorrect
2. Tracking shows MORE assets than actually exist
3. Protocol cannot fulfill all withdrawal requests
4. No recovery mechanism exists
5. Compounds with every operation

### Recommended Fix

**Change all strategies to return actual amount received:**

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 balanceBefore = token.balanceOf(address(this));
    protocol.withdraw(amount, address(this), address(this));
    uint256 balanceAfter = token.balanceOf(address(this));
    uint256 actualReceived = balanceAfter - balanceBefore;

    if (actualReceived < amount) {
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, actualReceived);
    }

    TokenUtils.safeApprove(address(token), msg.sender, actualReceived);

    // fix: Return actual amount received
    return actualReceived;
}
```

## Proof of Concept

## Add this to /test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "forge-std/Test.sol";
import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {IERC4626} from "../../lib/openzeppelin-contracts/contracts/interfaces/IERC4626.sol";
import {EulerUSDCStrategy} from "../strategies/mainnet/EulerUSDCStrategy.sol";
import {IMYTStrategy} from "../interfaces/IMYTStrategy.sol";

/**
 * @title PoC: Euler Strategy Accounting Desync
 * @notice Demonstrates how the EulerUSDCStrategy returns wrong amounts on deallocation,
 *         causing internal tracking to desync from real balances
 */
contract PoC_EulerAccountingBug is Test {
    address constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
    address constant EULER_VAULT = 0xe0a80d35bB6618CBA260120b279d357978c42BCE;
    address constant PERMIT2 = 0x000000000022D473030F116dDEE9F6B43aC78BA3;
    
    IERC20 usdc = IERC20(USDC);
    IERC4626 eulerVault = IERC4626(EULER_VAULT);
    
    EulerUSDCStrategy strategy;
    address mockVault = address(this); // Test contract acts as vault
    
    function setUp() public {
        // Fork mainnet at a recent block
        vm.createSelectFork(vm.envString("MAINNET_RPC_URL"), 22_089_302);
        
        // Create strategy params
        IMYTStrategy.StrategyParams memory params = IMYTStrategy.StrategyParams({
            owner: address(this),
            name: "EulerUSDC",
            protocol: "EulerUSDC",
            riskClass: IMYTStrategy.RiskClass.LOW,
            cap: 100_000e6,
            globalCap: 1e18,
            estimatedYield: 100e6,
            additionalIncentives: false,
            slippageBPS: 100 // 1%
        });
        
        // Deploy actual EulerUSDCStrategy
        strategy = new EulerUSDCStrategy(
            mockVault,      // MYT vault (we act as vault)
            params,
            USDC,
            EULER_VAULT,
            PERMIT2
        );
        
        // Give strategy some USDC
        deal(USDC, address(strategy), 100_000e6);
    }
    
    function test_RoundingCausesDesync() public {
        // Step 1: Allocate 10,000 USDC
        strategy.allocate(abi.encode(0), 10_000e6, bytes4(0), address(this));
        uint256 vaultTracking = 10_000e6;
        
        // Step 2: Check shares before
        uint256 sharesBefore = eulerVault.balanceOf(address(strategy));
        uint256 assetsBefore = eulerVault.convertToAssets(sharesBefore);
        
        emit log_named_uint("Shares before", sharesBefore);
        emit log_named_uint("Assets before", assetsBefore);
        
        // Step 3: Deallocate 1,000 USDC
        (, int256 change) = strategy.deallocate(
            abi.encode(vaultTracking),
            1_000e6,
            bytes4(0),
            address(this)
        );
        
        uint256 reported = uint256(-change);
        emit log_named_uint("Strategy reported", reported);
        
        // Step 4: Check shares after
        uint256 sharesAfter = eulerVault.balanceOf(address(strategy));
        uint256 assetsAfter = eulerVault.convertToAssets(sharesAfter);
        
        emit log_named_uint("Shares after", sharesAfter);
        emit log_named_uint("Assets after", assetsAfter);
        
        // Step 5: Calculate what actually happened
        uint256 sharesBurned = sharesBefore - sharesAfter;
        uint256 valueOfSharesBurned = eulerVault.convertToAssets(sharesBurned);
        
        emit log_named_uint("Shares burned", sharesBurned);
        emit log_named_uint("Value of shares burned", valueOfSharesBurned);
        
        // Step 6: Update vault tracking
        vaultTracking -= reported;
        
        emit log_named_uint("Vault tracking", vaultTracking);
        emit log_named_uint("Real balance", assetsAfter);
        
        // Step 7: Calculate desync
        int256 desync = int256(vaultTracking) - int256(assetsAfter);
        emit log_named_int("Desync:", desync);
        
        // The bug: Strategy reports requested amount, not actual
        // This causes vault tracking to diverge from reality
        assertEq(reported, 1_000e6, "Strategy reports requested amount");
    }
}
```
