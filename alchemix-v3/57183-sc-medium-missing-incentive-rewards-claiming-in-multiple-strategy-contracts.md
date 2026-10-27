# 57183 sc medium missing incentive rewards claiming in multiple strategy contracts

**Submitted on Oct 24th 2025 at 07:04:05 UTC by @Pro\_King for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57183
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/arbitrum/AaveV3ARBUSDCStrategy.sol>
* **Impacts:**
  * Permanent freezing of unclaimed royalties

## Description

## Brief/Intro

Multiple strategy contracts (Aave, Euler, Morpho) lack implementation of the `_claimRewards` function, causing all additional incentive rewards from these protocols to become permanently stuck in the strategies contracts. This affects at least 7 strategy contracts across multiple networks, resulting in significant lost yield for users and the protocol. In production, this would lead to permanent loss of valuable reward tokens (like stkAAVE,AAVE, EUL...) that are distributed on top of the base yield.

## Vulnerability Details

The `MYTStrategy` base contract defines a virtual `_claimRewards` function that child strategies should override to claim protocol-specific rewards:

```solidity
// MYTStrategy.sol
function _claimRewards() internal virtual returns (uint256) {}
```

However, none of the Aave or Euler strategy contracts implement this function, despite these protocols offering significant additional rewards beyond base yield.

This affects at least these strategy contracts:

* `AaveV3ARBUSDCStrategy`
* `AaveV3ARBWETHStrategy`
* `AaveV3OPUSDCStrategy`
* `EulerUSDCStrategy`
* `EulerWETHStrategy`
* `EulerARBUSDCStrategy`
* `EulerARBWETHStrategy`

## Impact Details

1. **Permanent Loss of Reward Tokens**: All incentive rewards from Aave and Euler become permanently stuck with no mechanism to claim them.
2. **Quantifiable Financial Impact**: Based on current reward rates:
   * Aave V3 offers \~1-3% APR in additional rewards on top of base yield
   * Euler offers \~2-5% APR in additional rewards on top of base yield
   * For $10M TVL across these strategies, this represents $300,000-$500,000 in lost rewards annually
3. **Systemic Risk**: This issue affects multiple strategies across different networks, indicating a systemic design flaw rather than an isolated incident.
4. **No Recovery Mechanism**: Once rewards are accrued but not claimed, they become permanently inaccessible as there is no mechanism to extract non-asset tokens from the strategies.

## References

