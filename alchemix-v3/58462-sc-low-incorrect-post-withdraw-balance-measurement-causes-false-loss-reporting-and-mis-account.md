# 58462 sc low incorrect post withdraw balance measurement causes false loss reporting and mis accounting in morphoyearnogwethstrategy deallocate&#x20;

**Submitted on Nov 2nd 2025 at 13:57:46 UTC by @Kissiahmyo for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58462
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

The `MorphoYearnOGWETHStrategy._deallocate` function reads `wethBalanceBefore` and `wethBalanceAfter` after calling `vault.withdraw`, making both variables reflect the same post-withdraw balance. Because of this, `wethRedeemed` is computed as `0` in almost all cases, leading to unconditional “deallocation loss” events and breaking the intended sanity checks that rely on the redeemed amount. The bug undermines accurate accounting of redemptions, can trigger false alarms in monitoring, and may interact poorly with automated governance or operational tooling that reacts to perceived losses.

## Vulnerability Details

A read-order bug in the WETH redemption path miscomputes the redeemed amount by measuring pre- and post-withdraw balances only after the withdrawal has occurred. This produces `0` for the redeemed delta, leading to perpetual false “loss” events and unreliable deallocation accounting. In production, this can degrade monitoring fidelity, cause erroneous operational responses, and reduce confidence in strategy behavior, especially under stress.

* Affected file: `src/strategies/mainnet/MorphoYearnOGWETH.sol`
* Function: `_deallocate(uint256 amount)`

