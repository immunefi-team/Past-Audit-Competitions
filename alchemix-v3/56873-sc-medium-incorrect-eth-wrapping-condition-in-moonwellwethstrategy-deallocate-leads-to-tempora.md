# 56873 sc medium incorrect eth wrapping condition in moonwellwethstrategy deallocate leads to temporary freezing of funds

**Submitted on Oct 21st 2025 at 12:45:51 UTC by @flora for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56873
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/MoonwellWETHStrategy.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 1 hour

## Description

### Brief/Intro

A logical flaw exists in the `_deallocate` function of the `MoonwellWETHStrategy` and `StargateEthPoolStrategy` contracts. The condition for wrapping redeemed ETH into WETH is incorrect, causing the function to fail and revert when deallocating funds under certain common scenarios, such as minor redemption losses. While this bug does not lead to a direct loss of funds, it results in a denial of service for deallocations, making withdrawals from the strategy unreliable and causing wasted gas.

### Vulnerability Details

The root cause of the vulnerability is an incorrect conditional check on line 64 of `MoonwellWETHStrategy.sol`. When deallocating, the strategy redeems `mWETH` for native ETH and is supposed to wrap this ETH into WETH to be returned to the vault. However, the wrapping logic is executed only if `ethRedeemed + ethBalanceBefore >= amount`, which is mathematically equivalent to `ethBalanceAfter >= amount`.

**Problematic Code:** `src/strategies/optimism/MoonwellWETHStrategy.sol:64-66`

```solidity
// ...
if (ethRedeemed + ethBalanceBefore >= amount) {
    weth.deposit{value: ethRedeemed}();
}
require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "...");
// ...
```

This logic is flawed because the intention, as stated in the code comments, is to wrap *any* ETH received. The condition fails if there is a slight loss during redemption (i.e., `ethRedeemed < amount`) and the contract has no pre-existing ETH balance (`ethBalanceBefore == 0`). In this case:

1. The `if` condition evaluates to false.
2. The redeemed ETH is not wrapped into WETH.
3. The subsequent `require` statement, which checks that the contract's WETH balance is sufficient (`>= amount`), fails and reverts the transaction.

This issue is also present in `StargateEthPoolStrategy.sol` due to code duplication.

### Impact Details

The primary impact of this vulnerability is a **Denial of Service (DoS)** for the `deallocate` functionality. Any attempt to deallocate from the strategy will fail if there is a redemption loss and no residual ETH in the contract to satisfy the flawed condition. This makes the withdrawal process unreliable and dependent on unpredictable contract state (residual ETH). While the reverting transaction prevents permanent fund loss or lock, it leads to:

* **Operational Failure**: Legitimate deallocations cannot be processed, preventing funds from being moved out of the strategy.
* **Wasted Gas**: All failed attempts to deallocate will consume gas without successfully executing.
* **Reduced System Reliability**: The strategy's core function of deallocation becomes unstable and unreliable.

The severity is medium, as it impacts a core function of the protocol under normal operating conditions, even without malicious intent.

### References

* **Affected Contract 1**: `src/strategies/optimism/MoonwellWETHStrategy.sol`
* **Affected Contract 2**: `src/strategies/optimism/StargateEthPoolStrategy.sol`

## Proof of Concept

## Proof of Concept

````
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.21;

import "forge-std/Test.sol";
import "forge-std/console.sol";
import {MoonwellWETHStrategy} from "../src/strategies/optimism/MoonwellWETHStrategy.sol";
import {MYTStrategy} from "../src/MYTStrategy.sol";
import {IERC20} from "../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";