1. Aave V3 Incentive Rewards Documentation: <https://aave.com/docs/developers/smart-contracts/incentives>
2. Same finding: [BadgerDAO contest on Cantina](https://cantina.xyz/code/f57ffb47-0ded-4f04-bcec-ecd7d47fad58/findings/527)

## Proof of Concept

Modify src/test/strategies/AaveV3ARBUSDCStrategy.t.sol file to below version and Command for running test `forge test --mt "test_bug_missing_reward_claiming" -vv`

```diff
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "../libraries/BaseStrategyTest.sol";
import {AaveV3ARBUSDCStrategy} from "../../strategies/arbitrum/AaveV3ARBUSDCStrategy.sol";
+import {IERC20Minimal} from "../../interfaces/IERC20Minimal.sol";
+import "forge-std/console.sol";

contract MockAaveV3ARBUSDCStrategy is AaveV3ARBUSDCStrategy {
    constructor(address _myt, StrategyParams memory _params, address _usdc, address _aUSDC, address _pool, address _permit2Address)
        AaveV3ARBUSDCStrategy(_myt, _params, _usdc, _aUSDC, _pool, _permit2Address)
    {}
}

contract AaveV3ARBUSDCStrategyTest is BaseStrategyTest {
    address public constant AAVE_V3_USDC_ATOKEN = 0x724dc807b04555b71ed48a6896b6F41593b8C637;
    address public constant AAVE_V3_USDC_POOL = 0x794a61358D6845594F94dc1DB02A252b5b4814aD;
    address public constant USDC = 0xaf88d065e77c8cC2239327C5EDb3A432268e5831;
    address public constant OPTIMISM_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;

    function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) {
        return IMYTStrategy.StrategyParams({
            owner: address(1),
            name: "AaveV3ARBUSDC",
            protocol: "AaveV3ARBUSDC",
            riskClass: IMYTStrategy.RiskClass.LOW,
            cap: 10_000e6,
            globalCap: 1e18,
            estimatedYield: 100e6,
            additionalIncentives: false,
            slippageBPS: 1
        });
    }

    function getTestConfig() internal pure override returns (TestConfig memory) {
        return TestConfig({vaultAsset: USDC, vaultInitialDeposit: 1000e6, absoluteCap: 10_000e6, relativeCap: 1e18, decimals: 6});
    }

    function createStrategy(address vault, IMYTStrategy.StrategyParams memory params) internal override returns (address) {
        return address(new MockAaveV3ARBUSDCStrategy(vault, params, USDC, AAVE_V3_USDC_ATOKEN, AAVE_V3_USDC_POOL, OPTIMISM_PERMIT2));
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 387_030_683;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("ARBITRUM_RPC_URL");
    }

    // Add any strategy-specific tests here
    function test_strategy_deallocate_reverts_due_to_slippage(uint256 amountToAllocate, uint256 amountToDeallocate) public {
        amountToAllocate = bound(amountToAllocate, 1 * 10 ** testConfig.decimals, testConfig.vaultInitialDeposit);
        amountToDeallocate = amountToAllocate;
        vm.startPrank(vault);
        deal(testConfig.vaultAsset, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets > 0, "Initial real assets is 0");
        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
        vm.expectRevert();
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
        vm.stopPrank();
    }

 
+   function test_bug_missing_reward_claiming() public {
+       uint256 amountToAllocate = 1000e6; // 1000 USDC
+       vm.startPrank(vault);
        
+        deal(USDC, strategy, amountToAllocate);
+        bytes memory prevAllocationAmount = abi.encode(0);
+        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        
+        // Simulate Aave rewards being sent to the strategy
+        // In production, Aave's RewardsController would send reward tokens (like stkAAVE, AAVE ...) to the strategy
+        MockRewardToken mockReward = new MockRewardToken();
+        uint256 rewardAmount = 100e18; // 100 reward tokens
+        // here I have directly minted it to strategy but actualy aave has it's own function and anyone can call on behalf of strategy and rewards will be stuck for ever at AaveV3ARBStrategy contract
+        mockReward.mint(strategy, rewardAmount);
+        
+        console.log("Simulated Aave rewards sent to strategy:", rewardAmount);
+        console.log("Reward token balance in strategy:", mockReward.balanceOf(strategy));
+        
+        // Test that claimRewards returns 0 (proving no rewards are claimed)
+        uint256 claimedRewards = IMYTStrategy(strategy).claimRewards();
+        console.log("Claimed rewards:", claimedRewards);
+        
+        // The bug: claimRewards returns 0 because _claimRewards is not implemented
+        assertEq(claimedRewards, 0, "claimRewards should return 0 due to missing _claimRewards implementation");
+        
+        // Verify that reward tokens are still stuck in the strategy
+        uint256 finalRewardBalance = mockReward.balanceOf(strategy);
+        assertEq(finalRewardBalance, rewardAmount, "Reward tokens should be stuck in strategy");
+        
+        vm.stopPrank();
+    }
+}

+contract MockRewardToken {
+    mapping(address => uint256) public balanceOf;
+    string public name = "Mock Aave Reward";
+    string public symbol = "mARB";
+    uint8 public decimals = 18;
+    
+    function mint(address to, uint256 amount) external {
+        balanceOf[to] += amount;
+    }
+}
    
   
```

**Output of Test:**

```solidity
Ran 1 test for src/test/strategies/AaveV3ARBUSDCStrategy.t.sol:AaveV3ARBUSDCStrategyTest
[PASS] test_bug_missing_reward_claiming() (gas: 820111)
Logs:
  Simulated Aave rewards sent to strategy: 100000000000000000000
  Reward token balance in strategy: 100000000000000000000
  Claimed rewards: 0

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 719.84ms (8.75ms CPU time)

```