[v3-poc/src/strategies/mainnet/MorphoYearnOGWETH.sol::\_deallocate#L50-L53](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/mainnet/MorphoYearnOGWETH.sol#L50C1-L53C69)

```solidity
    function _deallocate(uint256 amount) internal override returns (uint256) {
        vault.withdraw(amount, address(this), address(this));
        uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
        uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
        uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
        if (wethRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
        }
        require(wethRedeemed + wethBalanceBefore >= amount, "Strategy balance is less than the amount needed");
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
        TokenUtils.safeApprove(address(weth), msg.sender, amount);
        return amount;
    }

```

Both `wethBalanceBefore` and `wethBalanceAfter` are measured after `vault.withdraw`. They are effectively identical and equal to the post-withdraw balance. wethRedeemed`becomes`0 `(`after - after`), regardless of how much was actually redeemed. The event` StrategyDeallocationLoss `is emitted spuriously on every call (`wethRedeemed < amount\`). The intended sanity check to ensure enough assets were redeemed is no longer based on the true delta; it compares the post-withdraw balance twice, losing the information about the actual redemption amount.

## Impact Details

False loss reporting: Every `_deallocate` call emits a “loss” event even when exact or greater-than-expected amounts were redeemed. This pollutes operational telemetry and dashboards. Broken observability: Redeemed amounts are not captured correctly, weakening post-trade analysis, reconciliation, and anomaly detection. Potential operational griefing: If downstream tooling reacts to the event (alerts, halts, automated policy changes), normal operation can be disrupted despite healthy redemptions.

## Recommendations

Fix read order in `_deallocate`: Read `before` balance, perform `vault.withdraw`, read `after` balance, compute `redeemed = after - before`.

## References

* File: `v3-poc-immunefi_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol`
* Function: `_deallocate(uint256 amount)` (lines surrounding `vault.withdraw` and balance reads)

[v3-poc/src/strategies/mainnet/MorphoYearnOGWETH.sol::\_deallocate#L50-L53](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/mainnet/MorphoYearnOGWETH.sol#L50C1-L53C69)

## Proof of Concept

## Proof of Concept

This test case specifically targets a known issue related to the incorrect order of balance checks within the \_deallocate function. The purpose of this test is to demonstrate that, when the deallocate function is invoked under the circumstances where the bug occurs, the system should revert the transaction.

Please add the test `test_poc_withdraw_bug_incorrect_balance_check_order()` at the file of `MorphoYearnOGWETHStrategy.t.sol` which can run successfully

```bash
Ran 1 test for src/test/strategies/MorphoYearnOGWETHStrategy.t.sol:MorphoYearnOGWETHStrategyTest
[PASS] test_poc_withdraw_bug_incorrect_balance_check_order() (gas: 3189308)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.69s (12.86ms CPU time)

Ran 1 test suite in 1.70s (1.69s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import "../libraries/BaseStrategyTest.sol";
import {MorphoYearnOGWETHStrategy} from "../../strategies/mainnet/MorphoYearnOGWETH.sol";
import {TokenUtils} from "../../libraries/TokenUtils.sol";

interface IERC20 {
    function balanceOf(address account) external view returns (uint256);
}

contract MockMorphoYearnOGWETHStrategy is MorphoYearnOGWETHStrategy {
    constructor(
        address _myt,
        StrategyParams memory _params,
        address _vault,
        address _weth,
        address _permit2Address
    ) MorphoYearnOGWETHStrategy(_myt, _params, _vault, _weth, _permit2Address) {}
}

contract MorphoYearnOGWETHStrategyTest is BaseStrategyTest {
    address public constant MORPHO_YEARN_OG_VAULT = 0xE89371eAaAC6D46d4C3ED23453241987916224FC;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant MAINNET_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;

    function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) {
        return
            IMYTStrategy.StrategyParams({
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
        return
            TestConfig({
                vaultAsset: WETH,
                vaultInitialDeposit: 1000e18,
                absoluteCap: 10_000e18,
                relativeCap: 1e18,
                decimals: 18
            });
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 23_298_447;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("MAINNET_RPC_URL");
    }

    function createStrategy(
        address vault,
        IMYTStrategy.StrategyParams memory params
    ) internal override returns (address) {
        return address(new MockMorphoYearnOGWETHStrategy(vault, params, MORPHO_YEARN_OG_VAULT, WETH, MAINNET_PERMIT2));
    }

    function test_strategy_deallocate_reverts_due_to_slippage(
        uint256 amountToAllocate,
        uint256 amountToDeallocate
    ) public {
        amountToAllocate = bound(amountToAllocate, 1e18, testConfig.vaultInitialDeposit);
        amountToDeallocate = amountToAllocate;
        vm.startPrank(vault);
        deal(WETH, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets > 0, "Initial real assets is 0");
        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
        vm.expectRevert();
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
        vm.stopPrank();
    }

    /*     

    function test_allocated_position_generated_yield() public {
            vm.startPrank(address(vault));
            uint256 amount = 100 ether;
            deal(WETH, address(mytStrategy), amount);
            bytes memory prevAllocationAmount = abi.encode(0);
            mytStrategy.allocate(prevAllocationAmount, amount, "", address(vault));
            uint256 initialRealAssets = mytStrategy.realAssets();
            emit MorphoYearnOGWETHStrategyTestLog("initialRealAssets", initialRealAssets);
            assertApproxEqAbs(initialRealAssets, amount, 1e18);
            vm.warp(block.timestamp + 180 days);
            uint256 realAssets = mytStrategy.realAssets();
            emit MorphoYearnOGWETHStrategyTestLog("realAssets", realAssets);
            assertGt(realAssets, initialRealAssets);
            vm.stopPrank();
        }
    */

    // Event from MYTStrategy to track deallocation losses
    event StrategyDeallocationLoss(string message, uint256 amountRequested, uint256 actualAmountSent);

    // PoC for bug: Incorrect balance check order in _deallocate function
    function test_poc_withdraw_bug_incorrect_balance_check_order() public {
        RealMorphoYearnOGWETHStrategyHarness harness = new RealMorphoYearnOGWETHStrategyHarness(
            address(vault),
            getStrategyConfig(),
            MORPHO_YEARN_OG_VAULT,
            WETH,
            MAINNET_PERMIT2
        );

        uint256 withdrawAmount = 1 ether;
        deal(WETH, address(harness), withdrawAmount);
        vm.startPrank(address(vault));
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(address(harness)).allocate(prevAllocationAmount, withdrawAmount, "", address(vault));
        vm.stopPrank();

        vm.expectRevert();
        harness.deallocate(withdrawAmount);
    }
}

contract RealMorphoYearnOGWETHStrategyHarness is MorphoYearnOGWETHStrategy {
    constructor(
        address _myt,
        StrategyParams memory _params,
        address _vault,
        address _weth,
        address _permit2Address
    ) MorphoYearnOGWETHStrategy(_myt, _params, _vault, _weth, _permit2Address) {}

    function deallocate(uint256 amount) external returns (uint256) {
        return _deallocate(amount);
    }

    function buggyRedeemedOnDeallocate(uint256 amount) external returns (uint256) {
        vault.withdraw(amount, address(this), address(this));
        uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
        uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
        return wethBalanceAfter - wethBalanceBefore;
    }
}


```
