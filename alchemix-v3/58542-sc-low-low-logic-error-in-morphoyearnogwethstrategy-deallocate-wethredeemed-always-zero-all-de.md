# 58542 sc low low logic error in morphoyearnogwethstrategy deallocate wethredeemed always zero all deallocations emit strategydeallocationloss&#x20;

## #58542 \[SC-Low] \[Low] Logic Error in MorphoYearnOGWETHStrategy.\_deallocate(): \`wethRedeemed\` Always Zero → All Deallocations Emit \`StrategyDeallocationLoss\`

**Submitted on Nov 3rd 2025 at 06:16:49 UTC by @chief\_hunter888 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58542
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Broken accounting / misreported events

### Description

## \[Low] Logic Errors in MorphoYearnOGWETHStrategy.\_deallocate(): `wethRedeemed` Always Zero → All Deallocations Emit `StrategyDeallocationLoss`

### Summary of the bug

The `_deallocate()` implementation in `MorphoYearnOGWETHStrategy.sol` contains a logic error that causes the variable `wethRedeemed` to always record as zero, even when WETH is correctly redeemed and returned to the adapter contract.

This happens because the contract reads its WETH balance twice after the redemption using identical logic, but does not store the “before” value prior to calling the redemption. As a result, both readings occur after the redemption has completed — making their relative difference zero every time.

### \[Low] - Bug #1 Faulty Event Emission and Accounting Issues

This bug breaks both accounting and event reporting for every deallocate() execution:

1. Incorrect Event Emission: The `StrategyDeallocationLos`s\` event is always emitted, falsely signaling a strategy loss, when there was none. The event becomes a meaningless waste of gas and does not alert about real strategy losses.
2. Incorrect Data Reporting: The `StrategyDeallocationLos` event’s `wethRedeemed` field is always emitted as 0, even when WETH was actually redeemed and transferred correctly. This breaks the accounting of the backend services, because before this field was used to infer how much WETH was redeemed if the amount was less than the targeted amount. Now there is no way of knowing this from the event.
3. False Loss Reporting: The strategy will always appear to operate at a loss, which may prevent it from being selected or used in yield allocation decisions.
4. While funds are not directly stolen or misallocated, this bug erodes the integrity of accounting and monitoring, making it impossible to trust strategy-level performance data.

### \[Insight] Finding - Bug #2 Costly and Unnecessarily Complex Require Statement

Additionally, the require on line 57 relies on the faulty wethRedeemed value:

```solidity
require(wethRedeemed + wethBalanceBefore >= amount, "Strategy balance is less than the amount needed");
```

The lefthandside of the require statement introduces additional gas cost because the wethRedeemed + wethBalanceBefore is the same as the `wethBalanceAfter`, a variable available and already cached a few lines above, which could be used here instead to improve code clarity and reduce gas costs by avoiding duplicate calculations.

### Context

The `MorphoYearnOGWETHStrategy.sol` is a strategy adapter in the Alchemix system that: Role:

* Acts as a bridge between Alchemix's MYT Vault and the Morpho Yearn vault
* Receives WETH from the MYT vault
* Deposits it into the Morpho Yearn OG WETH vault
* Gets vault shares in return
* Withdraws WETH when deallocating (this is where the bug is!)

The adapter connects to the Morpho Yearn vault which is a yield-bearing ERC4626 tokenized vault created by Yearn Finance that is expected to accrue value over time. Upon deposit Alchemix gets a share in the tokenized vault, which reflects the amoun of deposited WETH and is expected to accrue more WETH over time (ctoken style).

Now, the bug is that the adapter's faulty implementation of the \_deallocate() functionality incorrectly registers the redeemed WETH amount as zero, when it was actually redeemed back to the adapter contract.

// Allocation (✅ Works correctly) \_allocate(100 WETH) { 1. Strategy receives 100 WETH from MYT vault 2. Approves Morpho Yearn vault to spend WETH 3. Calls vault.deposit(100 WETH) 4. Receives \~98 vault shares (ERC4626 conversion) 5. Strategy now holds shares representing 100 WETH value }

// Deallocation (❌ BUG HERE!) \_deallocate(105 WETH) { 1. vault.withdraw(105 WETH) → Burns shares, returns \~104.x WETH 2. wethBalanceBefore = 104.x WETH ❌ (measured AFTER withdrawal!) 3. wethBalanceAfter = 104.x WETH 4. wethRedeemed = 0 ❌ (always zero!) 5. Require statement would fail because it 0 + 104.x WETH >= 105 WETH, which fails. ❌ }

### Vulnerability Details

#### Root Cause

In the `_deallocate` function of `MorphoYearnOGWETHStrategy` (lines 49-61), the WETH balance is measured twice after the withdrawal has already occurred:

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    vault.withdraw(amount, address(this), address(this));              // Line 50: WETH arrives
    uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));  // Line 51: Balance AFTER withdrawal
    uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));   // Line 52: Balance AFTER withdrawal (same!)
    uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;       // Line 53: ALWAYS ZERO!
    if (wethRedeemed < amount) {
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed); // Always emits 0
    }
    require(wethRedeemed + wethBalanceBefore >= amount, "Strategy balance is less than the amount needed"); // Line 57
    require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed"); // Line 58
    TokenUtils.safeApprove(address(weth), msg.sender, amount);          // Line 59: Often reverts with ERC20InsufficientBalance
    return amount;
}
```