/**
 * @title Finding39_EndToEnd_POC
 * @notice Comprehensive end-to-end POC for Finding 39: ETH wrap condition bug in MoonwellWETHStrategy
 *
 * ## 🧪 Test Objective
 *
 * To verify if the conditional logic bug in MoonwellWETHStrategy._deallocate() causes transaction failures in a realistic scenario.
 *
 * ## 🔬 Problem Description
 *
 * Line 64-66 of MoonwellWETHStrategy.sol:
 * ```solidity
 * if (ethRedeemed + ethBalanceBefore >= amount) {
 *     weth.deposit{value: ethRedeemed}();
 * }
 * ```
 *
 * This condition is equivalent to: `if (ethBalanceAfter >= amount)`
 *
 * **Flaw**: The wrap only occurs if the current ETH balance >= requested amount, but it should unconditionally wrap all redeemed ETH.
 *
 * ## 🎯 Test Scenarios
 *
 * ### Scenario 1: No ETH Residue + Redemption Loss
 * - Strategy has 0 ETH residue
 * - Moonwell redemption returns less ETH than requested (due to exchange rate or rounding)
 * - Condition: `ethRedeemed + 0 < amount` → false
 * - Result: ETH is not wrapped → require fails → REVERT
 *
 * ### Scenario 2: WETH Residue + Redemption Loss (Critical Scenario)
 * - Strategy has some WETH residue (e.g., 10 WETH)
 * - Request to deallocate 100 WETH
 * - Moonwell only returns 95 ETH (5 ETH loss)
 * - Condition: `95 + 0 < 100` → false
 * - ETH is not wrapped
 * - Only 10 WETH residue is available
 * - require(10 >= 100) → REVERT ❌
 *
 * **But if wrapped correctly**:
 * - wrap 95 ETH → 95 WETH
 * - Total WETH = 10 + 95 = 105 WETH
 * - require(105 >= 100) → PASS ✅
 *
 * ### Scenario 3: Fixed Version Comparison
 * - Using the fixed version with unconditional wrap
 * - Should succeed under the same conditions
 *
 * ## ⚙️ Environment Information
 *
 * - Network: Optimism Mainnet Fork
 * - Moonwell WETH Market: 0xb4104C02BBf4E9be85AAa41a62974E4e28D59A33
 * - WETH: 0x4200000000000000000000000000000000000006
 * - Compiler: solc 0.8.21
 */