### Severity Justification

#### Impact Medium/Low

1. Faulty Strategy Loss Alerting: Produces misleading “loss” events for every deallocation. Alerting feature is lost, there is no way of knowing how much WETH was redeemed, if it was less than the requested amount.
2. Incorrect Accounting Data & Data Loss: Corrupts both on-chain and off-chain accounting data via faulty accounting information emission. While it does not immediately steal funds, it undermines the integrity of the protocol’s core accounting. Once this bug is deployed, there is no way of retroactively fixing the past events.

Please refer to the POC tests that illustrate the incorrect event emission. This is definitely at least a low finding because the code does not provide the desired returns, it does not loose funds, but given the impact on the backend accounting services is huge, this might be ranked even higher. The backend is required to be up to date to input correct input data into the deallocate() function.

#### Likelihood: EXTREMELY HIGH

* Affects every single deallocation operation
* Bug is always active, not dependent on external conditions
* Will manifest as soon as any yield accrues

### Recommendations

#### Fix

Move the `wethBalanceBefore` measurement to **before** the withdrawal:

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this)); // The bugfix
    vault.withdraw(amount, address(this), address(this));
    uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
    uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
    
    if (wethRedeemed < amount) {
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
    }
    require(wethBalanceAfter >= amount, "Insufficient WETH withdrawn");
    TokenUtils.safeApprove(address(weth), msg.sender, wethRedeemed); // Approve actual amount received
    return amount;
}
```

### Proof of Concept

### Proof of Concept

Please add this test to `MorphoYearnOGWETHStrategy.t.sol`

```
    /**
     * @notice Test 2: Broken Accounting on Every Withdrawal
     * @dev Clean version matching documentation
     */
    function test_POC_emits_StrategyDeallocationLoss_event_with_zero_wethRedeemed_which_is_untrue() public {
        vm.startPrank(vault);
        
        // Multiple deposits totaling 150 WETH
        deal(WETH, strategy, 150 ether);
        bytes memory prevAlloc = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAlloc, 150 ether, "", address(vault));
        
        uint256 assetsInStrategy = IMYTStrategy(strategy).realAssets();
        uint256 withdrawAmount = assetsInStrategy / 2;
        assertEq(IMYTStrategy(strategy).realAssets(), assetsInStrategy);
        // Event shows wethRedeemed = 0 even though funds were withdrawn!
        prevAlloc = abi.encode(150 ether);
        vm.expectEmit(true, true, true, true, strategy);
        emit StrategyDeallocationLoss("Strategy deallocation loss.", withdrawAmount, 0);
        IMYTStrategy(strategy).deallocate(prevAlloc, withdrawAmount, "", address(vault));
        // Withdrawal succeeds, but accounting is completely broken!
        // Amount has been withdrawn, 1 wei less than expected due to rounding
        assertEq(IMYTStrategy(strategy).realAssets(), assetsInStrategy - withdrawAmount-1);
        
        vm.stopPrank();
    }
```

full test file

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import "../libraries/BaseStrategyTest.sol";
import {MorphoYearnOGWETHStrategy} from "../../strategies/mainnet/MorphoYearnOGWETH.sol";
import {IERC20} from "../../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {console} from "../../../lib/forge-std/src/console.sol";
import {IERC4626} from "../../../lib/openzeppelin-contracts/contracts/interfaces/IERC4626.sol";
import {stdError} from "../../../lib/forge-std/src/StdError.sol";

contract MockMorphoYearnOGWETHStrategy is MorphoYearnOGWETHStrategy {
    constructor(address _myt, StrategyParams memory _params, address _vault, address _weth, address _permit2Address)
        MorphoYearnOGWETHStrategy(_myt, _params, _vault, _weth, _permit2Address)
    {}
}

contract MorphoYearnOGWETHStrategyTest is BaseStrategyTest {
    address public constant MORPHO_YEARN_OG_VAULT = 0xE89371eAaAC6D46d4C3ED23453241987916224FC;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant MAINNET_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;

    // Event for testing
    event StrategyDeallocationLoss(string message, uint256 amountRequested, uint256 actualAmountSent);

    function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) {
        return IMYTStrategy.StrategyParams({
            owner: address(1),
            name: "MorphoYearnOGETH",
            protocol: "MorphoYearnOGETH",
            riskClass: IMYTStrategy.RiskClass.LOW,
            cap: 10_000e18,
            globalCap: 1e18,
            estimatedYield: 100e18,
            additionalIncentives: false,
            slippageBPS: 1
        });
    }

    function getTestConfig() internal pure override returns (TestConfig memory) {
        return TestConfig({vaultAsset: WETH, vaultInitialDeposit: 1000e18, absoluteCap: 10_000e18, relativeCap: 1e18, decimals: 18});
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 23_298_447;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("MAINNET_RPC_URL");
    }

    function createStrategy(address vault, IMYTStrategy.StrategyParams memory params) internal override returns (address) {
        return address(new MockMorphoYearnOGWETHStrategy(vault, params, MORPHO_YEARN_OG_VAULT, WETH, MAINNET_PERMIT2));
    }

    function test_strategy_deallocate_reverts_due_to_slippage(uint256 amountToAllocate, uint256 amountToDeallocate) public {
        amountToAllocate = bound(amountToAllocate, 1e18, testConfig.vaultInitialDeposit);
        amountToDeallocate = amountToAllocate;
    }

    /**
     * @notice Helper function to simulate yield accrual in the Morpho Yearn vault
     * @dev For ERC4626 vaults, yield is simulated by increasing totalAssets without increasing totalSupply
     * @param yieldPercentBps Yield to add as basis points of current total assets (e.g. 500 = 5%)
     */
    function _simulateYieldPercent(uint256 yieldPercentBps) internal {
        // Calculate yield based on total vault assets, not just our deposit
        // This simulates realistic yield accrual across all vault participants
        uint256 vaultTotalAssets = IERC4626(MORPHO_YEARN_OG_VAULT).convertToAssets(IERC4626(MORPHO_YEARN_OG_VAULT).totalSupply());
        uint256 yieldAmount = (vaultTotalAssets * yieldPercentBps) / 10000;
        
        // Deal WETH directly to the Morpho Yearn vault to increase its totalAssets
        // This increases the share price (assets per share), simulating yield accrual
        deal(WETH, MORPHO_YEARN_OG_VAULT, IERC20(WETH).balanceOf(MORPHO_YEARN_OG_VAULT) + yieldAmount);
    }

    /**
     * @notice Test 2: Broken Accounting on Every Withdrawal
     * @dev Clean version matching documentation
     */
    function test_POC_emits_StrategyDeallocationLoss_event_with_zero_wethRedeemed_which_is_untrue() public {
        vm.startPrank(vault);
        
        // Multiple deposits totaling 150 WETH
        deal(WETH, strategy, 150 ether);
        bytes memory prevAlloc = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAlloc, 150 ether, "", address(vault));
        
        uint256 assetsInStrategy = IMYTStrategy(strategy).realAssets();
        uint256 withdrawAmount = assetsInStrategy / 2;
        assertEq(IMYTStrategy(strategy).realAssets(), assetsInStrategy);
        // Event shows wethRedeemed = 0 even though funds were withdrawn!
        prevAlloc = abi.encode(150 ether);
        vm.expectEmit(true, true, true, true, strategy);
        emit StrategyDeallocationLoss("Strategy deallocation loss.", withdrawAmount, 0);
        IMYTStrategy(strategy).deallocate(prevAlloc, withdrawAmount, "", address(vault));
        // Withdrawal succeeds, but accounting is completely broken!
        // Amount has been withdrawn, but 1 wei less than expected due to rounding
        assertEq(IMYTStrategy(strategy).realAssets(), assetsInStrategy - withdrawAmount-1);
        
        vm.stopPrank();
    }

    /**
     * @notice Test demonstrating the wethRedeemed always zero bug in _deallocate
     * @dev This test shows the exact scenario from high2-1.md:
     *      - vault.withdraw() happens first (line 50)
     *      - wethBalanceBefore is measured AFTER withdrawal (line 51) (WRONG!)
     *      - wethBalanceAfter is also measured AFTER withdrawal (line 52)
     *      - wethRedeemed = 0 always (WRONG!), even when WETH was actually redeemed
     *      - StrategyDeallocationLoss event always emits with wethRedeemed = 0
     *      - Require statement on line 57 can fail in loss scenarios
     */
    function test_MorphoYearnOGWETH_wethRedeemed_always_zero_bug() public {
        vm.startPrank(vault);
        
        console.log("\n=== TEST: wethRedeemed Always Zero Bug ===\n");
        
        // Step 1: Allocate 100 WETH to strategy
        uint256 allocAmount = 100 ether;
        deal(WETH, strategy, allocAmount);
        bytes memory prevAlloc = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAlloc, allocAmount, "", address(vault));
        
        uint256 wethBalanceBeforeAllocation = IERC20(WETH).balanceOf(strategy);
        console.log("Step 1: Allocated 100 WETH");
        console.log("WETH balance in strategy before deallocation:", wethBalanceBeforeAllocation / 1e18, "WETH");
        
        // Get strategy's vault shares
        uint256 strategyShares = IERC4626(MORPHO_YEARN_OG_VAULT).balanceOf(strategy);
        uint256 realAssets = IMYTStrategy(strategy).realAssets();
        console.log("Strategy holds vault shares worth:", realAssets / 1e18, "WETH");
        
        // Step 2: Measure WETH balance BEFORE calling _deallocate
        // This is what SHOULD be measured, but currently isn't
        uint256 wethBalanceBeforeDealloc = IERC20(WETH).balanceOf(strategy);
        console.log("\nStep 2: WETH balance BEFORE _deallocate():", wethBalanceBeforeDealloc / 1e18, "WETH");
        assertEq(wethBalanceBeforeDealloc, 0, "Strategy should have 0 WETH before deallocation (it's all in vault)");
        
        // Step 3: Attempt to deallocate (this will trigger the bug)
        uint256 amountToDeallocate = 50 ether; // Partial withdrawal
        prevAlloc = abi.encode(allocAmount);
        
        console.log("\nStep 3: Calling deallocate with 50 WETH");
        console.log("BUG: In _deallocate(), the flow is:");
        console.log("  1. vault.withdraw(50 WETH) executes - Returns ~49.x WETH");
        console.log("  2. wethBalanceBefore = measured AFTER withdrawal (WRONG!)");
        console.log("  3. wethBalanceAfter = measured AFTER withdrawal");
        console.log("  4. wethRedeemed = 0 (always!)");
        
        // Expect the StrategyDeallocationLoss event with wethRedeemed = 0
        // This happens because wethBalanceBefore and wethBalanceAfter are both measured AFTER withdrawal
        vm.expectEmit(true, true, true, true, strategy);
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amountToDeallocate, 0);
        
        IMYTStrategy(strategy).deallocate(prevAlloc, amountToDeallocate, "", address(vault));
        
        // Step 4: Verify WETH was actually received (even though event said 0)
        uint256 wethBalanceAfterDealloc = IERC20(WETH).balanceOf(strategy);
        console.log("\nStep 4: WETH balance AFTER deallocation:", wethBalanceAfterDealloc / 1e18, "WETH");
        
        // The strategy SHOULD have received WETH, but the event reported 0
        // Due to ERC4626 rounding, we might get slightly less than requested
        assertGt(wethBalanceAfterDealloc, 0, "Strategy should have received WETH");
        assertApproxEqAbs(wethBalanceAfterDealloc, amountToDeallocate, 1e17, "Should have received approximately 50 WETH");
        
        console.log("\n=== BUG CONFIRMED ===");
        console.log("WETH was actually redeemed:", wethBalanceAfterDealloc / 1e18, "WETH");
        console.log("But StrategyDeallocationLoss event reported: 0 WETH (WRONG!)");
        console.log("Accounting is completely broken!");
        
        vm.stopPrank();
    }
}
```