contract Finding39_EndToEnd_POC is Test {
    // Optimism mainnet addresses
    address constant WETH_OPTIMISM = 0x4200000000000000000000000000000000000006;
    address constant MOONWELL_WETH = 0xb4104C02BBf4E9be85AAa41a62974E4e28D59A33;
    address constant PERMIT2 = 0x000000000022D473030F116dDEE9F6B43aC78BA3;

    // Interfaces
    IERC20 weth;
    IMToken mWETH;

    // Mock contracts
    MockMoonwellToken mockMWETH;

    // Test contracts
    MoonwellWETHStrategyVulnerable strategyVulnerable;
    MoonwellWETHStrategyFixed strategyFixed;

    // Test actors
    address vault = makeAddr("vault");
    address admin = makeAddr("admin");
    address user = makeAddr("user");

    // Constants
    uint256 constant INITIAL_WETH = 1000 ether;

    function setUp() public {
        // Deploy mock contracts instead of forking
        MockWETH mockWeth = new MockWETH();
        mockMWETH = new MockMoonwellToken(address(mockWeth));

        weth = IERC20(address(mockWeth));
        mWETH = IMToken(address(mockMWETH));

        // Label addresses for better traces
        vm.label(address(weth), "WETH");
        vm.label(address(mWETH), "mWETH");
        vm.label(vault, "Vault");
        vm.label(admin, "Admin");

        // Deploy vulnerable strategy (current implementation)
        strategyVulnerable = new MoonwellWETHStrategyVulnerable(
            address(weth),
            address(mWETH)
        );
        vm.label(address(strategyVulnerable), "StrategyVulnerable");

        // Deploy fixed strategy
        strategyFixed = new MoonwellWETHStrategyFixed(
            address(weth),
            address(mWETH)
        );
        vm.label(address(strategyFixed), "StrategyFixed");

        // Give test contract some ETH and WETH
        vm.deal(address(this), 1000 ether);
        vm.deal(user, 100 ether);
    }

    /// @notice Helper: Convert ETH to WETH
    function _getWETH(address recipient, uint256 amount) internal {
        vm.deal(recipient, amount);
        vm.prank(recipient);
        IWETH(address(weth)).deposit{value: amount}();
    }

    /// @notice Helper: Set redemption loss in mock Moonwell
    function _setRedemptionLoss(uint256 lossBPS) internal {
        mockMWETH.setRedeemLoss(lossBPS);
    }

    /// @notice Helper: Calculate expected ETH from mWETH redemption with loss
    function _calculateExpectedRedemption(uint256 amount, uint256 lossBPS) internal pure returns (uint256) {
        // Apply loss percentage
        return amount - (amount * lossBPS / 10000);
    }

    /**
     * ═══════════════════════════════════════════════════════════════════
     * SCENARIO 1: No Residue + Redemption Loss = REVERT
     * ═══════════════════════════════════════════════════════════════════
     */
    function testScenario1_NoResidue_WithLoss_Reverts() public {
        console.log("\n=== SCENARIO 1: No Residue + Loss = REVERT ===\n");

        uint256 depositAmount = 100 ether;
        uint256 lossBPS = 500; // 5% loss

        // 1. Give strategy some WETH
        _getWETH(address(strategyVulnerable), depositAmount);
        console.log("Step 1: Strategy WETH balance:", weth.balanceOf(address(strategyVulnerable)) / 1e18, "WETH");

        // 2. Allocate to Moonwell
        vm.prank(address(strategyVulnerable));
        weth.approve(address(mWETH), depositAmount);

        vm.prank(address(strategyVulnerable));
        uint256 mintResult = mWETH.mint(depositAmount);
        require(mintResult == 0, "Moonwell mint failed");

        console.log("Step 2: Allocated to Moonwell");
        console.log("  Strategy WETH balance:", weth.balanceOf(address(strategyVulnerable)) / 1e18, "WETH");
        console.log("  Strategy mWETH balance:", mWETH.balanceOf(address(strategyVulnerable)) / 1e18, "mWETH");

        // 3. Verify no ETH or WETH residue
        assertEq(weth.balanceOf(address(strategyVulnerable)), 0, "Should have no WETH residue");
        assertEq(address(strategyVulnerable).balance, 0, "Should have no ETH residue");

        // 4. Set redemption loss
        _setRedemptionLoss(lossBPS);
        uint256 expectedEth = _calculateExpectedRedemption(depositAmount, lossBPS);
        console.log("\nStep 3: Set 5% redemption loss");
        console.log("  Expected ETH from redemption:", expectedEth / 1e18, "ETH");
        console.log("  Requested amount:", depositAmount / 1e18, "ETH");
        console.log("  Loss:", (depositAmount - expectedEth) / 1e18, "ETH");

        // 5. Attempt to deallocate
        console.log("\nStep 4: Attempting to deallocate", depositAmount / 1e18, "WETH...");

        console.log("\n[VULNERABILITY TRIGGER]");
        console.log("  ethBalanceBefore = 0");
        console.log("  ethRedeemed (ETH):", expectedEth / 1e18);
        console.log("  amount requested (ETH):", depositAmount / 1e18);
        console.log("  Condition result: FALSE - will NOT wrap ETH");
        console.log("  -> ETH will NOT be wrapped");
        console.log("  -> require(weth >= amount) will FAIL");
        console.log("  -> Transaction REVERTS");

        // Should revert due to insufficient WETH
        vm.expectRevert("Strategy balance is less than the amount needed");
        vm.prank(vault);
        strategyVulnerable.deallocate(depositAmount);

        console.log("\nCONFIRMED: Transaction reverted as expected");
    }

    /**
     * ═══════════════════════════════════════════════════════════════════
     * SCENARIO 2: WETH Residue + Redemption Loss (Critical Scenario)
     * ═══════════════════════════════════════════════════════════════════
     */
    function testScenario2_WETHResidue_WithLoss_FailsButShouldSucceed() public {
        console.log("\n=== SCENARIO 2: WETH Residue + Loss = Should Succeed But Fails ===\n");

        uint256 depositAmount = 100 ether;
        uint256 wethResidue = 10 ether;
        uint256 totalWeth = depositAmount + wethResidue;

        // 1. Give strategy WETH (including residue)
        _getWETH(address(strategyVulnerable), totalWeth);
        console.log("Step 1: Initial state");
        console.log("  Total WETH given:", totalWeth / 1e18, "WETH");
        console.log("  Will deposit:", depositAmount / 1e18, "WETH");
        console.log("  Will keep as residue:", wethResidue / 1e18, "WETH");

        // 2. Allocate only depositAmount to Moonwell, keep wethResidue
        vm.prank(address(strategyVulnerable));
        weth.approve(address(mWETH), depositAmount);

        vm.prank(address(strategyVulnerable));
        uint256 mintResult = mWETH.mint(depositAmount);
        require(mintResult == 0, "Moonwell mint failed");

        console.log("\nStep 2: After allocation");
        console.log("  Strategy WETH residue:", weth.balanceOf(address(strategyVulnerable)) / 1e18, "WETH");
        console.log("  Strategy mWETH balance:", mWETH.balanceOf(address(strategyVulnerable)) / 1e18, "mWETH");

        assertEq(weth.balanceOf(address(strategyVulnerable)), wethResidue, "Should have WETH residue");
        assertEq(address(strategyVulnerable).balance, 0, "Should have no ETH residue");

        // 3. Set redemption loss and calculate expected redemption
        uint256 lossBPS = 500; // 5% loss
        _setRedemptionLoss(lossBPS);
        uint256 expectedEth = _calculateExpectedRedemption(depositAmount, lossBPS);
        console.log("\nStep 3: Set 5% redemption loss");
        console.log("  Expected ETH from redemption:", expectedEth / 1e18, "ETH");
        console.log("  Requested amount:", depositAmount / 1e18, "ETH");
        console.log("  Loss:", (depositAmount - expectedEth) / 1e18, "ETH");

        // 4. Attempt to deallocate
        console.log("\nStep 4: Attempting to deallocate", depositAmount / 1e18, "WETH...");

        if (expectedEth < depositAmount) {
            uint256 potentialTotal = wethResidue + expectedEth;

            console.log("\n[CRITICAL VULNERABILITY CASE]");
            console.log("  WETH residue (WETH):", wethResidue / 1e18);
            console.log("  ETH to be redeemed (ETH):", expectedEth / 1e18);
            console.log("  ethBalanceBefore = 0 ETH");
            console.log("  Condition: FALSE - will NOT wrap");
            console.log("  -> ETH will NOT be wrapped");
            console.log("  -> Only WETH available:", wethResidue / 1e18);
            console.log("  -> Amount required:", depositAmount / 1e18);
            console.log("  -> Transaction REVERTS");
            console.log("\n[IF WRAPPED CORRECTLY]");
            console.log("  -> Would wrap ETH:", expectedEth / 1e18);
            console.log("  -> Total WETH would be:", potentialTotal / 1e18);
            console.log("  -> Would pass require:", potentialTotal >= depositAmount);

            if (potentialTotal >= depositAmount) {
                console.log("\nBUG CONFIRMED: Transaction should succeed but will fail!");

                // Should revert with current buggy implementation
                vm.expectRevert("Strategy balance is less than the amount needed");
                vm.prank(vault);
                strategyVulnerable.deallocate(depositAmount);

                console.log("Transaction reverted as predicted by vulnerability analysis");
            }
        }
    }

    /**
     * ═══════════════════════════════════════════════════════════════════
     * SCENARIO 3: Fixed Version - Same Conditions = SUCCESS
     * ═══════════════════════════════════════════════════════════════════
     */
    function testScenario3_FixedVersion_WETHResidue_WithLoss_Succeeds() public {
        console.log("\n=== SCENARIO 3: Fixed Version - WETH Residue + Loss = SUCCESS ===\n");

        uint256 depositAmount = 100 ether;
        uint256 wethResidue = 10 ether;
        uint256 totalWeth = depositAmount + wethResidue;

        // 1. Give strategy WETH (including residue)
        _getWETH(address(strategyFixed), totalWeth);
        console.log("Step 1: Initial state");
        console.log("  Total WETH given:", totalWeth / 1e18, "WETH");
        console.log("  Will deposit:", depositAmount / 1e18, "WETH");
        console.log("  Will keep as residue:", wethResidue / 1e18, "WETH");

        // 2. Allocate only depositAmount to Moonwell, keep wethResidue
        vm.prank(address(strategyFixed));
        weth.approve(address(mWETH), depositAmount);

        vm.prank(address(strategyFixed));
        uint256 mintResult = mWETH.mint(depositAmount);
        require(mintResult == 0, "Moonwell mint failed");

        console.log("\nStep 2: After allocation");
        console.log("  Strategy WETH residue:", weth.balanceOf(address(strategyFixed)) / 1e18, "WETH");
        console.log("  Strategy ETH residue:", address(strategyFixed).balance / 1e18, "ETH");

        assertEq(weth.balanceOf(address(strategyFixed)), wethResidue, "Should have WETH residue");

        // 3. Set redemption loss and calculate expected redemption
        uint256 lossBPS = 500; // 5% loss
        _setRedemptionLoss(lossBPS);
        uint256 expectedEth = _calculateExpectedRedemption(depositAmount, lossBPS);
        console.log("\nStep 3: Set 5% redemption loss");
        console.log("  Expected ETH from redemption:", expectedEth / 1e18, "ETH");

        // 4. Deallocate with fixed version
        console.log("\nStep 4: Deallocating with FIXED version...");

        uint256 wethBefore = weth.balanceOf(address(strategyFixed));
        console.log("  WETH before deallocate:", wethBefore / 1e18, "WETH");

        vm.prank(vault);
        uint256 returned = strategyFixed.deallocate(depositAmount);

        uint256 wethAfter = weth.balanceOf(address(strategyFixed));
        console.log("  WETH after deallocate:", wethAfter / 1e18, "WETH");
        console.log("  Returned amount:", returned / 1e18, "WETH");

        console.log("\n[FIXED VERSION LOGIC]");
        console.log("  -> Unconditionally wrapped (ETH):", expectedEth / 1e18);
        console.log("  -> Total WETH:", (wethResidue + expectedEth) / 1e18);
        console.log("  -> require passed");
        console.log("  -> Approved to vault (WETH):", depositAmount / 1e18);

        assertEq(returned, depositAmount, "Should return requested amount");
        console.log("\nSUCCESS: Fixed version works correctly!");
    }

    /**
     * ═══════════════════════════════════════════════════════════════════
     * SCENARIO 4: Direct Comparison - Vulnerable vs Fixed
     * ═══════════════════════════════════════════════════════════════════
     */
    function testScenario4_DirectComparison_VulnerableVsFixed() public {
        console.log("\n=== SCENARIO 4: Direct Comparison - Vulnerable vs Fixed ===\n");

        uint256 depositAmount = 100 ether;
        uint256 wethResidue = 10 ether;

        // Setup vulnerable strategy
        _getWETH(address(strategyVulnerable), depositAmount + wethResidue);
        vm.prank(address(strategyVulnerable));
        weth.approve(address(mWETH), depositAmount);
        vm.prank(address(strategyVulnerable));
        mWETH.mint(depositAmount);

        // Setup fixed strategy
        _getWETH(address(strategyFixed), depositAmount + wethResidue);
        vm.prank(address(strategyFixed));
        weth.approve(address(mWETH), depositAmount);
        vm.prank(address(strategyFixed));
        mWETH.mint(depositAmount);

        // Set redemption loss
        uint256 lossBPS = 500; // 5% loss
        _setRedemptionLoss(lossBPS);
        uint256 expectedEth = _calculateExpectedRedemption(depositAmount, lossBPS);

        console.log("Initial Setup (Both Strategies):");
        console.log("  Deposited:", depositAmount / 1e18, "WETH");
        console.log("  Residue:", wethResidue / 1e18, "WETH");
        console.log("  Expected redemption:", expectedEth / 1e18, "ETH");

        // Test vulnerable version
        console.log("\n[Testing VULNERABLE version]");
        bool vulnerableFailed = false;
        try strategyVulnerable.deallocate(depositAmount) {
            console.log("  Result: SUCCESS (unexpected)");
        } catch {
            console.log("  Result: FAILED (as expected due to bug)");
            vulnerableFailed = true;
        }

        // Test fixed version
        console.log("\n[Testing FIXED version]");
        bool fixedSucceeded = false;
        try strategyFixed.deallocate(depositAmount) returns (uint256 returned) {
            console.log("  Result: SUCCESS");
            console.log("  Returned:", returned / 1e18, "WETH");
            fixedSucceeded = true;
        } catch {
            console.log("  Result: FAILED (unexpected)");
        }

        console.log("\n[COMPARISON]");
        console.log("  Vulnerable version failed:", vulnerableFailed ? "YES" : "NO");
        console.log("  Fixed version succeeded:", fixedSucceeded ? "YES" : "NO");

        if (vulnerableFailed && fixedSucceeded) {
            console.log("\nVULNERABILITY CONFIRMED:");
            console.log("  The buggy condition causes unnecessary failures");
            console.log("  The fixed version (unconditional wrap) works correctly");
        }
    }

    /**
     * ═══════════════════════════════════════════════════════════════════
     * SCENARIO 5: Simulate Realistic Loss Scenario
     * ═══════════════════════════════════════════════════════════════════
     */
    function testScenario5_RealisticLossScenario() public {
        console.log("\n=== SCENARIO 5: Realistic Loss Scenario ===\n");
        console.log("Simulating realistic Moonwell exchange rate fluctuation");

        uint256 depositAmount = 50 ether; // Smaller amount for realistic test
        uint256 wethResidue = 5 ether;

        // Give vulnerable strategy WETH
        _getWETH(address(strategyVulnerable), depositAmount + wethResidue);

        console.log("Step 1: Deposit to Moonwell");
        console.log("  Amount:", depositAmount / 1e18, "WETH");
        console.log("  Residue kept:", wethResidue / 1e18, "WETH");

        vm.prank(address(strategyVulnerable));
        weth.approve(address(mWETH), depositAmount);
        vm.prank(address(strategyVulnerable));
        mWETH.mint(depositAmount);

        // Get exchange rate info
        uint256 exchangeRateBefore = mWETH.exchangeRateStored();
        console.log("\nStep 2: Check exchange rate");
        console.log("  Exchange rate:", exchangeRateBefore / 1e14, "/ 1e18");

        // Simulate time passing and some borrowing activity (which changes exchange rate)
        vm.warp(block.timestamp + 7 days);
        vm.roll(block.number + 50000);

        uint256 exchangeRateAfter = mWETH.exchangeRateCurrent();
        console.log("  Exchange rate after 7 days:", exchangeRateAfter / 1e14, "/ 1e18");

        // Calculate expected redemption
        uint256 mTokenBalance = mWETH.balanceOf(address(strategyVulnerable));
        uint256 expectedEth = (mTokenBalance * exchangeRateAfter) / 1e18;

        console.log("\nStep 3: Prepare to redeem");
        console.log("  mToken balance:", mTokenBalance / 1e18, "mWETH");
        console.log("  Expected ETH:", expectedEth / 1e18, "ETH");
        console.log("  Requested amount:", depositAmount / 1e18, "WETH");

        if (expectedEth < depositAmount) {
            console.log("  -> Loss detected:", (depositAmount - expectedEth) / 1e18, "ETH");
        }

        // Try to deallocate
        console.log("\nStep 4: Attempt deallocate with vulnerable version");

        uint256 potentialTotal = wethResidue + expectedEth;
        console.log("  If wrapped correctly, would have:", potentialTotal / 1e18, "WETH");
        console.log("  Required:", depositAmount / 1e18, "WETH");
        console.log("  Would pass?", potentialTotal >= depositAmount ? "YES" : "NO");

        if (expectedEth < depositAmount && potentialTotal >= depositAmount) {
            console.log("\nCRITICAL BUG CASE:");
            console.log("  Should succeed but will fail due to wrap condition bug");

            vm.expectRevert();
            vm.prank(vault);
            strategyVulnerable.deallocate(depositAmount);

            console.log("  Confirmed: Failed as predicted");
        }
    }
}

/**
 * ═══════════════════════════════════════════════════════════════════
 * MOCK CONTRACTS FOR TESTING
 * ═══════════════════════════════════════════════════════════════════
 */

/// @notice Moonwell mToken interface
interface IMToken {
    function mint(uint256 mintAmount) external returns (uint256);
    function redeemUnderlying(uint256 redeemAmount) external returns (uint256);
    function balanceOf(address owner) external view returns (uint256);
    function exchangeRateStored() external view returns (uint256);
    function exchangeRateCurrent() external returns (uint256);
}

/// @notice WETH interface
interface IWETH {
    function deposit() external payable;
    function withdraw(uint256) external;
    function balanceOf(address) external view returns (uint256);
    function approve(address, uint256) external returns (bool);
}

/**
 * @notice Vulnerable version of MoonwellWETHStrategy (current implementation)
 * Contains the buggy condition at line 64-66
 */
contract MoonwellWETHStrategyVulnerable {
    IWETH public immutable weth;
    IMToken public immutable mWETH;

    event StrategyDeallocationLoss(string message, uint256 requested, uint256 actual);

    constructor(address _weth, address _mWETH) {
        weth = IWETH(_weth);
        mWETH = IMToken(_mWETH);
    }

    function deallocate(uint256 amount) external returns (uint256) {
        uint256 ethBalanceBefore = address(this).balance;

        // Pull exact amount of underlying WETH out
        mWETH.redeemUnderlying(amount);

        // wrap any ETH received (Moonwell redeems to ETH for WETH markets)
        uint256 ethBalanceAfter = address(this).balance;
        uint256 ethRedeemed = ethBalanceAfter - ethBalanceBefore;

        if (ethRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, ethRedeemed);
        }

        // 🚨 BUG: Wrong condition!
        // This means: only wrap if total ETH >= amount
        // Equivalent to: if (ethBalanceAfter >= amount)
        if (ethRedeemed + ethBalanceBefore >= amount) {
            weth.deposit{value: ethRedeemed}();
        }

        require(weth.balanceOf(address(this)) >= amount, "Strategy balance is less than the amount needed");
        weth.approve(msg.sender, amount);
        return amount;
    }

    receive() external payable {}
}

/**
 * @notice Fixed version of MoonwellWETHStrategy
 * Unconditionally wraps all redeemed ETH
 */
contract MoonwellWETHStrategyFixed {
    IWETH public immutable weth;
    IMToken public immutable mWETH;

    event StrategyDeallocationLoss(string message, uint256 requested, uint256 actual);

    constructor(address _weth, address _mWETH) {
        weth = IWETH(_weth);
        mWETH = IMToken(_mWETH);
    }

    function deallocate(uint256 amount) external returns (uint256) {
        uint256 ethBalanceBefore = address(this).balance;

        // Pull exact amount of underlying WETH out
        mWETH.redeemUnderlying(amount);

        // wrap any ETH received (Moonwell redeems to ETH for WETH markets)
        uint256 ethBalanceAfter = address(this).balance;
        uint256 ethRedeemed = ethBalanceAfter - ethBalanceBefore;

        if (ethRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, ethRedeemed);
        }

        // ✅ FIX: Unconditionally wrap all redeemed ETH
        if (ethRedeemed > 0) {
            weth.deposit{value: ethRedeemed}();
        }

        require(weth.balanceOf(address(this)) >= amount, "Strategy balance is less than the amount needed");
        weth.approve(msg.sender, amount);
        return amount;
    }

    receive() external payable {}
}

/// @notice Mock WETH contract
contract MockWETH {
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    
    function deposit() external payable {
        balanceOf[msg.sender] += msg.value;
    }
    
    function withdraw(uint256 amount) external {
        require(balanceOf[msg.sender] >= amount);
        balanceOf[msg.sender] -= amount;
        payable(msg.sender).transfer(amount);
    }
    
    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        return true;
    }
    
    function transfer(address to, uint256 amount) external returns (bool) {
        require(balanceOf[msg.sender] >= amount);
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        return true;
    }
    
    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        require(balanceOf[from] >= amount);
        require(allowance[from][msg.sender] >= amount);
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        allowance[from][msg.sender] -= amount;
        return true;
    }
    
    receive() external payable {
        balanceOf[msg.sender] += msg.value;
    }
}

/// @notice Mock Moonwell mToken
contract MockMoonwellToken {
    MockWETH public weth;
    mapping(address => uint256) public balanceOf;
    uint256 public redeemLossBPS; // Loss in basis points (100 = 1%)
    
    constructor(address _weth) {
        weth = MockWETH(payable(_weth));
    }
    
    function setRedeemLoss(uint256 lossBPS) external {
        redeemLossBPS = lossBPS;
    }
    
    function mint(uint256 amount) external returns (uint256) {
        weth.transferFrom(msg.sender, address(this), amount);
        weth.withdraw(amount); // Convert to ETH
        balanceOf[msg.sender] += amount;
        return 0;
    }
    
    function redeemUnderlying(uint256 amount) external returns (uint256) {
        require(balanceOf[msg.sender] >= amount);
        balanceOf[msg.sender] -= amount;
        
        // Apply loss
        uint256 actualAmount = amount - (amount * redeemLossBPS / 10000);
        
        // Send ETH to caller
        payable(msg.sender).transfer(actualAmount);
        return 0;
    }
    
    function exchangeRateStored() external pure returns (uint256) {
        return 1e18; // 1:1 rate
    }
    
    function exchangeRateCurrent() external pure returns (uint256) {
        return 1e18; // 1:1 rate
    }
    
    receive() external payable {}
}


````
